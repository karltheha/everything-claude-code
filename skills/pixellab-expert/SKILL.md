---
name: pixellab-expert
description: PixelLab AI pixel art generation via MCP -- characters, animations, tilesets, map objects, and isometric tiles. Covers async workflows, concurrency limits, prompt engineering, style consistency, and cost management. Use when generating pixel art game assets.
origin: ECC
---

# PixelLab Pixel Art Generation

Expert reference for generating pixel art game assets via the PixelLab MCP server.

## When to Activate

- User wants to generate pixel art characters, sprites, or animations
- Creating tilesets for game maps (top-down, sidescroller, isometric)
- Generating map objects or items with transparent backgrounds
- Any PixelLab MCP tool usage (`create_character`, `animate_character`, `create_topdown_tileset`, etc.)
- User says "pixel art", "sprite", "tileset", "game asset", "PixelLab", or similar
- Discussing pixel art style consistency, palettes, or prompt engineering
- Working on DinoSquad Adventure asset generation

## MCP Requirement

PixelLab MCP server must be configured. The API key must come from the environment -- never hardcode it.

```bash
# Load API key from macOS Keychain
export PIXELLAB_API_KEY=$(security find-generic-password -s pixellab-api-key -w)
```

---

## Quick Reference

### Creation Tools

| Tool | Output | Async Time | Cost (generations) | Key Parameters |
|------|--------|------------|-------------------|----------------|
| `create_character` | 4 or 8 directional sprites | 2-5 min | 1 (standard) / 20-40 (pro) | description, size, body_type, mode, proportions |
| `animate_character` | Sprite sheet per direction | 2-4 min | 1/dir (template) / 20-40/dir (custom) | character_id, template_animation_id or action_description |
| `create_topdown_tileset` | 16-23 Wang tiles | ~100 sec | 1 | lower_description, upper_description, tile_size |
| `create_sidescroller_tileset` | Platform tiles | ~100 sec | 1 | lower_description, transition_description, tile_size |
| `create_tiles_pro` | 1-16 tile variations | 15-30 sec | 1 | description, n_tiles, tile_type, tile_size |
| `create_isometric_tile` | Single isometric tile | 10-20 sec | 1 | description, size, tile_shape |
| `create_map_object` | Object with transparent bg | 15-30 sec | 1 | description, width, height, view |

### Retrieval Tools

| Tool | Purpose |
|------|---------|
| `get_character` | Check character status, get rotations, animations, download URL |
| `get_topdown_tileset` | Retrieve completed tileset with PNG + metadata |
| `get_sidescroller_tileset` | Retrieve completed sidescroller tileset |
| `get_tiles_pro` | Retrieve completed tile variations |
| `get_isometric_tile` | Retrieve completed isometric tile |
| `get_map_object` | Retrieve completed map object |
| `list_characters` | List all characters (supports tag filtering) |
| `list_topdown_tilesets` | List all top-down tilesets |
| `list_sidescroller_tilesets` | List all sidescroller tilesets |
| `list_tiles_pro` | List all tiles pro |
| `list_isometric_tiles` | List all isometric tiles |

---

## Async Workflow Pattern

All PixelLab creation tools are **non-blocking**. They return immediately with a job/asset ID. You must poll to retrieve results.

### Submit -> Poll -> Retrieve

```
Step 1: SUBMIT — call create_* tool
         Returns: { job_id, character_id/tileset_id/tile_id }

Step 2: WAIT — respect async processing times
         Characters: 2-5 minutes
         Animations: 2-4 minutes
         Tilesets: ~100 seconds
         Tiles/objects: 15-30 seconds

Step 3: POLL — call get_* tool with the returned ID
         Check status field:
           "processing" -> wait and retry
           "completed"  -> download/use result
           "failed"     -> report error, consider retry

Step 4: USE — extract images from response
         Characters: rotation images + ZIP download URL
         Tilesets: PNG tileset + metadata JSON
         Tiles/objects: individual PNG images
```

### Example: Character Creation

```
# 1. Submit
result = create_character(
    description="cute wizard with blue robes and pointy hat",
    name="Blue Wizard",
    size=48,
    outline="single color black outline",
    shading="medium shading",
    detail="medium detail",
    n_directions=8
)
# Returns: { character_id: "abc-123", job_id: "job-456" }

# 2. Wait ~3 minutes, then poll
character = get_character(character_id="abc-123")
# Check character.status -- repeat if still "processing"

# 3. Use results
# character.rotations[] — individual direction images
# character.download_url — ZIP of all assets
```

