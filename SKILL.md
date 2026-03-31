---
name: unity-prefab
description: Read, understand, and edit Unity prefab files (.prefab) using the ubridge CLI. Use when asked to inspect prefab hierarchy, modify prefab components/properties, create or restructure GameObjects in a prefab, or understand a prefab's structure. Triggers on Unity prefab editing, prefab inspection, GameObject hierarchy, component modification, nested prefab analysis. Requires the `ubridge` CLI (npm i -g github:yulcat/unity-yaml-bridge).
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

The `--project` flag resolves GUIDs to human-readable names (script names, prefab names, sprite names) **and expands nested prefab hierarchies recursively**. Always use it when the Unity project is available.

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
| `GOName#N` | Name collision — sibling GOs with same name get `#1`, `#2`, etc. |

#### Nested Prefab Expansion

When `--project` is provided, nested prefab instances (`{PrefabName}`) are **recursively expanded** — the full child hierarchy from the source prefab is inlined at the correct tree depth:

```
_Card_Template [CameraFacingBillboard, CardBehaviour]
├─ Frame [Image]
├─ _Header_Text {_Header_Text}
│  └─ Text [TextMeshProUGUI*]
└─ small circle {Medal_Template} [MedalDisplayUI*, Animator]
   ├─ Circle_Image [Image*]
   ├─ Small_Circle_Image [Image*]
   └─ ActivationParticles_Template {ActivationParticles_Template}
      ├─ Burst_UIParticles [UIParticles*]
      └─ Activated_UIParticles [UIParticles*]
```

- Children of nested prefabs appear indented under the instance node
- `*` markers on components = overridden in this prefab
- Expansion is recursive (nested-in-nested prefabs also expand)
- Cycle detection prevents infinite loops

#### DETAILS — Component Properties

```ini
[PlayerUI/HUD/HealthBar:Image]
m_Sprite = {21300000, abc123def456}
m_Color = (1, 0.3, 0.3, 1)
m_Type = 3
```

