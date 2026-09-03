# Content plugins

Optional content kept self-contained, so that adding or deleting a plugin's directories adds or
removes the feature, and syncing with `upstream/LostCityRS/Content` stays trivial.

Tooling lives in the engine fork under `engine/tools/plugins/`.

## A plugin is two directories

Models must live under `content/models/` — both `PackFile.ts` and `graphics/pack.ts` hard-code that
scan root, and a symlink does not work because the directory walk is lstat-based. So a plugin is
split, tied together by its `plugin.pack` manifest. This is the one unavoidable wart.

```
content/
├─ models/plugins/<name>/*.ob2              ← assets
└─ scripts/plugins/<name>/
    ├─ plugin.pack                          ← id manifest (source of truth)
    ├─ configs/<name>.{obj,npc,loc}
    └─ scripts/<name>.rs2
```

The packer's folder validator requires `.rs2` in a directory named `scripts` (or whose parent is)
and other configs in `configs`, which allows exactly the one extra nesting level used above.

## Ids

`content/pack/*.pack` is **derived — never hand-edit it.** Each plugin declares its ids in its own
fragment, and `SyncPluginPacks.ts` regenerates the pack files from all fragments.

```
# <type> <id> <name>
model 20000 plugin_dragon_scimitar
obj   20000 dragon_scimitar
```

Ids come from a reserved range so they can never collide with upstream, which allocates densely
from 0. See `engine/tools/plugins/PluginIds.ts`.

| type | plugin base | ceiling | ceiling comes from |
| --- | --- | --- | --- |
| `obj` `model` `seq` `spotanim` | 20000 | 50000 | packer index buffer |
| `varbit` | 4000 | 50000 | packer index buffer |
| `loc` | **12000** | **16384** | `Loc` packs the type into 14 bits |
| `varp` | **1200** | **2000** | the client stores values in a fixed `int[2000]` |
| `npc` | **1792** | **2048** | 11-bit field in the info encoder |

**`loc` and `npc` have their own ceilings, and both fail silently.** `Loc` packs type, shape,
angle and layer into one int and gives the type 14 bits (`src/engine/entity/Loc.ts`:
`(type & 0x3fff) | ...`), and the getter masks again on the way out. A loc id above 16383 is
therefore placed as `id & 0x3fff` - a completely different loc - with no error anywhere. It looks
exactly like "the loc did not spawn": something appears at the tile, it is not yours, and
`loc_find` never sees it. Id 20000 lands on 3616.

Npc type ids are packed into 11 bits by the info encoder
(`src/network/rsbuf/info.ts` → `pbit(11, ntype)`) and `pbit` masks *silently*, so an id of 2048 or
above renders as a different npc with no error at all. The 50000 ceiling on the others comes from
the packers' fixed `Packet.alloc(3)` index buffers at 2 bytes per entry.

Sparse ids are safe — the packers write empty entries across gaps and the client never requests a
version-0 file. The cost is memory for default config objects, roughly 13-15 MB at these bases.

**Naming:** game-facing names stay natural (`dragon_scimitar`, so `::give dragon_scimitar` and the
automatic `cert_` link work). Asset names take a `plugin_` prefix, because upstream renames
placeholder model names and `PackFile.register` silently rebinds a duplicate name to the new id.

## Revision bumps

The bases are only safe while they sit above everything upstream uses, because `SyncPluginPacks`
rebuilds a pack by deleting every id at or above the base. A revision bump can quietly break that,
so both tools check it against the upstream branch for the revision being built
(`upstream/<engine.revision>`) and refuse rather than write:

```
REFUSING  loc: upstream reaches 5115 at upstream/289, at or above the plugin base 5000
          syncing would delete upstream entries between 5000 and 5115
```

Measured against the 377 branch, which is the likely next step:

| pack | base | 377 upstream max | 377 ceiling | verdict |
| --- | --- | --- | --- | --- |
| `obj` | 20000 | 7955 | 50000 | unaffected |
| `model` | 20000 | 14926 | 50000 | unaffected |
| `seq` | 20000 | 3996 | 50000 | unaffected |
| `spotanim` | 20000 | 665 | 50000 | unaffected |
| `npc` | 1792 | **3851** | 8192 | **must move** - but the field widens to 13 bits at 377, so there is far more room than the 256 slots here |
| `loc` | 12000 | **14973** | 16384 | **must move**, and this is the tight one: `Loc` still masks to 14 bits at 377, so upstream's 14973 leaves roughly 1,400 ids |

