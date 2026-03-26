---
name: unity-prefab
description: Read, understand, and edit Unity prefab files (.prefab) using the ubridge CLI. Use when asked to inspect prefab hierarchy, modify prefab components/properties, create or restructure GameObjects in a prefab, or understand a prefab's structure. Triggers on Unity prefab editing, prefab inspection, GameObject hierarchy, component modification, nested prefab analysis. Requires the `ubridge` CLI (npm i -g unity-yaml-bridge).
---

# Unity Prefab Editing

Edit Unity prefab files through the `ubridge` CLI, which converts Unity's verbose YAML into a compact `.ubridge` format optimized for AI comprehension.

## Prerequisites

- `ubridge` CLI installed globally:
  ```bash
  npm i -g github:yulcat/unity-yaml-bridge
  ```
  If install fails, clone manually:
  ```bash
  git clone https://github.com/yulcat/unity-yaml-bridge.git
  cd unity-yaml-bridge && npm install && npm link
  ```
- Know the Unity project root path (needed for GUID resolution)

## Core Workflow

### 1. Read a Prefab

```bash
ubridge parse <file.prefab> --project <unity-project-root> -o <output.ubridge>
```

The `--project` flag resolves GUIDs to human-readable names (script names, prefab names, sprite names). Always use it when the Unity project is available.

### 2. Understand the Output

The `.ubridge` format has 3 sections:

```
# ubridge v1 | prefab
--- STRUCTURE          ← GameObject tree (read this first)
--- DETAILS            ← Component properties (edit these)
--- REFS               ← fileID map (never touch this)
```

#### STRUCTURE — The Hierarchy

```
PlayerUI [PlayerController, Health]
├─ HUD [Canvas]
│  ├─ HealthBar [Image, Slider]
│  └─ ScoreText [TextMeshProUGUI]
├─ Weapon {Sword}
│  └─ Blade [MeshRenderer, BoxCollider]
└─ + DebugPanel [Text]
```

| Syntax | Meaning |
|--------|---------|
| `[Comp1, Comp2]` | Components (boilerplate like Transform filtered out) |
| `{PrefabName}` | Nested prefab instance (children belong to source prefab) |
| `Comp*` | Overridden in this variant |
| `+ GOName` | Added in this variant |
| `- GOName` | Removed in this variant |

#### DETAILS — Component Properties

```ini
[PlayerUI/HUD/HealthBar:Image]
m_Sprite = {21300000, abc123def456}
m_Color = (1, 0.3, 0.3, 1)
m_Type = 3
```

- Path format: `GOPath:ComponentType`
- Only non-default values shown
- See references/value-syntax.md for all value types

#### REFS — fileID Map (Do Not Edit)

Tool-only section for lossless write-back. Ignore completely.

### 3. Edit the .ubridge File

Edit STRUCTURE and DETAILS sections as needed:

- **Change a property**: Modify the value in DETAILS
- **Add a component**: Add a new `[GOPath:ComponentType]` section in DETAILS
- **Add a GameObject**: Add it to STRUCTURE with `+ ` prefix and add DETAILS
- **Remove a GameObject**: Mark with `- ` prefix in STRUCTURE

### 4. Write Back to Unity YAML

```bash
ubridge write <edited.ubridge> --yaml <original.prefab> -o <output.prefab>
```

The `--yaml` flag provides the original prefab for merging. New GameObjects/components get auto-generated fileIDs.

## Examples

### Inspect a prefab hierarchy

```bash
ubridge parse Assets/Prefabs/Player.prefab --project . | head -20
```

Read just the STRUCTURE section to understand the hierarchy without loading full details.

### Change a sprite reference

```bash
ubridge parse Assets/Prefabs/Card.prefab --project . -o card.ubridge
# Edit card.ubridge: change m_Sprite GUID
ubridge write card.ubridge --yaml Assets/Prefabs/Card.prefab -o Assets/Prefabs/Card.prefab
```

### Batch inspect all prefabs

```bash
find Assets -name "*.prefab" | while read f; do
  echo "=== $f ==="
  ubridge parse "$f" --project . 2>/dev/null | sed -n '/^--- STRUCTURE/,/^--- DETAILS/p' | head -20
done
```

## Variant Prefabs

Variants show only overrides relative to the base:

```
# ubridge v1 | variant | base-guid:abc123...
--- STRUCTURE
Enemy [EnemyAI*, SpriteRenderer*]
├─ Hitbox [BoxCollider2D]
└─ + Shield [SpriteRenderer]
--- DETAILS
[Enemy:EnemyAI]
speed = 5.0
damage = 20

[+ Shield:SpriteRenderer]
m_Sprite = {21300000, shield-guid-here}
```

- `*` marks overridden components
- `+` marks added GameObjects
- Only modified properties appear in DETAILS

## Limitations

- Scene files (`.unity`) not yet supported
- Nested prefab internal components must be edited via the source prefab
- Some complex AnimationCurve data may appear as raw values

## Value Syntax Reference

See [references/value-syntax.md](references/value-syntax.md) for the complete value type reference.
