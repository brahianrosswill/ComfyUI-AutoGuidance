# ComfyUI-AutoGuidance

Adds an **AutoGuidance + CFG** **GUIDER** node for **ComfyUI**, based on the paper:

- **Guiding a Diffusion Model with a Bad Version of Itself** (arXiv:2406.02507) :contentReference[oaicite:1]{index=1}
- Link: https://arxiv.org/abs/2406.02507

This node lets you guide a “good” diffusion model using a “bad” version of it, while still using normal CFG. In practice, you provide:
- a **good_model** (the one you want),
- a **bad_model** (intentionally worse / different),
- and the node injects an AutoGuidance delta on top of CFG (with safety caps and ramp controls).

---

## Install

Clone into:

`ComfyUI/custom_nodes/ComfyUI-AutoGuidance`

Then restart ComfyUI.

---

## Quick Start (how to wire it)

1) Create your two models:
- **good_model**: checkpoint + your normal LoRA stack (character LoRA, LCM/Lightning, style, etc.)
- **bad_model**: checkpoint + a *worse/different* stack (examples below)

2) Add the node:
- `AutoGuidance CFG Guider (good+bad)`

3) Plug it into:
- `SamplerCustomAdvanced` (GUIDER input)

That’s it.

---

## Extremely important: `dual_models_2x_vram` REQUIRES a second checkpoint file path

If you pick **`swap_mode = dual_models_2x_vram`**, you **must** load the checkpoint **twice as two distinct model instances**.

In ComfyUI, if both loaders point to the *same* checkpoint path, Comfy’s model caching can yield a shared underlying model object. In that situation, `dual_models_2x_vram` **cannot actually be dual-model** and will **effectively behave like a shared-mode internally**, which is **dramatically slower** (people commonly see “order-of-magnitude” slowdowns vs normal CFG).

### Do this (recommended)
Make a second checkpoint file path and load each one:

- Copy the checkpoint file (simple + reliable), e.g.
  - `model.safetensors` → `model_copy.safetensors`
- Or use a filesystem hardlink if you know what you’re doing and your OS/filesystem supports it.

Then in ComfyUI:
- `CheckpointLoaderSimple` → load `model.safetensors` (good path)
- `CheckpointLoaderSimple` → load `model_copy.safetensors` (bad path)

Now `dual_models_2x_vram` will behave correctly and avoids the expensive patch/unpatch swapping overhead.

**VRAM note:** dual-model mode uses ~2× the base checkpoint VRAM (plus LoRAs, plus activations).

---

## Choosing a good “bad_model” (what actually works)

AutoGuidance only does something meaningful if **good_model and bad_model produce meaningfully different predictions**.

Practical bad-model recipes:
- **Remove** the quality/style LoRA(s) from bad_model (good has them, bad doesn’t)
- Use a **weaker/incorrect style LoRA** on bad_model
- Use the same LoRA but at a **wrong weight** (too high, too low, or negative)
- If good uses LCM/Lightning, try bad without it (or vice versa) to force a strong mismatch
- Use a “bad prompt conditioning” strategy (less common): keep model same but feed bad_model a deliberately worse positive conditioning

If you set bad_model too similar to good_model, AutoGuidance will be subtle no matter what.

---

## Node Parameters

### Core knobs

#### `cfg`
Normal classifier-free guidance scale (same as usual).

#### `w_autoguide`
Paper-style weight:
- `w_autoguide = 1.0` → AutoGuidance effectively off
- `2.0` → moderate
- `3.0+` → strong

Internally this is used as a multiplier for the AutoGuidance delta direction (on top of CFG).

---

## Swap modes (performance + correctness)

### `shared_safe_low_vram` (default)
- Lowest VRAM overhead
- Most compatibility / correctness across Comfy variants
- Can be **very slow** because it must swap LoRA stacks on a shared model safely

### `shared_fast_extra_vram`
- Faster swapping than safe mode (uses inplace updates / device-friendly behavior)
- Still shares one model object, so there’s still unavoidable overhead
- Uses extra VRAM

### `dual_models_2x_vram`
- Fastest **when you truly load two separate checkpoint instances** (see the “second checkpoint path” section above)
- Uses ~2× checkpoint VRAM
- Avoids the expensive shared swapping path entirely

---

## AutoGuidance delta mode (`ag_delta_mode`)

### `bad_conditional` (recommended default)
Uses the most “LoRA-sensitive” direction in practice:
- compare **good conditional output** vs **bad conditional output**

This tends to show the biggest differences when you change LoRAs on the bad path.

