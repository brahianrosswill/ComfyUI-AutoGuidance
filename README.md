# ComfyUI-AutoGuidance

Adds an **AutoGuidance + CFG** **GUIDER** node for **ComfyUI**, based on the AutoGuidance paper: **“AutoGuidance: Automatic Guidance for Diffusion Models”**  - `https://arxiv.org/abs/2406.02507`

This node lets you guide sampling using **two models**:

- **good_model**: your normal model path (checkpoint + your “good” LoRAs, prompt, etc.)
- **bad_model**: an intentionally different path (often “baseline” / “undesired” / “early-bad” LoRA stack)

At each denoise step, the guider computes the usual **CFG** output from the **good** model and then applies an **AutoGuidance delta** derived from the difference between **good** and **bad** predictions.

---

## Install

Clone into:

`ComfyUI/custom_nodes/ComfyUI-AutoGuidance`

Restart ComfyUI.

---

## Node

**AutoGuidance CFG Guider (good+bad)**  
Category: `sampling/guiders`  
Output: `GUIDER` (plug into `SamplerCustomAdvanced`)

---

## Quick start

### 1) Build your good and bad models

Typical graph layout:

- `CheckpointLoaderSimple` → **good_model**
- `CheckpointLoaderSimple` → **bad_model**
- Apply LoRAs on each path however you want:
  - good path: character/style/LCM/etc.
  - bad path: “bad” LoRA stack (or a different style, different character LoRA, or *no* LoRAs)

Then:

- `CLIPTextEncode` positive/negative → `AutoGuidance CFG Guider (good+bad)`
- `AutoGuidance CFG Guider (good+bad)` → `SamplerCustomAdvanced`

### 2) Turn it off / on

- **AG off**: `w_autoguide = 1.0`
- Start here: `w_autoguide = 2.0`, `ag_max_ratio = 0.35`
- Stronger: `w_autoguide = 3.0–5.0`, `ag_max_ratio = 0.6–1.0`

---

## Swap modes (this matters a lot)

### `dual_models_2x_vram` (recommended for speed)
- Requires **two truly separate model instances** (≈ **2× VRAM** for the checkpoint).
- **You MUST load the checkpoint twice from two different file paths** so ComfyUI does not reuse the same underlying model object.

**Do this:**
- Copy your checkpoint file:
  - `SDXL.safetensors` → `SDXL_copy.safetensors`
- Load `SDXL.safetensors` for **good_model**
- Load `SDXL_copy.safetensors` for **bad_model**

**Do NOT do this:**
- Loading the *same* checkpoint path for both inputs.
  - If you do, ComfyUI will reuse the same base model object.
  - That means **dual mode will not actually be dual** internally.
  - You will effectively be running a shared-model swap path instead.
  - This is not “maybe”; it is how the underlying object reuse works.

**Why you care:**  
Dual mode removes the expensive “swap LoRA stacks on the same model repeatedly” overhead. If you don’t truly have two model instances, performance can get **orders of magnitude worse** in some setups (people have seen ~**20×** slowdowns).

---

### `shared_safe_low_vram`
- One model instance, swaps patch state between good/bad on the same object.
- Lowest VRAM.
- **Can be extremely slow** depending on your ComfyUI build + LoRA stack + hook stack, because it repeatedly unpatches/patches weights.

### `shared_fast_extra_vram`
- One model instance, but keeps more things resident to reduce swap overhead.
- Uses extra VRAM to reduce the cost of swapping.
- Still fundamentally a shared-model swap approach.

---

## AutoGuidance settings explained

### `w_autoguide`
Paper-style control:
- `1.0` = off
- `2.0` = moderate
- `3.0–5.0` = strong

Internally the node turns this into the scale used to add the AG delta on top of CFG.

### `ag_delta_mode`
How the AG direction is computed:

- **`bad_conditional` (default, most sensitive to LoRA differences)**  
  Compares *good CFG output* vs *bad conditional-only* output.

