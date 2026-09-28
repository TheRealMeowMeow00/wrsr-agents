# Workshop items

Part of wrsr-agents (CC BY 4.0). Measured on the game executable and on about
3 400 subscribed items, September 2026.

## Where items live

| | Path |
|---|---|
| Your items, local or being edited | `media_soviet/workshop_wip/<id>/` |
| Subscribed items | `<Steam library>/steamapps/workshop/content/784150/<id>/` |

The game scans `workshop_wip` at startup, and any numeric folder name works.
For local experiments pick ids far from real Workshop ids; TesmioLoader uses
`9100000001` and up.

## workshopconfig.ini

One per item, at the item root:

```
$ITEM_ID 9100000006
$OWNER_ID 0
$ITEM_TYPE WORKSHOP_ITEMTYPE_BUILDING
$VISIBILITY 0
$OBJECT_BUILDING FurnitureFactory
$ITEM_NAME "Furniture Factory"
$ITEM_DESC "Steam BBCode; may span several lines; must not contain a double quote"
$END
```

- `$ITEM_TYPE`: every value the exe knows, with how many subscribed items use
  it: `WORKSHOP_ITEMTYPE_BUILDING` (1807), `_VEHICLE` (1136),
  `_BUILDINGSKIN` (216), `_VEHICLESKIN` (80), `_LANDSCAPE` (80),
  `_SCRIPT` (37), `_BUILDINGEDITOR_ELEMENT` (16), `_TEXT` (14).
- `$OBJECT_BUILDING`, `$OBJECT_VEHICLE`, `$OBJECT_SCRIPT`,
  `$OBJECT_BUILDING_EDITOR_ELEMENT` name the subfolder that holds one object,
  exactly. An item may declare several objects; vehicle packs usually do.
- `$OWNER_ID` and `$VISIBILITY` are written by the game's own upload flow. Copy
  them from an item of yours rather than inventing values.
- Items that ship native plugins are commonly published as
  `WORKSHOP_ITEMTYPE_SCRIPT` (an observed convention: the game has no DLL
  type).

## Steam page limits

- Description: 8000 characters. Title: 128.
- A comment: 1000 characters. A comment containing a URL can be held by Steam
  for review for hours, so name things instead of linking them there.

## A building item

```
<id>/
  workshopconfig.ini
  previewimage.png
  material.mtl            shared by every object in the item
  material_e.mtl          only when the donor has an emissive material
  <Object>/               the folder $OBJECT_BUILDING names
    building.ini          the type definition
    renderconfig.ini      model and destruction
    model.nmf
    building.bbox
    building.fire
    imagegui.png          the build-menu icon
```

Details: `buildings.md`.

## A vehicle item

Observed on 400 subscribed vehicle items holding 1 531 objects:

```
<id>/
  workshopconfig.ini      WORKSHOP_ITEMTYPE_VEHICLE, one $OBJECT_VEHICLE per object
  previewimage.png
  <Object>/
    script.ini            the vehicle definition
    main.nmf
    Material.mtl          plus Material_1.mtl, material_2.mtl ... for variants
    preview.dds           plus preview_1.dds ... matching each variant
    preview_side.dds      plus preview_side_1.dds ...
    bbox.bin
```

Details: `vehicles.md`.

## Native plugins inside an item

Put the DLL and its ini in a `plugins/` folder at the item root. That one
layout serves both loaders:

- **Republic Mod Loader** discovers DLLs in place, in a folder named `plugin`
  or `plugins` at the item root or one level below it.
- **TLMM** installs TesmioLoader plugins from an item: `plugins/` is copied
  into `tesmioloader/build/plugins/`, `vfs/` into the loader's VFS, and
  `<target>.ini.merge` fragments (at the item root or in `plugins/`) are
  merged into existing plugin inis without overwriting keys the user already
  has.

Everything else in the item (buildings, vehicles) is loaded by the game as
usual. Details: `native-mods.md`.

## Before publishing

- The whole item folder is uploaded, including agent files, backups and build
  output. Keep what should not ship somewhere else.
- `$TEXTURE` paths in a `.mtl` are relative to `media_soviet/`, so a material
  can use base-game textures without copying them.
- TesmioLoader's building guide treats borrowed base-game geometry as fine for
  local work and to be replaced before publishing.
- Credit donors, sources and the mods yours builds on in the description.