So an obj-shaped plugin survives a bump untouched, while `npc` and `loc` need re-numbering - `loc`
being the one to watch, since upstream grows into a ceiling that does not.

## Flags and hooks

Two things a content plugin cannot express for itself are generated into
`scripts/plugins/_generated/` by the sync. Both are derived - never edit them.

**Flags.** RuneScript cannot read server config (only `map_members` and `map_live` are exposed as
ops), so the enabled state is compiled in as a constant:

```
^plugin_ghost_chest_enabled = 1
```

A plugin is on unless listed under `disabled` in `engine/data/config/plugins.json`:

```json
{ "disabled": ["ghost_chest"] }
```

Toggling needs a repack, which is the same cost as editing any other content. The data stays in the
cache either way - the flag gates behaviour, not existence.

**Hooks.** RuneScript triggers are single-owner: there is exactly one `[login,_]` in the game, so
plugins cannot each own it and none can react to login without editing core content. A plugin opts
in by defining a proc:

```
[proc,ghost_chest_login]
mes("...");
```

and the generated dispatcher fans core's single call out to whichever plugins implement it, guarded
by the flag. Hooks available: `login`, `logout`. Add more in `tools/plugins/GenerateContent.ts`
along with the matching core seam.

> The login dispatcher is called **above** the tutorial branch, not at the end of the trigger.
> That branch ends in `@start_tutorial`, which is a tail jump rather than a call, so anything after
> it never runs for a new player - and new players are exactly who a login hook tends to care about.

## Commands

```sh
cd engine
npx tsx --test tools/plugins/*.test.ts              # tooling tests
npx tsx tools/plugins/VerifySeams.ts                # seams still present in engine and content
npx tsx tools/plugins/SyncPluginPacks.ts            # regenerate content/pack and _generated/
npx tsx tools/plugins/SyncPluginPacks.ts --check    # exit 1 on drift, write nothing
npx tsx tools/plugins/VerifyPluginPacks.ts          # invariants
npm run build                                       # repack the cache
```

All of this runs in CI (`.github/workflows/plugins.yml`) on every push and pull request, so a
dropped seam or a drifted pack fails there rather than in game.

After an upstream sync, a conflict in `content/pack/*.pack` is resolved by one deterministic
command rather than by judgement:

```sh
git checkout --theirs content/pack
npx tsx tools/plugins/SyncPluginPacks.ts
```

## Getting assets in

**Across Lost City revisions the `.ob2` and `.anim` file formats are unchanged, so an *unrigged*
model ports as a plain file copy.** Verified: model id 2373 is `inv_scimitar` on 289 and
`obj_bronze_scimitar` on 377-wip and the two files are byte-identical.

**Rigged models do NOT port — see the next section before importing an npc, a worn item, or any
animated model.**

```sh
npx tsx tools/plugins/import/FromLostCityRev.ts \
  --plugin dragon_scimitar --branch 377-wip \
  --model models/obj/obj_dragon_scimitar.ob2 --as plugin_dragon_scimitar
```

That copies the blob out of the other branch and allocates it an id in the plugin's fragment.

### Rigged models do not port across revisions

A model carrying face labels (TSKIN) or vertex labels (VSKIN) is *rigged* - those labels bind its
vertices to animation groups. **Jagex re-assigned rig labels between revisions**, so an imported
rigged model animated by this revision's seqs deforms into mangled limbs.

Measured on `npc_1279`, which exists in both revisions at identical length:

| segment | bytes differing (289 vs 377) |
| --- | --- |
| TSKIN (face labels) | 843 / 859 |
| VSKIN (vertex labels) | 425 / 500 |
| geometry (vertexX/Y/Z) | 1 / 1130 |
| face colours, indices, priority, alpha | 0 |

Same mesh, same colours - different skeleton. And the relabelling is **per model, not a global
permutation**: deriving a 377 -> 289 label map from 152 model pairs produced 8,962 conflicts (191
even when restricted to human-skeleton models), so it cannot be remapped mechanically.