### `raw_delta`
Uses a raw difference direction between guided outputs. Can be harsher / less predictable.

### `project_cfg`
Projects the “push-away-from-bad” direction onto the **actual CFG update direction**.
This keeps changes more aligned with CFG, often more conservative.

### `reject_cfg`
Removes the component of the “push-away-from-bad” direction that is parallel to the **actual CFG update direction**.
This can increase composition changes when AG otherwise behaves like “CFG++”.

---

## Safety cap: `ag_max_ratio`

AutoGuidance is capped relative to the magnitude of the **actual CFG update** (`cfg_out - uncond_good`):

- Higher `ag_max_ratio` = stronger visible effect (up to destabilization if extreme)
- Lower `ag_max_ratio` = subtler

A good starting range:
- SDXL: `0.35` to `0.80`

If your output looks “basically like normal CFG”, this cap (and/or the ramp) is the first place to look.

---

## Ramp over steps (this controls “composition vs detail”)

These parameters scale **the cap** (`ag_max_ratio`) over denoise progress.

Important concept:
- Progress is defined as **0 early (high sigma)** → **1 late (low sigma)**

### `ag_ramp_mode`
- `flat`  
  Constant cap across all steps.  
  **Use this if you want composition changes.**
- `detail_late`  
  Weak early, strong late → mostly affects fine detail.  
  If `ag_ramp_floor = 0`, early steps can get near-zero AG and the image composition will stay very close to normal CFG.
- `compose_early`  
  Strong early, weaker late → pushes structure/composition more than detail.
- `mid_peak`  
  Strongest mid-steps, weaker at ends.

### `ag_ramp_power`
Controls steepness (shape). Higher = more extreme.

### `ag_ramp_floor`
Minimum always-on fraction of the cap.
- `0.0` means the ramp can reduce AG close to zero at some parts of the schedule.
- If you use `detail_late` but still want *some* early influence, set `ag_ramp_floor` to something like `0.10–0.30`.

### Practical presets

**If you want bigger composition changes (more like NAG-style behavior):**
- `ag_ramp_mode = flat` OR `compose_early`
- `ag_ramp_floor = 0.10–0.30` (optional but helpful)
- Raise `ag_max_ratio` before raising `w_autoguide` to absurd values

**If you mainly want detail/texture improvements while keeping composition stable:**
- `ag_ramp_mode = detail_late`
- `ag_ramp_floor = 0.0–0.15`
- `ag_ramp_power = 2.0–4.0`

---

## Advanced / safety toggles

### `safe_force_clean_swap`
Only relevant in shared-model modes. Forces clean swaps between patch stacks to avoid state leakage / washed-out results on some Comfy builds.
- Safer
- Slower

### `uuid_only_noop`
Debug option. Treat “same patches_uuid” as a no-op even if ownership tracking is imperfect.
Use only for debugging.

### `debug_swap`
Prints swap/patch diagnostics (patch counts, uuids, etc.)

### `debug_metrics`
Prints direction / magnitude diagnostics (useful for confirming AG is actually being applied).
It also logs active `sampler_post_cfg_function` hooks at step 0, which helps diagnose post-CFG rescale/clamp behavior that can suppress visible AG effects.

---

## Troubleshooting

### “It looks almost the same as normal CFG”
Common causes:
1) **Your ramp/cap is effectively zero early**, so composition won’t move.
   - If you used `detail_late` with `ag_ramp_floor = 0`, early AG can be near-zero by design.
   - Fix: use `flat` or `compose_early`, or raise `ag_ramp_floor`.

2) **Bad model is not actually “bad enough”**
   - Make bad_model deliberately wrong for one test (strong wrong LoRA, remove key LoRA, etc.)

3) **You’re not truly in dual-model mode**
   - If you selected `dual_models_2x_vram` but did not load the checkpoint via two different file paths, you’re not actually dual-model and performance/behavior will be off.

### “dual_models_2x_vram is insanely slow”
That happens when both loaders resolve to the same underlying model instance (same checkpoint path / caching). Fix it by creating a second checkpoint file path and loading both.

### “My LoRA changes on the bad path do nothing”
Check `debug_swap` / `patch_info`:
- If patch counts change when you edit bad LoRAs, the patches are applied.
- If the image still doesn’t move, your ramp/cap likely prevents early-step influence (composition stays locked).

---

## Credits

- Paper: **Guiding a Diffusion Model with a Bad Version of Itself** (Karras et al.). :contentReference[oaicite:2]{index=2}
- This repository implements an AutoGuidance-style guider node for ComfyUI’s custom sampler pipeline.

---
