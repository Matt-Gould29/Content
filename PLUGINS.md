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

| type | plugin base | ceiling |
| --- | --- | --- |
| `obj` `loc` `model` `seq` `spotanim` | 20000 | 50000 |
| `npc` | **1792** | **2048** |

**`npc` is the exception.** Npc type ids are packed into 11 bits by the info encoder
(`src/network/rsbuf/info.ts` → `pbit(11, ntype)`) and `pbit` masks *silently*, so an id of 2048 or
above renders as a different npc with no error at all. The 50000 ceiling on the others comes from
the packers' fixed `Packet.alloc(3)` index buffers at 2 bytes per entry.

Sparse ids are safe — the packers write empty entries across gaps and the client never requests a
version-0 file. The cost is memory for default config objects, roughly 13-15 MB at these bases.

**Naming:** game-facing names stay natural (`dragon_scimitar`, so `::give dragon_scimitar` and the
automatic `cert_` link work). Asset names take a `plugin_` prefix, because upstream renames
placeholder model names and `PackFile.register` silently rebinds a duplicate name to the new id.

## Commands

```sh
cd engine
npx tsx tools/plugins/SyncPluginPacks.ts            # regenerate content/pack from fragments
npx tsx tools/plugins/SyncPluginPacks.ts --check    # exit 1 on drift, write nothing
npx tsx tools/plugins/VerifyPluginPacks.ts          # invariants
npm run build                                       # repack the cache
```

After an upstream sync, a conflict in `content/pack/*.pack` is resolved by one deterministic
command rather than by judgement:

```sh
git checkout --theirs content/pack
npx tsx tools/plugins/SyncPluginPacks.ts
```

## Getting assets in

**Across Lost City revisions the `.ob2` and `.anim` formats are unchanged, so a port is a file
copy.** Verified: model id 2373 is `inv_scimitar` on 289 and `obj_bronze_scimitar` on 377-wip and
the two files are byte-identical; `models/_animset/anim_124.anim` matches across branches too.

```sh
npx tsx tools/plugins/import/FromLostCityRev.ts \
  --plugin dragon_scimitar --branch 377-wip \
  --model models/obj/obj_dragon_scimitar.ob2 --as plugin_dragon_scimitar
```

That copies the blob out of the other branch and allocates it an id in the plugin's fragment.

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