---

## Concurrency & Cost Management

### Hard Limits

| Constraint | Value | Consequence |
|-----------|-------|-------------|
| Max concurrent jobs | **2** | Third submission will fail or queue indefinitely |
| Cancel support | **None** | Once submitted, a job cannot be cancelled -- only waited for |
| Pro character cost | **20-40 gen** | vs 1 gen for standard mode |
| Custom animation cost | **20-40 gen/direction** | vs 1 gen/direction for template animations |

### Polling Pattern

```
ALWAYS follow this sequence:
1. Submit job 1
2. Submit job 2 (max concurrent reached)
3. Poll job 1 until complete
4. Only THEN submit job 3
5. Poll job 2 until complete
6. Submit job 4
... continue alternating
```

### Cost Confirmation Protocol

**BEFORE submitting any expensive operation, ALWAYS:**

1. Calculate total cost in generations
2. Present cost to user with comparison to cheaper alternative
3. Wait for explicit user approval
4. Only then submit with `confirm_cost=true` (for custom animations)

**Expensive operations:**
- `create_character` with `mode="pro"`: 20-40 generations (vs 1 for standard)
- `animate_character` without `template_animation_id`: 20-40 generations **per direction** (vs 1/dir for template)
- 8-direction custom animation: up to 320 generations total

**Example cost warning:**
```
This custom animation will cost approximately:
- 8 directions x ~30 gen/direction = ~240 generations
- Template alternative: 8 directions x 1 gen/direction = 8 generations

Do you want to proceed with the custom animation (240 gen) or use a template (8 gen)?
```

---

## Character Creation Guide

### Humanoid Characters

Standard humanoid characters use skeleton-based generation with full style control.

**Required parameters:**
- `description`: Visual appearance (10-30 words, see Prompt Engineering section)

**Key parameters:**

| Parameter | Default | Options | Notes |
|-----------|---------|---------|-------|
| `size` | 48 | 16-128 px | Character is ~60% of canvas. Canvas = size * ~1.4 |
| `n_directions` | 8 | 4 or 8 | 4 = S/W/E/N. 8 = adds diagonals |
| `mode` | standard | standard / pro | Pro = 20-40x cost, always 8 dirs, ignores style params |
| `body_type` | humanoid | humanoid / quadruped | |
| `view` | low top-down | low top-down / high top-down / side | |
| `outline` | single color black outline | single color black outline / single color outline / selective outline / lineless | |
| `shading` | basic shading | flat / basic / medium / detailed | |
| `detail` | medium detail | low / medium / high | |
| `ai_freedom` | 750 | 100-999 | 100 = strict adherence, 999 = creative interpretation |
| `proportions` | default preset | JSON string (see below) | Humanoid only |

**Proportions presets:**
```json
{"type": "preset", "name": "default"}
{"type": "preset", "name": "chibi"}
{"type": "preset", "name": "cartoon"}
{"type": "preset", "name": "stylized"}
{"type": "preset", "name": "realistic_male"}
{"type": "preset", "name": "realistic_female"}
{"type": "preset", "name": "heroic"}
```

**Custom proportions** (all values 0.5-2.0):
```json
{
  "type": "custom",
  "head_size": 1.5,
  "arms_length": 0.8,
  "legs_length": 0.9,
  "shoulder_width": 0.7,
  "hip_width": 0.8
}
```

**Size recommendations by game type:**

| Use Case | Size | Proportions | Notes |
|----------|------|-------------|-------|
| 16-bit RPG | 16-24 px | default or chibi | Classic SNES feel |
| Modern indie | 32-48 px | cartoon or stylized | Good balance of detail and readability |
| Detailed game | 64-96 px | realistic or heroic | Room for expression and detail |
| Portrait/close-up | 128 px | any | Maximum detail, emotion variants |

### Quadruped Characters

For four-legged animals. Requires a `template` parameter.

**Available templates:** `bear`, `cat`, `dog`, `horse`, `lion`

```
create_character(
    description="fierce orange tiger with black stripes",
    name="Tiger",
    body_type="quadruped",
    template="cat",
    size=48,
    view="side"
)
```

