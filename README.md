# Milkdrop On Fire

Custom `.milk` visualization presets for [Poweramp](https://powerampapp.com/) v3 / Poweramp Equalizer on Android.

The first set is **Combustion** — a digital fire effect driven by a full-range
spectrum analyzer. It's a modern take on the old Winamp fire visualizations,
built around an actual convection + incandescence model rather than a
scrolling palette.

---

## Install

Copy the `.milk` files into Poweramp's presets directory:

| Android version | Path |
| --- | --- |
| Android 10+ | `/Android/data/com.maxmpz.audioplayer/milk_presets` |
| Android 5–9 | `/Android/data/_com.maxmpz.audioplayer/milk_presets` (note the leading `_`) |

On **Android 10 and above** the OS blocks direct file manipulation in an app's
`Android/data` directory. Poweramp (build 885+) exposes a ContentProvider for
this — most file managers that support Storage Access Framework can write
there, or you can push files via the Poweramp API. Poweramp does **not**
migrate presets across an Android upgrade, so a phone upgraded from 9 to 10+
may have files sitting in the old underscore path.

Then: **Poweramp → Settings → Visualization** and pick the preset.

> The filename is metadata, not decoration. Poweramp parses `name - author.milk`
> and makes *name* and *author* separately searchable in the preset list.
> Renaming these files changes how they appear in the picker.

---

## The presets

Three files, same fire, descending order of GPU feature reliance. Poweramp
translates Milkdrop's EEL to Lua and its DirectX HLSL shaders to OpenGL ES
GLSL, and maxmpz explicitly warns that *"not all .milk presets can be loaded
or properly rendered"* as a result. So this is a diagnostic ladder as much as
an aesthetic one — **install all three and try them in order.**

| Preset | Uses | If it works |
| --- | --- | --- |
| **Combustion** | Warp shader + composite bloom | Everything translated. This is the intended look. |
| **Combustion Core** | Warp shader only | Blackbody shader is fine; `GetBlur1`/`GetBlur2` bloom taps are the problem. |
| **Combustion Lite** | No shaders (Milkdrop v1) | HLSL→GLSL translation is failing entirely. |

If **Combustion** looks right, you can delete the other two.

---

## How it works

**Fuel — Poweramp's native spectrum bars.** Stock Milkdrop only exposes three
audio bands (`bass`, `mid`, `treb`), which is nowhere near "full range."
Poweramp's `bars_*` extension does a real FFT natively at up to **128 bins**
(`bars_num_x`, the documented max). The bars are scaled short (`bars_sy=0.3`)
so they act as a bed of burning fuel rather than a bar chart, and Poweramp
draws them into the feedback buffer — so this frame's bars become next
frame's flame.

**Convection — the warp mesh.** The per-pixel equations push the image upward
with `dy` negative, at a speed that *increases with height* (buoyancy: hot gas
accelerates as it rises). Lateral waver scales with height squared, so the bed
stays steady while the tips go unstable. A slight inward pull tapers the plume.

**Incandescence — the warp shader.** This is the part that makes it read as
fire rather than orange smoke. Milkdrop's scalar `fDecay` dims toward grey and
*preserves hue*, so a white-hot pixel fades to grey. Instead, the warp shader
decays each channel at a different rate:

```hlsl
c *= float3(0.9880, 0.9550, 0.9160);   // R persists, B dies first
c  = max(c - 0.0030, 0.0);             // subtractive floor
```

Blue dies fastest and red persists longest, so colour tracks temperature the
way real incandescence does: **white → yellow → orange → red → black.** The
subtractive floor is the `- cooling` term from the classic demoscene fire
routine; without it, dim haze accumulates and the flames lose their dark edges.

**Glow — the composite shader.** Two blur taps for the heat halo, then an
exponential rolloff (`1 - exp(-c * 1.7)`) so hot cores saturate smoothly to
white instead of clipping into flat plates, while lifting dim embers enough to
stay visible.

---

## Tuning

Everything below is a one-line edit; no need to touch the shaders.

### Responsiveness

| Key | Current | Effect |
| --- | --- | --- |
| `bars_smooth` | `0.500` | Temporal smoothing of bar motion. **Start here.** |
| `bars_sensitivity` | `0.650` | Overall amplitude response. |
| `bars_bass_sensitivity` | `0.550` | Low-frequency response specifically. |

⚠️ **`bars_smooth` direction is unverified.** Poweramp documents it only as
"bars motion smoothing factor 0..1" without saying which end is smoother.
Poweramp's own examples ship `0.85`; I've set `0.50` on the assumption that
lower = more responsive. **If the fire feels sluggish, try `0.85` — and if
that's snappier, the scale is inverted from what I assumed.** Worth pinning
down first, since it gates the whole "super responsive" goal.

### Flame shape

| Key | Current | Effect |
| --- | --- | --- |
| `bars_sy` | `0.300` | Fuel bed height. Higher = more visible bars, less pure flame. |
| `per_pixel_2` `0.0130` | — | Rise speed at the tips. Higher = taller, faster flames. |
| `per_pixel_2` `0.0045` | — | Rise speed at the bed. Higher = embers linger less. |
| `per_pixel_3` `0.0075` | — | Waver amplitude. Higher = wilder, more chaotic. |
| `per_pixel_5` `0.0070` | — | Inward taper. `0` = flames rise straight up. |
| `per_pixel_7` `0.10` | — | Upper-flame turbulence. `0` = smooth laminar flow. |

### Colour and heat

| Key | Effect |
| --- | --- |
| `fDecay` / `per_frame_3` | Flame height via cooling rate. Higher = taller. |
| `warp_4` `float3(...)` | The blackbody ramp. Widen the spread between the three numbers for a more aggressive colour shift; narrow it toward equal values to fade closer to white. |
| `warp_5` `0.0030` | Subtractive floor. Higher = crisper dark edges, shorter flames. |
| `comp_4/5` multipliers | Bloom intensity. |
| `bars_color_b` / `_t` | Fuel bed gradient, `0xAARRGGBB`. |
| `bars_thr_color_*` | Colour of loud-bin tips, above the `bars_thr` fraction. |

Want blue-base gas flame instead of wood fire? Set `bars_color_b` to something
like `0xFFB0D8FF` and widen the blue decay in `warp_4`.

---

## Status

Written against the documented Milkdrop and Poweramp preset formats and
cross-checked against Poweramp's official examples, but **not yet verified on
a device** — the author of these files can't render Milkdrop. Treat the first
run as a bring-up test and expect tuning. Feedback on what actually appears
on screen is what drives the next revision.

## Sources

- [Poweramp visualization presets example + `bars_*` reference](https://github.com/maxmpz/powerampapi/tree/master/poweramp_vis_presets_example) — maxmpz
- [Milkdrop Preset Authoring Guide](https://www.geisswerks.com/milkdrop/milkdrop_preset_authoring.html) — Ryan Geiss
- [poweramp-visualizer-presets](https://github.com/SpasilliumNexus/poweramp-visualizer-presets) — SpasilliumNexus, used to confirm real-world Poweramp shader conventions

## License

MIT — see [LICENSE](LICENSE).
