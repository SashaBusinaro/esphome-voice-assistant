# Local fork: esphome-intercom audio components

Vendored from [n-IA-hane/esphome-intercom](https://github.com/n-IA-hane/esphome-intercom)
at tag `v2026.5.0` (commit `805bf37`): `audio_processor`, `i2s_audio_duplex`,
`esp_afe`.

## Why a fork instead of upgrading

Upstream `2026.6.x` deleted `i2s_audio_duplex` and rebuilt the audio layer on
Espressif GMF (`esp_audio_stack` + GMF `esp_afe`). Two reasons not to follow:

1. The new GMF dual-mic AFE structurally requires `se_enabled: true`
   (`dual-mic esp_afe requires se_enabled: true (SE/BSS is structural)` in its
   config validation). This board's A/B-tested pipeline is dual-mic with SE
   **off** (AEC → NS → AGC), which gave better wake-word + STT.
   **Nuance:** in 5.0, SE off means the AFE runs in MR mode with 1 mic channel
   (`afe_mic_channels_()`), and the RX decimator extracts only
   `tdm_mic_slot` + ref — the second mic slot is ignored. So the current
   effective pipeline is reproducible on 6.x as `mic_num: 1` +
   `tdm_mic_slot: 0` + `use_tdm_reference` (the maintained Spotpear profile
   runs GMF esp_afe single-mic with SE off and NS+AGC on). What 6.x cannot
   reproduce is the *runtime option* of flipping the SE switch to dual-mic
   MMR without reflashing. Note: no maintained 6.x profile covers the
   single-mic + ES7210 TDM-reference combination specifically.
2. The 6.x stack was heavily in flux at evaluation time (June 2026):
   stabilization/hardening commits up to days before the 6.2 tag.

The leak fixed upstream in `c695250` ("Harden AFE reconfigure and fetch ring",
explicit `CapsRingBuffer` destructor) does **not** apply here: on ESPHome
2026.5.x, `ring_buffer::RingBuffer::~RingBuffer()` already frees both the
FreeRTOS handle and the storage, and 6.x only needed its own destructor after
vendoring the base class.

## Backported / local changes vs v2026.5.0

### 1. Speaker restart stabilization (upstream `9bcec60`, i2s_audio_duplex part)

- `request_speaker_reset_` is now serviced by the audio task even while the
  pipeline is parked (`service_speaker_reset_()` called from both the outer
  `audio_task_()` loop and `audio_session_()`).
- `start()` no longer posts a speaker-buffer reset. Previously, audio written
  by `play()` between `start()` and the audio task servicing the request was
  dropped — clipping the start of TTS/playback after the pipeline had parked.
  `stop_speaker()` still posts the reset, and it is now serviced even when the
  stop parks the pipeline.
- The upstream commit's `esp_afe` half is **not** backported: it targets the
  GMF-migrated esp_afe, not the ESP-SR esp_afe in this tree.

### 2. `start_speaker()` no longer resets `play_ref_decimator_` (derived from upstream `f40e3aa`)

`start_speaker()` runs on the main thread and can be called while a mic-only
audio session is active; memset-ing the FIR delay line concurrently with the
audio task processing through it is a data race. The decimator resets in
`start()` are kept (the audio task is parked there, and clearing stale FIR
state is correct). Upstream's heap-preservation rationale does not apply to
this tree: the 5.0 `FirDecimator::reset()` is a state memset, not an
alloc/free.

## Maintenance

Upstream comparison base: `git log v2026.5.0..` in the `esphome-intercom/`
checkout. If upstream 6.x ever allows dual-mic with SE off (or an MR-equivalent
pipeline) and has settled down, re-evaluate migrating to `esp_audio_stack` —
it brings a real ES7210 driver via `esp_codec_dev` (would replace the manual
TDM register pokes in `waveshare.yaml` `on_boot`), per-channel reference gain
in YAML, and a `dma_frame_num` knob worth ~23 KB of contiguous internal heap.