**Notes:**
- Quadruped ignores `proportions` parameter
- Each template has different animation sets (see Animation Guide)
- Dinosaurs do NOT map directly to these templates -- for cartoon dinos, use humanoid body type with descriptive prompts (this is how DinoSquad characters were created)

### Standard vs Pro Mode Decision Guide

| Factor | Standard | Pro |
|--------|----------|-----|
| Cost | 1 generation | 20-40 generations |
| Directions | 4 or 8 | Always 8 |
| Quality | Good for most uses | Higher fidelity, more consistent |
| Style control | Full (outline, shading, detail, proportions) | Limited (only description, size, view, body_type) |
| Speed | 2-3 min | 3-5 min |
| When to use | Default choice, iterating on design | Final production asset after validating with standard |

**Rule: Always start with standard mode.** Only upgrade to pro if standard quality is insufficient after 1-2 attempts.

---

## Animation Guide

### Template Animations

Template animations cost **1 generation per direction** and are the default choice.

**Humanoid animations (mannequin template):**

| Category | Animation IDs |
|----------|--------------|
| **Movement** | `walk`, `walk-1`, `walk-2`, `walking`, `walking-2` through `walking-10`, `walking-4-frames`, `walking-6-frames`, `walking-8-frames`, `running-4-frames`, `running-6-frames`, `running-8-frames`, `crouched-walking`, `sad-walk`, `scary-walk` |
| **Jumping** | `jumping-1`, `jumping-2`, `two-footed-jump`, `running-jump`, `front-flip`, `backflip` |
| **Combat** | `lead-jab`, `cross-punch`, `high-kick`, `roundhouse-kick`, `flying-kick`, `hurricane-kick`, `leg-sweep`, `surprise-uppercut`, `fireball` |
| **Actions** | `picking-up`, `pushing`, `pull-heavy-object`, `throw-object`, `drinking` |
| **States** | `breathing-idle`, `fight-stance-idle-8-frames`, `crouching`, `getting-up`, `running-slide` |
| **Death** | `falling-back-death`, `taking-punch` |

**Quadruped animations** vary by template. Use `get_character()` after creation to see the available animation list for your specific animal template.

Common quadruped animations: `walk`, `run`, `idle`, `attack`, `eat`, `sleep`

### Animation Usage

```
# Template animation (cheap: 1 gen/direction)
animate_character(
    character_id="abc-123",
    template_animation_id="walking-8-frames"
)

# Specific directions only
animate_character(
    character_id="abc-123",
    template_animation_id="walk",
    directions=["south", "east"]
)
```

### Custom Animations

Custom animations use `action_description` instead of `template_animation_id`.

**Cost: 20-40 generations per direction.** For an 8-direction character, this can be 160-320 generations.

```
# Step 1: Get cost estimate (confirm_cost defaults to false)
animate_character(
    character_id="abc-123",
    action_description="casting a lightning spell with both hands raised"
)
# Response shows estimated cost

# Step 2: Show cost to user and get approval

# Step 3: Only after user confirms
animate_character(
    character_id="abc-123",
    action_description="casting a lightning spell with both hands raised",
    confirm_cost=true,
    directions=["south", "east", "west", "north"]
)
```

**Custom animation tips:**
- Focus on movement/pose only -- avoid environmental details
- Keep descriptions short: "running while holding sword above head"
- Bad: "running through a forest with wind blowing" (environmental details)
- Defaults to south only if `directions` not specified -- use this for testing before committing to all 8

---

## Tileset Generation

### Top-Down Wang Tilesets

Creates 16 tiles (or 23 with full transition) for corner-based autotiling in top-down maps.

```
create_topdown_tileset(
    lower_description="ocean water",
    upper_description="sandy beach",
    transition_description="wet sand with foam",
    transition_size=0.5,
    tile_size={"width": 16, "height": 16},
    view="high top-down",
    text_guidance_scale=8
)
```

**Parameters:**