- `raw_delta`  
  Uses the delta between the full good-guided output and full bad-guided output.

- `project_cfg`  
  Projects the “push-away-from-bad” direction onto the CFG direction (more conservative).
  - `ag_allow_negative` controls whether the projection can flip direction.

### `ag_max_ratio`
Caps AG magnitude relative to CFG magnitude:
- If AG looks too subtle: **increase this first** (e.g. `0.35 → 0.75`).
- If AG breaks structure / looks “wrong VAE-ish”: lower it.

### Ramp controls (AG strength over the denoise timeline)

These shape **how much of `ag_max_ratio` is allowed** early vs late.

- `ag_ramp_mode`:
  - `flat` (recommended default): constant cap across steps
  - `detail_late`: weaker early, stronger late (detail emphasis)
  - `compose_early`: stronger early, weaker late (composition emphasis)
  - `mid_peak`: strongest mid-steps (often a good “balanced” feel)

- `ag_ramp_power`:
  - Shapes the curve (higher = more extreme curve)

- `ag_ramp_floor`:
  - Minimum always-on fraction of the cap (0..1)
  - **Important for low-step samplers (LCM, 6–10 steps):**
    - If you use `detail_late` with `ag_ramp_floor = 0`, AG can be near-zero for most of the run.
    - If you want visible changes with LCM, use:
      - `flat`, **or**
      - `detail_late` + `ag_ramp_floor = 0.1–0.3`, **or**
      - `compose_early`

**Practical rule:**  
- Want **composition/body pose** to change more → use `compose_early` (and/or raise `ag_max_ratio`)  
- Want **micro-detail** changes → use `detail_late`  
- Want consistent influence → use `flat`

---

## Swap / safety / debug options

### `safe_force_clean_swap`
For shared modes: forces a “clean” swap to avoid state leak across patch stacks.
- Safer correctness-wise
- Slower

### `uuid_only_noop`
Debug option: treats “same UUID” as no-op without extra validation.
- Useful while diagnosing swap behavior
- Can hide broken swaps if misused

### `debug_swap`, `debug_metrics`
Prints debug lines:
- patch counts and sample keys (helps confirm LoRAs are actually present on each path)
- AG magnitude metrics (helps confirm AG isn’t being capped to ~0)

---

## What to expect (so you don’t chase ghosts)

### “It looks almost like normal CFG”
Common causes:

1) **AG is being capped to ~0**
- Your debug line will show something like:
  - `ratio_eff: 0.0`, `limit: 0.0`, `scale: 0.0`
- Fix:
  - Use `ag_ramp_mode = flat`, or raise `ag_ramp_floor`, or use `compose_early`

2) **Your “bad” path isn’t meaningfully different**
- Fix:
  - Make the bad path actually different (different LoRA stack, different checkpoint, different style LoRA, etc.)

3) **You’re using very few steps (LCM) + a “late” ramp**
- With 6–10 steps, “late” is a tiny part of the run.
- Fix:
  - `flat` or `compose_early`, or set `ag_ramp_floor`

### “Changing LoRAs on the bad path does nothing”
Use `debug_swap` and look at `patch_info`:
- If bad patch count / keys change when you change LoRAs, the bad path is receiving them.
- If they never change, you’re not actually feeding the modified model into the guider.

---

## Recommended presets

### LCM / very low steps (e.g. 6–10)
- `ag_delta_mode = bad_conditional`
- `ag_ramp_mode = flat` (or `compose_early`)
- `ag_max_ratio = 0.5–1.0`
- `w_autoguide = 2.0–4.0`

### “More composition change”
- `ag_ramp_mode = compose_early`
- raise `ag_max_ratio`
- raise `w_autoguide`

### “Mostly detail change”
- `ag_ramp_mode = detail_late`
- set `ag_ramp_floor = 0.1–0.3` (especially for low-step samplers)

---

## License

MIT (see `LICENSE.txt`).

---

## Credits / Reference

AutoGuidance paper: https://arxiv.org/abs/2406.02507
