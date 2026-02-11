# ComfyUI-AutoGuidance

Adds an **AutoGuidance + CFG** **GUIDER** node for **ComfyUI**, based on:

- **Guiding a Diffusion Model with a Bad Version of Itself** (arXiv:2406.02507)  
  https://arxiv.org/abs/2406.02507

This node lets you run normal CFG on a “good” model while also using a “bad” model to compute an additional **AutoGuidance delta** that is added on top of the CFG output (optionally capped and ramped over steps).

You provide:
- a **good_model** (what you actually want),
- a **bad_model** (intentionally different/worse),
- and this guider mixes them according to the selected **delta mode**, **cap**, and **ramp**.

---

## Install

Clone into:

`ComfyUI/custom_nodes/ComfyUI-AutoGuidance`

Restart ComfyUI.

---

## Node

- **AutoGuidance CFG Guider (good+bad)** → outputs a **GUIDER**  
  Plug it into **SamplerCustomAdvanced** (GUIDER input).

---

## Quick wiring

1) Load your two models:
- `good_model`: your normal checkpoint + normal LoRA stack
- `bad_model`: same base checkpoint (often), but intentionally different LoRA stack or setup

2) Add:
- **AutoGuidance CFG Guider (good+bad)**

3) Connect:
- its **GUIDER** output → **SamplerCustomAdvanced** → **GUIDER**

That’s it.

---

## Critical: `dual_models_2x_vram` requires two distinct model instances

`swap_mode = dual_models_2x_vram` is only “dual” if ComfyUI actually loads **two separate model objects**.

ComfyUI can reuse/cache model instances when you load the *same* checkpoint path twice. If both “good” and “bad” end up referencing the same underlying model object, `dual_models_2x_vram` cannot behave as intended.

### Reliable way to force separation

Use two different checkpoint file paths:

- Copy the checkpoint file (most reliable):
  - `model.safetensors` → `model_copy.safetensors`

Then load:
- good loader → `model.safetensors`
- bad loader → `model_copy.safetensors`

> Notes:
> - Hardlinks/symlinks may or may not defeat caching depending on how your setup keys the cache.
> - If you want maximum certainty, use an actual copy (different path + different file).

---

## What makes a “good” `bad_model`

AutoGuidance only has leverage if **good_model and bad_model produce meaningfully different predictions** (at the denoiser output level). If they’re effectively the same, the delta direction is small or becomes visually negligible after other hooks.

Ways to make `bad_model` meaningfully different (examples, not promises):
- Remove some LoRAs on the bad path (good has them, bad doesn’t)
- Use a different LoRA stack on the bad path (style/character/etc.)
- Use the same LoRA at a deliberately different weight (including negative if your workflow supports it)
- Change conditioning strategy on the bad path (e.g. different prompt conditioning / different text encoder path)

The point isn’t “worse” aesthetically—it’s **different** in a way that changes denoiser predictions.

---

## Parameters

### `cfg`

Normal CFG scale. This node does not replace CFG; it adds an additional delta on top of it.

### `w_autoguide`

Paper-style parameterization:
- `w_autoguide = 1.0` → AutoGuidance delta is effectively off
- `w_autoguide > 1.0` → delta strength increases with `(w_autoguide - 1)`

Internally the delta multiplier is:
- `w = max(w_autoguide - 1, 0)`

---

## Swap modes

### `shared_safe_low_vram`

- Minimal extra VRAM
- Most conservative/compatible swapping behavior
- Slowest when bad/good stacks differ, because it needs to safely patch/unpatch on a shared model object

### `shared_fast_extra_vram`

- Optimizes shared-model swapping (may use more VRAM and device-friendly behavior)
- Still shares one model object, so there is still inherent swap overhead

### `dual_models_2x_vram`

- Requires two truly separate model instances (see section above)
- Avoids shared swapping overhead entirely
- Uses ~2× checkpoint VRAM (plus LoRAs + activations)

---

## AutoGuidance delta modes (`ag_delta_mode`)

All modes ultimately build a direction `d_ag_dir` and add:

`denoised = cfg_out + w * d_ag_dir` (optionally capped/ramped)

### `bad_conditional`

Uses a conditional-only push-away-from-bad direction:

- `d_ag_dir = (cond_good - cond_bad)`

This is the most direct “good conditional vs bad conditional” comparison.

### `raw_delta`

Uses the difference between the *guided outputs* of good vs bad (i.e. after CFG pipeline):

- `d_ag_dir = (cfg_out_good - cfg_out_bad)`

This can behave differently depending on what CFG functions/hooks your workflow uses.

### `project_cfg`

Projects the conditional push-away-from-bad direction onto the **actual applied CFG update direction** (not just `cond-uncond`):

- compute `cfg_update_dir = (cfg_out - uncond_good)`
- project `(cond_good - cond_bad)` onto `cfg_update_dir`

This keeps AutoGuidance aligned with the direction CFG is actually applying.