| Parameter | Default | Options | Notes |
|-----------|---------|---------|-------|
| `lower_description` | (required) | text | Base/lower terrain |
| `upper_description` | (required) | text | Upper/elevated terrain |
| `transition_description` | null | text | Blending zone (required if transition_size > 0) |
| `transition_size` | 0 | 0.0 / 0.25 / 0.5 / 1.0 | 0 = no transition, 1.0 = full tile (23 tiles) |
| `tile_size` | {w:16, h:16} | 16 or 32 px | Object with width and height |
| `view` | high top-down | low top-down / high top-down | |
| `tile_strength` | 1 | 0.1-2.0 | Pattern consistency |
| `tileset_adherence` | 100 | 0-500 | Structure strictness |
| `tileset_adherence_freedom` | 500 | 0-900 | Structure flexibility |
| `lower_base_tile_id` | null | UUID | For chaining (see below) |
| `upper_base_tile_id` | null | UUID | For chaining (see below) |

### Sidescroller Platform Tilesets

Creates platform tiles for 2D side-view games with transparent backgrounds.

```
create_sidescroller_tileset(
    lower_description="stone brick",
    transition_description="grass",
    transition_size=0.25,
    tile_size={"width": 16, "height": 16},
    text_guidance_scale=8
)
```

**Parameters:**

| Parameter | Default | Notes |
|-----------|---------|-------|
| `lower_description` | (required) | Platform/center material |
| `transition_description` | (required) | Surface/top layer |
| `transition_size` | 0 | 0 = none, 0.25 = light, 0.5 = heavy |
| `base_tile_id` | null | For chaining tilesets |
| `tile_size` | {w:16, h:16} | 16 or 32 px |

### Tiles Pro

Generates 1-16 tile variations with fine control over tile shape and style.

```
create_tiles_pro(
    description="1). grass tile 2). dirt tile 3). stone tile 4). water tile",
    n_tiles=4,
    tile_type="square_topdown",
    tile_size=32,
    tile_view="high top-down"
)
```

**Key parameters:**

| Parameter | Default | Options |
|-----------|---------|---------|
| `n_tiles` | auto | 1, 2, 4, 6, 8, 9, 10, 12, 16 (must form rectangular grid -- 3, 5, 7 NOT allowed) |
| `tile_type` | isometric | hex / hex_pointy / isometric / octagon / square_topdown |
| `tile_size` | 32 | 16-128 px |
| `tile_view` | low top-down | top-down / high top-down / low top-down / side |
| `outline_mode` | outline | outline (gray + outlines) / segmentation (RED/BLUE zones, cleaner) |
| `style_images` | null | JSON array of reference tiles to match style |
| `style_options` | null | JSON: `{"color_palette": true, "outline": true, "detail": true, "shading": true}` |

**Prompting tip:** Number each tile description for best control: `"1). grass tile 2). stone tile 3). lava tile"`. The count should match `n_tiles`.

### Tileset Chaining Pattern

Chaining ensures visual consistency across terrain transitions. **You must complete the first tileset before creating dependent ones.**

```
# Step 1: Create base tileset
result1 = create_topdown_tileset(
    lower_description="deep ocean water",
    upper_description="sandy beach"
)
# Wait for completion...

# Step 2: Retrieve base tile IDs
tileset1 = get_topdown_tileset(tileset_id=result1.tileset_id)
beach_base_tile_id = tileset1.upper_base_tile_id

# Step 3: Chain next tileset using the beach tile as lower reference
result2 = create_topdown_tileset(
    lower_description="sandy beach",
    lower_base_tile_id=beach_base_tile_id,
    upper_description="green grass"
)
# Wait for completion...

# Step 4: Continue chain
tileset2 = get_topdown_tileset(tileset_id=result2.tileset_id)
grass_base_tile_id = tileset2.upper_base_tile_id

result3 = create_topdown_tileset(
    lower_description="green grass",
    lower_base_tile_id=grass_base_tile_id,
    upper_description="grey stone path"
)
```

**Chain order matters:** ocean -> beach -> grass -> stone. Each tileset must complete before the next can reference its tile IDs.

---

## Map Objects & Isometric Tiles

### Map Objects

Creates objects with transparent backgrounds for placement on game maps.

**Basic mode** (standalone object):
```
create_map_object(
    description="wooden barrel with metal bands",
    width=64,
    height=64,
    view="high top-down",
    outline="single color outline",
    shading="medium shading",
    detail="medium detail"
)
```

**Style matching mode** (matches existing map style):
```
create_map_object(
    description="stone fountain with water",
    background_image="{\"type\": \"path\", \"path\": \"assets/my-map.png\"}",
    inpainting="{\"type\": \"oval\", \"fraction\": 0.4}"
)
```