How badly this shows depends on how many label groups the model uses. A full humanoid deforms
visibly - `barrows_karil` came through mangled. A model bound to only a few groups, such as a weapon
held in one hand, often animates acceptably: the imported `plugin_dragon_scimitar_manwear` looks and
animates correctly in game despite carrying 377 labels. So treat the warning as "check this in
game", not "this is broken".

`FromLostCityRev.ts` warns on import and `VerifyPluginPacks.ts` warns on every rigged model under
`models/plugins/`. Both are warnings, not errors - a rigged import that looks right is fine to keep.

**To re-rig**, open the model in ob2blender or the Model & Anim Editor and re-assign VSKIN/TSKIN to
this revision's scheme. For reference, 289's human skeleton uses labels `0-88` plus `255` for
"unlabelled", sampled across its 351 human-rigged models; the dominant groups by vertex count are:

| label | verts | | label | verts |
| --- | --- | --- | --- | --- |
| 16 | 5543 | | 7 | 761 |
| 19 | 3464 | | 6 | 510 |
| 8 | 1022 | | 22 | 501 |
| 50 | 920 | | 21 | 484 |
| 5 | 451 | | 29 | 430 |

The practical shortcut: open a *native* 289 model of the same body plan (e.g. `npc_1279`, the
Draugen - a single-model humanoid driven by `human_walk_*`) alongside the import, and copy its
label assignment.

For authoring or editing models, use the community tools rather than writing new ones:

- [AmVoidGuy/LostCityModelAndAnimEditor](https://github.com/AmVoidGuy/LostCityModelAndAnimEditor) —
  web model *and animation* editor built for Lost City; exports `.ob2` and `.Frame`.
- [stone-temple-pilot/ob2blender](https://github.com/stone-temple-pilot/ob2blender) — Blender 4.5
  add-on for `.ob2`/`.dat`, handling HSL16/RGB15 colour, face/vertex labels, priorities and alpha.
- [LostCityRS/ModelEditor](https://github.com/LostCityRS/ModelEditor) — VS Code `.ob2` previewer.

**Importing from OSRS is a different problem** and no importer here handles it: OSRS ships a JS5
container (`.dat2`/`.idx`) and a much later model encoding with per-face textures and per-vertex
alpha, none of which the 2004 renderer can express. Route it through a community converter such as
[scape-tools-osrs-data-converter](https://github.com/Rune-Status/scape-tools-osrs-data-converter),
or write a new importer that produces the same contract as `FromLostCityRev.ts` — a directory of
`.ob2` files plus fragment lines. Nothing else needs to change to accommodate one.

## Gotchas

- **`build.verify` must be `false`** in `engine/data/config/world.json`. `PackShared.ts:313` checks
  packed configs against hard-coded CRCs of the authentic 289 data, so adding *any* obj/npc/loc is
  otherwise a hard build failure. This is also why upstream's own packs are frozen.
- **Rigged imports animate wrongly** until re-labelled - see above. Unrigged models (inventory
  icons, static scenery) are unaffected.
- **Centrepiece loc models use the bare name.** `LocConfig.ts` looks a centrepiece model up under
  the exact config name and the shape-suffix loop explicitly *skips* `_8`. Importing a `_8`-suffixed
  file from a later revision and keeping that name fails with `Failed to find suitable loc models` —
  register it without the suffix. Other shapes (`_1`, `_q`, `_9`, …) do use their suffix.
- **`::give` and debugprocs need `node.production: false`**, which also grants *every* login
  `staffmodlevel = 4` (`LoginThread.ts:50`). Only do this on a server that isn't reachable.
- **Removing an obj id that players already hold voids those items** on their next load. Inherent to
  deleting content, not something tooling can fix.

## Installed plugins

| plugin | proves | ids |
| --- | --- | --- |
| `dragon_scimitar` | obj + inventory/worn models + `iop2` wield + level gate | obj 20000-20001, model 20000-20001 |
| `barrows_karil` | npc + model, reusing existing 289 seqs | npc 1792, model 20002 |
| `ghost_chest` | loc + two-state `oploc`/`loc_change` + centrepiece model naming | loc 20000-20001, model 20003-20004 |

Spawn them with `::give dragon_scimitar 1`, `::~npc barrows_karil`, `::~loc ghost_chest_closed`.