### `reject_cfg`

Removes the component of the conditional push-away-from-bad direction that is parallel to the **actual applied CFG update direction**:

- `d_ag_dir = d_ag_cond - proj_{cfg_update_dir}(d_ag_cond)`

This is intended to avoid AutoGuidance degenerating into “CFG++” behavior in cases where the delta is mostly parallel to CFG.

---

## Safety cap (`ag_max_ratio`)

The node can cap the AutoGuidance delta magnitude relative to the magnitude of the **actual applied CFG update**:

- `cfg_update = cfg_out - uncond_good`
- cap compares `||ag_delta||` against `ag_max_ratio * ||cfg_update||`

What this means:
- If `ag_max_ratio` is small, you may see very little effect even if good/bad differ.
- If `ag_max_ratio` is large, the delta can dominate and destabilize the result.

This repo includes debug metrics to show whether the cap is biting (see Debug section). Don’t guess—look at:
- `n_cfg`, `n_delta`, `n_delta_applied`, `scale`

---

## Ramp over steps (cap modulation)

Ramp settings scale **the cap** over denoise steps. This controls *when* in the schedule AutoGuidance is allowed to be strong.

Progress is defined as:
- `prog = step / (total_steps - 1)`
- `prog = 0` at the first step, `prog = 1` at the last step

### `ag_ramp_mode`

- `flat`: constant cap over steps
- `detail_late`: weak early, stronger late
- `compose_early`: strong early, weaker late
- `mid_peak`: strongest mid-steps, weaker at ends

### `ag_ramp_power`

Controls steepness of the selected ramp curve.

### `ag_ramp_floor`

A floor on the ramp factor (fraction of the cap that is always active).  
This prevents ramps like `detail_late` from effectively disabling early steps.

> No numeric “presets” here on purpose: the right values depend on your sampler, schedule, CFG pipeline, and any post-CFG hooks. Use the debug metrics to verify what’s actually being applied.

---

## Advanced toggles

### `safe_force_clean_swap`

Shared-model modes only. Forces conservative clean swaps between patch stacks.
- More likely to avoid state leakage across wrapper stacks
- Slower

### `uuid_only_noop`

Debug-only. Treat “same patches_uuid” as a no-op even if ownership tracking is imperfect. This can hide bugs, so only use it when debugging.

---

## Debugging and verification

### `debug_swap`

Prints information about patch stacks and swapping, including:
- patch UUIDs
- patch counts
- (optional) layer digests to confirm models/patchers actually differ

This answers: **“Are my good/bad stacks actually different?”**

### `debug_metrics`

Prints numerical diagnostics for the AutoGuidance delta and cap, including (per selected steps):
- `n_cfg` (magnitude of actual applied CFG update)
- `n_delta` (magnitude of raw AG delta before cap)
- `n_delta_applied` (magnitude after cap)
- `scale` (how hard you’re being clamped)
- `n_good_minus_bad_cond` (how different good/bad conditional predictions are)

It also prints the active `sampler_post_cfg_function` hooks at step 0, which helps diagnose workflows where post-CFG logic rescale/clamps the result and can suppress visible differences.

This answers: **“Is AG being applied, and is it being clamped?”**

---

## Troubleshooting

### “It looks the same as normal CFG”

Don’t guess—check logs:

1) **Is AG actually applied?**
- Ensure `w_autoguide > 1.0`
- Ensure `ag_max_ratio > 0.0`
- Look at `debug_metrics`:
  - if `n_delta_applied` is ~0 or `scale` is ~0, you’re effectively clamped to nothing
  - if `n_good_minus_bad_cond` is very small, your bad path isn’t meaningfully different

2) **Are good/bad models actually different in the run you think they are?**
- Look at `patch_info`:
  - if bad shows `count: 0` but you expected LoRAs, your bad stack isn’t applied
- In dual mode, check the printed digests:
  - if digests barely change even when you expect them to, your “bad” setup isn’t actually changing weights the way you think

3) **Are post-CFG hooks suppressing the effect?**
- `debug_metrics` prints `post_cfg_functions` at step 0.
- Some workflows clamp/rescale after CFG. If those hooks strongly normalize the output, visible deltas can shrink.
- For diagnosis, temporarily remove/disable post-CFG hooks in your graph and compare.

### “`dual_models_2x_vram` is slow / behaves like shared”

- Most commonly: both loaders resolved to the same underlying model instance.
- Fix: load from two distinct checkpoint paths (copy file).

### “Changing LoRAs on bad path does nothing”

- Look at `patch_info`:
  - If the bad patch count doesn’t change when you change bad LoRAs, the patches are not applied to the bad path.
- If patches change but output doesn’t:
  - inspect `scale` and `n_delta_applied`
  - inspect post-CFG hooks

---

## Credits

- Paper: https://arxiv.org/abs/2406.02507  
- This repository implements an AutoGuidance-style guider node compatible with ComfyUI’s custom sampler pipeline.