**Size limits:**
- Basic mode: max 400x400 (160,000 total pixels)
- Inpainting mode: max 192x192 (36,864 total pixels)
- Minimum: 32x32

**Inpainting options:**

| Type | Format | Use Case |
|------|--------|----------|
| Default (no inpainting param) | Auto oval 60% | Good starting point |
| Oval | `{"type": "oval", "fraction": 0.3}` | Round objects (trees, pots, rocks) |
| Rectangle | `{"type": "rectangle", "fraction": 0.5}` | Rectangular objects (buildings, crates) |
| Custom mask | `{"type": "mask", "mask_image": "base64..."}` | Complex shapes |

**Background image options:**
- **Path** (saves tokens): `{"type": "path", "path": "assets/map.png"}` -- returns curl command
- **Base64** (inline): `{"type": "base64", "base64": "iVBORw0..."}` -- uses many tokens, small images only

### Isometric Tiles

Creates individual isometric tiles for isometric game maps.

```
create_isometric_tile(
    description="grass on top of dirt",
    size=32,
    tile_shape="block",
    outline="lineless",
    shading="basic shading",
    detail="medium detail",
    text_guidance_scale=8,
    seed=42
)
```

| Parameter | Default | Options |
|-----------|---------|---------|
| `size` | 32 | 16-64 px (above 24px produces better quality) |
| `tile_shape` | block | thin tile (~10% height) / thick tile (~25% height) / block (~50% height) |
| `outline` | lineless | single color outline / selective outline / lineless |
| `shading` | basic shading | flat / basic / medium / detailed / highly detailed |
| `detail` | medium detail | low / medium / highly detailed |
| `seed` | null | Integer for reproducible results |

---

## Prompt Engineering

### What Works Well

**Visual language patterns:**
- Action words: "swirling", "glowing", "bursting", "flowing", "crackling"
- Materials: "metal", "stone", "wood", "crystal", "smoke", "energy", "flame"
- Lighting: "glowing", "shimmering", "shadowed", "backlit"
- Texture: "rough", "smooth", "mossy", "weathered", "polished"

**Structure for character descriptions (10-30 words):**
```
[art style] [character type], [key visual features], [clothing/accessories], [pose/expression], transparent background
```

Examples:
```
"cute cartoon wizard with flowing blue robes, pointy star hat, holding glowing staff, friendly expression, transparent background"
"fierce armored knight, silver plate armor, red cape, battle stance, transparent background"
"small green goblin, ragged brown clothes, mischievous grin, holding stolen gold coin, transparent background"
```

**Structure for tileset descriptions (5-15 words):**
```
[material/surface] [modifier]
```

Examples:
```
Lower: "deep blue ocean water with gentle waves"
Upper: "golden sandy beach with scattered shells"
Transition: "wet sand with white foam"
```

**Numbered tile descriptions (for tiles_pro):**
```
"1). lush green grass 2). dry brown dirt 3). grey cobblestone 4). dark grey stone bricks 5). shallow blue water 6). deep blue water"
```

### What Doesn't Work

| Avoid | Why | Instead |
|-------|-----|---------|
| Game mechanics language | AI doesn't understand game concepts | Use visual descriptions |
| "This is a health potion" | Too abstract | "small glass bottle with glowing red liquid" |
| "Jump ability sprite" | Mechanical, not visual | "character leaping upward with arms raised" |
| Long paragraphs | Dilutes focus, worse results | Keep to 10-30 words |
| Environmental details in animations | Confuses action focus | Describe movement/pose only |
| Negative prompts | PixelLab doesn't support them well | Describe what you want, not what to avoid |

### Resolution Recommendations

| Asset Type | Recommended Size | Notes |
|-----------|-----------------|-------|
| Overworld characters | 16-32 px | Small, readable at game zoom |
| Battle/detail characters | 48-96 px | Room for expression |
| Portrait/close-up | 128 px | Maximum character detail |
| Items/icons | 16-32 px | Small, clear silhouette |
| NPCs | 32-64 px | Between item and character |
| Map tiles | 16 or 32 px | Standard tile sizes |
| Isometric tiles | 32-64 px | 24+ px for better quality |
| Map objects | 32-128 px | Scale to game world |
| Environments | 160-256 px | Background scenes |