- Path format: `GOPath:ComponentType`
- Only non-default values shown
- See [Value Syntax Reference](#value-syntax-reference) for all types

#### REFS — fileID Map (Do Not Edit)

Tool-only section for lossless write-back. AI agents should **not modify** REFS entries, but **must consult REFS** when writing `->` path references — the REFS keys are the only valid reference targets.

### 3. Edit the .ubridge File

Edit STRUCTURE and DETAILS sections as needed:

- **Change a property**: Modify the value in DETAILS
- **Set a reference**: Use `->GOPath:Component` to point to an object in the same prefab
- **Add a component**: Add a new `[GOPath:ComponentType]` section in DETAILS
- **Add a GameObject**: Add it to STRUCTURE with `+ ` prefix and add DETAILS
- **Remove a GameObject**: Mark with `- ` prefix in STRUCTURE

### 4. Write Back to Unity YAML

```bash
ubridge write <edited.ubridge> --yaml <original.prefab> -o <output.prefab>
```

The `--yaml` flag provides the original prefab for merging. `-o` can safely overwrite the original because ubridge reads the full file before writing.

---

## Path References (`->` syntax)

This is the key feature for readable cross-references within a prefab. Instead of opaque `{fileID}` numbers, references use human-readable paths.

### Reading Path References

In the DETAILS section, internal references appear as:
```ini
activateDisplayText = ->Button_Text:TextMeshProUGUI
attackRoot = ->MouthEnd:Transform
targetGraphic = ->Background:Image
```

These mean "this field points to the TextMeshProUGUI component on the Button_Text GameObject" etc.

### Writing Path References

When editing, use `->` references — but **only use keys that already exist in the REFS section**. Do NOT invent paths.

```ini
# ✅ Correct — these keys exist in REFS
activateDisplayText = ->Button_Text:TextMeshProUGUI
moveTarget = ->Waypoint:Transform

# ❌ WRONG — do not construct paths from the STRUCTURE tree
activateDisplayText = ->Canvas/Panel/Button_Text:TextMeshProUGUI
```

**Critical rule**: The `->` value must **exactly match a key in the REFS section**. The REFS section lists all valid reference targets. Before writing a `->` reference, look up the REFS section and use the exact key string.

### How to find the right key

1. Look at the `--- REFS` section at the bottom of the file
2. Find the target component (e.g., `Button_Text:TextMeshProUGUI = 12345`)
3. Use exactly that key: `->Button_Text:TextMeshProUGUI`

### Path Reference Rules

| Pattern | Resolves To |
|---------|-------------|
| `->Name:Component` | The Component on that GO (key from REFS) |
| `->Name` | Shorthand for the GO (resolves to its fileID) |
| `@Name:Component` | Alias for `->` (both work identically) |
| `[->A, ->B:Image]` | Array of internal references |

**Important**: Path references only work for objects **within the same prefab** (including expanded nested prefab children). Cross-asset references still use `{fileID, guid}` format.

### Nested Prefab References

Components inside nested prefab instances are also listed in the REFS section:

```
--- REFS
...
_Header_Text:TextMeshProUGUI = 7213628277689136018
small circle:MedalDisplayUI = 6683245448512787790
```

Use these keys directly:
```ini
headerText = ->_Header_Text:TextMeshProUGUI
medalDisplay = ->small circle:MedalDisplayUI
```

**Never construct paths by joining STRUCTURE tree nodes with `/`.** Only use keys from REFS.

---

## Variant Prefabs

Variants show overrides relative to the base prefab. When the base can be resolved, the full inherited tree is shown.

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
- Added GameObjects in variants are also properly handled in the REFS section

---

## MonoBehaviour GUID Resolution

When `--project` is provided, custom MonoBehaviour scripts are resolved from GUIDs to their class names. Instead of seeing a raw GUID, you see the actual C# class name in the component list:

```
Player [PlayerController, Health, DamageDealer]
```

This works by looking up the script GUID in the project's `.meta` files and extracting the class name.

---

## Value Syntax Reference

| Unity Type | .ubridge | Example |
|------------|----------|---------|
| int, float | literal | `5`, `0.42` |
| bool | `0` / `1` | `1` |
| string | literal | `Hello World` |
| Vector2/3/4 | tuple | `(0.5, 0.5)`, `(1, 2, 3)` |
| Color | `(r, g, b, a)` | `(1, 0.9, 0.3, 1)` |
| Asset ref | `{fileID, guid}` | `{21300000, e197d4e8...}` |
| Internal ref | `->Path:Component` | `->Button_Text:TextMeshProUGUI` |
| Null | `null` | `null` |
| Array | `[item1, item2]` | `[->GO1, ->GO2]` |
| LayerMask | `bits:N` | `bits:512` |
| AnimationCurve | `curve[...]` | `curve[(0, 1, 0, 0), (1, 0, 0, 0)]` |

### Transform Shorthand

```ini
[Button:RectTransform]
pos = (10, 20)              # anchoredPosition
size = (200, 50)            # sizeDelta
anchor = (0, 0)-(1, 1)     # anchorMin-anchorMax
pivot = (0.5, 0.5)         # pivot
scale = (1, 1, 1)          # localScale
rot = (0, 45, 0)           # euler angles
```

### Complex Arrays

```ini
[Chomper:DamageDealer]
attackPoints:
  - radius = 0.42
    offset = (0, 0, 0)
    attackRoot = ->MouthEnd:Transform
```

See [references/value-syntax.md](references/value-syntax.md) for the complete reference.

---

## Examples

### Inspect a prefab hierarchy
```bash
ubridge parse Assets/Prefabs/Player.prefab --project . | head -20
```

### Change a sprite reference
```bash
ubridge parse Assets/Prefabs/Card.prefab --project . -o /tmp/card.ubridge
# Edit /tmp/card.ubridge: change m_Sprite GUID
ubridge write /tmp/card.ubridge --yaml Assets/Prefabs/Card.prefab -o Assets/Prefabs/Card.prefab
```

### Batch inspect all prefabs
```bash
find Assets -name "*.prefab" | while read f; do
  echo "=== $f ==="
  ubridge parse "$f" --project . 2>/dev/null | sed -n '/^--- STRUCTURE/,/^--- DETAILS/p' | head -20
done
```

### Rewire a reference inside a prefab
```bash
ubridge parse Assets/Prefabs/UI.prefab --project . -o /tmp/ui.ubridge
# Edit: change targetGraphic = ->NewButton:Image
ubridge write /tmp/ui.ubridge --yaml Assets/Prefabs/UI.prefab -o Assets/Prefabs/UI.prefab
```

## Limitations

- Scene files (`.unity`) not yet supported
- Nested prefab internal components must be edited via the source prefab
- Sibling GOs with identical names use `#N` disambiguation — path references skip ambiguous cases
- Some complex AnimationCurve data may appear as raw values