---

## Style Consistency

### Shared Style Parameters

These parameters are available across most creation tools and should be kept consistent within a project:

| Parameter | Options | Effect |
|-----------|---------|--------|
| `outline` | `single color black outline` / `single color outline` / `selective outline` / `lineless` | Edge rendering style |
| `shading` | `flat shading` / `basic shading` / `medium shading` / `detailed shading` / `highly detailed shading` | Light/shadow complexity |
| `detail` | `low detail` / `medium detail` / `high detail` / `highly detailed` | Surface detail density |
| `text_guidance_scale` | 1-20 (default 8) | Prompt adherence. Higher = stricter. 6-10 is the sweet spot |
| `ai_freedom` | 100-999 (default 750, characters only) | Creative interpretation. Lower = more predictable |

### Style Combinations by Aesthetic

| Style | Outline | Shading | Detail | Guidance | Notes |
|-------|---------|---------|--------|----------|-------|
| Classic 8-bit | lineless | flat | low | 10 | NES/Game Boy feel |
| 16-bit SNES | single color outline | basic | medium | 8 | Clean retro look |
| Modern indie | single color black outline | medium | medium | 8 | Crisp, readable |
| Detailed RPG | selective outline | detailed | high | 6 | Rich, painterly |
| Cute/cartoon | single color black outline | basic-medium | medium | 8 | Friendly, approachable |

### Using style_image for Palette Enforcement

The `style_images` parameter (in `create_tiles_pro`) and `style_image` (in REST API) allow matching an existing asset's visual style.

```
# Tiles Pro: match style of existing tiles
create_tiles_pro(
    description="1). forest floor 2). mushroom patch 3). flower meadow",
    n_tiles=4,
    style_images="[{\"base64\": \"...\", \"width\": 32, \"height\": 48}]",
    style_options="{\"color_palette\": true, \"outline\": true, \"detail\": true, \"shading\": true}"
)
```

### Seed Reproducibility

Use `seed` parameter for reproducible results (available on isometric tiles and tiles_pro):
```
create_isometric_tile(
    description="mossy stone block",
    size=32,
    seed=42
)
# Same seed + same parameters = same result
```

Seeds are useful for:
- A/B testing small prompt changes
- Reproducing a result you liked
- Creating consistent tile sets with minor variations

---

## DinoSquad Style Spec

### "Warm Pixel Adventure" Style

The canonical art style for DinoSquad Adventure. Apply these parameters consistently for all DinoSquad assets.

**MCP parameters:**
```
outline:              "single color black outline"
shading:              "medium shading"
detail:               "medium detail"
text_guidance_scale:  8
ai_freedom:           750
proportions:          {"type": "preset", "name": "chibi"}
```

**Prompt prefix:** Always start descriptions with `"cute cartoon pixel art, "` and end with `", transparent background"`.

**Color palette:**

| Color | Hex | Usage |
|-------|-----|-------|
| Green | #006b1b | Foliage, dinosaur skin tones |
| Orange | #874e00 | Warm accents, earth tones |
| Blue | #005e9f | Water, sky elements, cool accents |
| Cream | #f0faf2 | Background, light surfaces |

### Asset Dimensions

| Asset Type | Dimensions | Example |
|-----------|-----------|---------|
| Characters | 128x128 px | `size=128` in create_character |
| Items | 32x32 px | Map objects or tiles_pro |
| NPCs | 64x64 px | `size=64` in create_character |
| Environments | 160x100 px | Map objects (basic mode) |
| Effects | 32x32 px | Map objects or tiles_pro |

### Existing Characters

| Name | Description |
|------|-------------|
| Rex | green t-rex dinosaur |
| Trixie | blue triceratops dinosaur |
| Percy | orange pterodactyl dinosaur |
| Spike | red stegosaurus dinosaur |
| Vince | purple velociraptor dinosaur |
| Bree | teal brachiosaurus dinosaur |
| Tank | gray ankylosaurus dinosaur |
| Pari | pink parasaurolophus dinosaur |
| Finn | blue ichthyosaurus dinosaur |
| Karl | boy with brown hair, human character |
| Sarah | girl with red hair, human character |
| Edward | boy with blonde hair, human character |
| Harriett | girl with dark hair, human character |
| Bernie | elderly man with white beard, human character |

### Existing Emotions

Available for prompt reference (currently only Bernie has all variants generated):

| Emotion | Prompt Fragment |
|---------|----------------|
| happy | "happy expression, smiling, joyful, bright eyes" |
| scared | "scared expression, wide fearful eyes, trembling" |
| determined | "determined expression, focused eyes, clenched jaw, brave" |
| surprised | "surprised expression, wide open eyes, open mouth, shocked" |
| sad | "sad expression, droopy eyes, frowning, downcast" |
| angry | "angry expression, furrowed brows, gritting teeth, fierce" |

### SceneCompositor Requirements

DinoSquad renders sprites via a CSS SceneCompositor that layers images. Assets must:
- Have transparent backgrounds (PNG with alpha channel)
- Use `image-rendering: pixelated` CSS for sharp scaling
- Maintain consistent canvas sizes within asset type categories
- Asset manifest is auto-generated by `scripts/generate-manifest.ts`

---

## Workflow Templates

### 1. New Character from Scratch

```
Step 1: CREATE — standard mode first
  create_character(
      description="cute cartoon pixel art, brave young knight with silver armor and red plume helmet, transparent background",
      name="Silver Knight",
      size=48,
      mode="standard",
      n_directions=8,
      outline="single color black outline",
      shading="medium shading",
      detail="medium detail",
      proportions="{\"type\": \"preset\", \"name\": \"cartoon\"}",
      view="low top-down"
  )

Step 2: WAIT 3-5 minutes

Step 3: REVIEW — check output quality
  get_character(character_id="<id>", include_preview=true)

Step 4: ITERATE if needed
  - Adjust description wording
  - Try different proportions preset
  - Modify ai_freedom (lower = more predictable)
  - Only upgrade to pro mode if standard is insufficient

Step 5: ANIMATE — add template animations
  animate_character(
      character_id="<id>",
      template_animation_id="walking-8-frames"
  )
  # Wait, then add more:
  animate_character(
      character_id="<id>",
      template_animation_id="breathing-idle"
  )

Step 6: DOWNLOAD — get all assets
  get_character(character_id="<id>")
  # Use download_url for ZIP of all sprites + animations
```

### 2. Emotion Variant Set

Generate emotion variants for an existing character style.

```
Step 1: ESTABLISH BASE — create the idle/neutral character first (or use existing)

Step 2: CREATE VARIANTS — one per emotion, using style_image for consistency
  For each emotion in [happy, scared, determined, surprised, sad, angry]:

  create_character(
      description="cute cartoon pixel art, <character description>, <emotion prompt fragment>, transparent background",
      name="<Character>-<emotion>",
      size=128,
      n_directions=4,
      outline="single color black outline",
      shading="medium shading",
      detail="medium detail",
      ai_freedom=500
  )

  # IMPORTANT: Queue max 2 at a time, poll between batches

Step 3: REVIEW — compare all variants side by side
  Use get_character for each to verify consistency

Step 4: ITERATE — regenerate any that don't match the character
  Lower ai_freedom (closer to 100) for tighter adherence
```

**DinoSquad-specific emotion workflow:**
```
create_character(
    description="cute cartoon pixel art, green t-rex dinosaur, happy expression, smiling, joyful, bright eyes, transparent background",
    name="Rex-happy",
    size=128,
    outline="single color black outline",
    shading="medium shading",
    detail="medium detail",
    text_guidance_scale=8,
    ai_freedom=500
)
```

### 3. Environment Scene (Day/Night Pair)

```
Step 1: CREATE DAY SCENE
  create_map_object(
      description="lush green forest, tall trees, sunlight filtering through leaves, grass, flowers, colorful cartoon pixel art game background",
      width=160,
      height=100,
      view="side",
      outline="lineless",
      shading="medium shading",
      detail="medium detail"
  )

Step 2: WAIT for completion, retrieve result

Step 3: CREATE NIGHT VARIANT — use day scene as style reference
  create_map_object(
      description="dark forest at night, moonlight, glowing mushrooms, fireflies, starry sky, mysterious, cartoon pixel art game background",
      width=160,
      height=100,
      background_image="{\"type\": \"path\", \"path\": \"<path-to-day-scene.png>\"}",
      inpainting="{\"type\": \"rectangle\", \"fraction\": 0.95}",
      view="side"
  )

Step 4: REVIEW both scenes for palette harmony
```

### 4. Tileset Chain for a Game Map

Build a complete terrain system with seamless transitions.

```
Step 1: PLAN the chain (lowest to highest elevation)
  ocean -> beach -> grass -> stone -> mountain

Step 2: CREATE first tileset
  result1 = create_topdown_tileset(
      lower_description="deep blue ocean water with gentle waves",
      upper_description="golden sandy beach",
      transition_description="wet sand with white foam",
      transition_size=0.5,
      tile_size={"width": 16, "height": 16},
      view="high top-down",
      outline="selective outline",
      shading="medium shading",
      text_guidance_scale=8
  )

Step 3: WAIT ~100 seconds, then retrieve
  tileset1 = get_topdown_tileset(tileset_id=result1.tileset_id)
  # Extract: beach_base_tile_id = tileset1.upper_base_tile_id

Step 4: CHAIN second tileset
  result2 = create_topdown_tileset(
      lower_description="golden sandy beach",
      lower_base_tile_id=beach_base_tile_id,
      upper_description="green grass with small flowers",
      transition_description="dry grass and sand mix",
      transition_size=0.5,
      tile_size={"width": 16, "height": 16},
      view="high top-down",
      outline="selective outline",
      shading="medium shading",
      text_guidance_scale=8
  )

Step 5: CONTINUE chain for each terrain transition
  # Repeat: wait -> retrieve base tile ID -> create next tileset

Step 6: VALIDATE — visually inspect all tilesets together
```

### 5. Batch Item Generation

Generate a set of consistent game items using tiles_pro.

```
Step 1: GENERATE batch of items (max 16 per call)
  create_tiles_pro(
      description="1). red health potion in glass bottle 2). blue mana potion in glass bottle 3). green poison bottle 4). golden key ornate 5). silver key simple 6). wooden treasure chest closed",
      n_tiles=6,
      tile_type="square_topdown",
      tile_size=32,
      tile_view="top-down",
      outline_mode="segmentation",
      seed=42
  )

Step 2: WAIT 15-30 seconds, then retrieve
  get_tiles_pro(tile_id="<id>")

Step 3: REVIEW — check style consistency across items

Step 4: GENERATE more items with style matching
  # Use first batch as style reference for consistency
  create_tiles_pro(
      description="1). iron sword 2). wooden shield 3). magic wand with star 4). leather boots",
      n_tiles=4,
      style_images="[<first batch tile images as base64>]",
      style_options="{\"color_palette\": true, \"outline\": true, \"detail\": true, \"shading\": true}"
  )

Step 5: EXTRACT individual items from the tile grid
```

**DinoSquad items example:**
```
create_tiles_pro(
    description="1). treasure chest closed 2). golden key 3). red gem 4). blue gem 5). campfire lit 6). dinosaur egg",
    n_tiles=6,
    tile_type="square_topdown",
    tile_size=32,
    tile_view="top-down",
    outline_mode="outline",
    seed=42
)
```

---

## Guardrails

These rules are **mandatory** for every PixelLab session:

### 1. Max 2 Concurrent Jobs
Queue at most 2 jobs at a time. Poll the first to completion before submitting a third. Violating this will cause failures.

### 2. No Cancel Support
Once a job is submitted, it cannot be cancelled. Only wait for it to complete or fail. Think before you submit.

### 3. Cost Confirmation Required
NEVER submit a pro-mode character or custom animation without first calculating the cost, presenting it to the user alongside the cheaper alternative, and receiving explicit approval.

### 4. Pro Mode is a Last Resort
Always start with standard mode (1 generation). Only upgrade to pro (20-40 generations) after standard quality has been evaluated and found insufficient.

### 5. API Key from Environment
Never hardcode the PixelLab API key. Always use `PIXELLAB_API_KEY` environment variable, loaded from macOS Keychain:
```bash
export PIXELLAB_API_KEY=$(security find-generic-password -s pixellab-api-key -w)
```

### 6. Async Polling Required
All creation tools return immediately. You must poll with the corresponding `get_*` tool to check completion status. Never assume a job is complete without checking.

### 7. Tileset Chaining Order
When creating connected tilesets, the first tileset MUST complete and its base tile IDs MUST be retrieved before creating dependent tilesets. Never submit chained tilesets in parallel.
