# libtaudplay

<p align="center"><em>You are currently looking at the C99 version.</em></p>
<p align="center"><a href="https://github.com/Microtone-Project/microtone-taudplay">JavaScript</a> · C99 · <a href="https://github.com/Microtone-Project/taudplay-java">Java</a> · <a href="https://github.com/Microtone-Project/taudplay-godot">Godot 4</a> · <a href="https://github.com/Microtone-Project/taudplay-libgdx">LibGDX</a></p>

---

**Play `.taud` songs from C. No tracker, no browser, no dependencies.**

The playback engine of the [Microtone](https://microtone.cc) tracker, as a
portable C99 library. Since Microtone 5.0 this is not a port of that engine but
the engine itself: the tracker and the JavaScript library
[`taudplay`](https://github.com/curioustorvald/microtone-taudplay) run these
same sources, compiled to WebAssembly. So a render here is the tracker's, note
for note and **sample for sample** — bit-identical on every platform, verified
over the whole Microtone test corpus, a differential fuzzer and an API-level
scenario test (see [Verification](#verification)).

The surface is the JS library's, kept deliberately small:

| | |
|---|---|
| **Knob** | `taud_set_voice_gain(p, v, gain, fade_seconds)` — one fader per lane |
| **Probe** | `taud_get_voice_volume(p, v)` — how loud lane `v` is, 0…1 |
| **Probe** | `taud_get_voice_pan(p, v)` — where it sits, 0 (left) … 0.5 … 1 (right) |

…plus the transport, and the sixteen **interrupts** a song can fire at the
program playing it (`Int0`…`IntF` in a note column, each with a 16-bit
argument), delivered in time with the music.

## Build

```
make            # libtaudplay.a and the taudplay CLI
```

or CMake (`add_subdirectory` works too):

```
cmake -B build && cmake --build build
```

Any C99 compiler; the only library needed is libm. Two build flags matter:

- **`-ffp-contract=off`** (both build files set it). A fused multiply-add rounds
  once where the reference rounds twice; GCC contracts by default on ARM64 and
  on any target with FMA, and one fused operation is enough to lose bit-exactness.
- **Never `-ffast-math`.** On 32-bit x87 targets also add
  `-fexcess-precision=standard` (or build for SSE2).

Nothing else about the platform is assumed: the engine keeps no global state,
and the transcendental functions come from its own fdlibm port (below), so the
output does not depend on the C library's `sin` or `exp`.

## Play something

```c
#include "taudplay.h"

TaudError err;
TaudPlayer *p = taud_open(file_bytes, file_len, NULL, &err);   /* NULL config = defaults */
if (p == NULL) { puts(taud_error_string(err)); return; }

taud_play(p);
const float *block;
while ((block = taud_render_chunk(p)) != NULL) {
    /* TAUD_CHUNK (128) frames of interleaved stereo float at 48 kHz */
}
taud_close(p);
```

The file bytes are only read during `taud_open`; everything the player needs is
copied out.

### From an audio callback

```c
void audio_callback(void *user, float *out, int frames)
{
    taud_render(player, out, frames);   /* any frame count; silence after the end */
}
```

`taud_render` buffers across block boundaries; `taud_render_s16` does the same
into interleaved `int16_t` (clamped and rounded, without the dither the clean
WAV writer adds).
`taud_render_chunk_u8` renders the Taud device's own output — noise-shaped,
dithered unsigned 8-bit.

### Render at the device's rate

```c
TaudConfig cfg;
taud_config_init(&cfg);
cfg.sample_rate = 44100;        /* or 32000, 22050, … */
TaudPlayer *p = taud_open(bytes, len, &cfg, NULL);
```

The engine renders natively at any rate — every rate-derived quantity (tick
length, filter and Amiga-LPF coefficients, anti-click ramps, the binaural HRIRs)
is computed from it — so a device running at 44.1 kHz needs no resampler. The
JavaScript library does the same when its engine rate is changed, and the two
still agree bit for bit at any rate; 48 kHz is simply the rate the JS player
uses in a browser.

### Mix it

```c
taud_set_voice_gain(p, 4, 0.0, 1.2);   /* fade lane 5 out over 1.2 s of rendered audio */
taud_set_voice_gain(p, 4, 1.0, 0.4);   /* …and back in, faster */

for (int v = 0; v < taud_channel_count(p); v++)   /* duck everything but the drums */
    if (v != DRUMS) taud_set_voice_gain(p, v, 0.3, 0.8);
```

A fader reaches everything its lane spawned — NNA ghosts and metainstrument
layers included — so a faded lane really does take its whole sound with it.

A song's instrument mixing chains (`IMix`: per-instrument or group buses of
trim, expander/gate, EQ, dynamic EQ, multiband transient shaper, multiband
compressor, unison, saturator, tape delay, reverb, limiter and mix tool) and its mastering chain (`sMst`) are installed
from the file and played as the tracker plays them; there is nothing to set.

### Let the song call you

```c
static void on_cue(int n, unsigned arg, void *user) { flash_light(arg); }

taud_set_interrupt(p, 0, on_cue, NULL);   /* NULL fn unregisters */
```

Handlers run on the rendering thread, from inside the render call, once per
128-frame block the interrupt fired in (2.7 ms at 48 kHz), lowest number first.
If one interrupt fires twice inside a block you are called once, with the later
argument. A handler that must not block the audio thread should post the event
somewhere and return.

### Watch it

```c
for (int v = 0; v < taud_channel_count(p); v++)
    draw_meter(v, taud_get_voice_volume(p, v), taud_get_voice_pan(p, v));
```

The probes describe the state at the end of the last rendered block.

## The CLI

```
taudplay song.taud                          describe the file
taudplay song.taud -o song.wav              render song 0 to a 16-bit WAV
taudplay song.taud --gritty -o song.wav     …or to the device's own 8-bit sound
taudplay song.taud -s 2 -t 60 -o x.wav      song 2, at most 60 seconds
taudplay song.taud -r 44100 -o x.wav        render natively at 44.1 kHz
taudplay song.taud --wav-rate 44100 -o x.wav  render at 48 kHz, resample the WAV
taudplay song.taud --binaural -o x.wav      head-model downmix of a surround song
taudplay song.taud --mute 0,3 -o x.wav      without lanes 1 and 4
taudplay song.taud --interrupts             print the song's interrupt cue sheet
taudplay song.taud --raw x.f32              interleaved float32, engine rate
taudplay song.taud --raw - | ffplay -f f32le -ar 48000 -ac 2 -i pipe:0
taudplay song.taud --raw - --s16 | aplay -f S16_LE -r 48000 -c 2
taudplay song.taud -o - | ffplay -i pipe:0  WAV to stdout
```

`--raw` streams as it renders (little-endian whatever the host), so a player
on the other end of a pipe starts at once; `-o -` has to render the whole song
first, because a WAV header carries the length. With `-r` the stream is at that
rate, so pass the same `-ar` to ffplay. Messages, and the `--interrupts` log
when stdout carries audio, go to stderr. `-t` still caps the length (600 s by
default).

## API

All in [`include/taudplay.h`](include/taudplay.h).

- **Lifetime:** `taud_config_init`, `taud_open`, `taud_close`, `taud_error_string`
- **The file:** `taud_title`, `taud_song_count`, `taud_song_info`, `taud_select_song`,
  `taud_current_song`, `taud_sample_rate`
- **Transport:** `taud_play` (from the top), `taud_stop`, `taud_seek_cue`,
  `taud_set_volume`, `taud_set_binaural`; `taud_is_playing`, `taud_cue`, `taud_row`,
  `taud_bpm`, `taud_speed`, `taud_channel_count`
- **Knob / probes:** `taud_set_voice_gain`, `taud_get_voice_gain`,
  `taud_get_voice_volume`, `taud_get_voice_pan`
- **Interrupts:** `taud_set_interrupt`, `taud_clear_interrupts`
- **Rendering:** `taud_render_chunk`, `taud_render`, `taud_render_s16`,
  `taud_render_chunk_u8`
- **Utilities:** `taud_encode_wav_quality` (with the library's Kaiser-sinc
  resampler) and `taud_encode_wav` (its clean case), `taud_gain_to_fader`,
  `taud_fader_to_gain`

A WAV comes in two qualities, the JS library's two. `TAUD_WAV_CLEAN` (the
default) is 16-bit with a plain TPDF dither of one LSB; `TAUD_WAV_GRITTY` is
8-bit through the device's own noise-shaped dither, so at the engine's rate it
is byte for byte what `taud_render_chunk_u8` delivers. Each dither starts
afresh, so the same floats always make the same file.

## Embedding notes

- **Threads.** No global state: each `TaudPlayer` is independent and may live on
  its own thread. One player must not be called from two threads at once;
  calling the knobs from a UI thread while the audio thread renders needs your
  own lock (or a command queue).
- **Memory.** The format's sample pool is a flat 8 MB image. It is decompressed
  once at open, then trimmed to its last non-zero byte (a read past the end is
  0, which is what the full pool holds there), so a player keeps about 3 MB of
  its own state plus the song's actual sample data (each voice carries room for
  an OPL channel, a PSG pair and a formant tract with a chip channel of its
  own for a source, so a sounding one never allocates). Peak memory
  during `taud_open` is about 11 MB, while the full image is decompressed.
  An instrument mixing chain adds its bus when its song loads: a few hundred
  KB, or about 1.5 MB per channel of the bus when it holds a tape delay (four
  seconds of binary64) — so a delay on a binaural render of a 3-D surround
  song, sixteen channels wide, is about 25 MB. A blank pattern is stored once,
  however many slots hold one.
- **Allocation on the render path.** None in steady state; the instrument
  buses are built at the first block after a song is selected or the monitor
  mode changes. The voice pool grows
  (in blocks of 64 voices) only when a note storm outgrows every earlier one,
  and an instrument's invert/modification mask (8 KiB) is allocated the first
  time an effect needs it. Pools are reclaimed by marking what the lanes can
  still reach.
- **Hostile input.** Every length and offset in the container is checked, the
  zstd and inflate decoders are bounds-checked, and the engine is exercised
  under AddressSanitizer + UBSan with thousands of corrupted files
  (`make robust`). A malformed field (an envelope loop point past its 25
  nodes, say) is read as the nearest valid entry; no well-formed file is
  affected by that.
- **Randomness.** Instruments may ask for random volume/pan swing, and some
  effects are randomised. Those draw from a seeded mulberry32
  (`cfg.random_seed`, default 1) or your own `cfg.random` — so a render is
  reproducible. (The tracker and the JS library seed the same generator
  at random, which is what swing is for; seeded alike —
  `setRandomSource(makeSeededRandom(1))` there — they render exactly this.)
- **Speed.** About 50× real time on one core of a desktop x86-64 for an ordinary
  32-lane song (`-O2`), ~25× for a dense one full of metainstruments and SF2
  filters.

## Numerics

The engine does its arithmetic in binary64 and narrows the mix to binary32 once
per frame; the order of every sum is part of the result, which is why the
library must be built without fused multiply-adds and without fast-math.

It also brings its own mathematics. The engine was first written in JavaScript,
and its numbers were defined by what V8 computes for `Math.sin`, `cos`, `tan`,
`exp`, `log`, `log10`, `log2`, `acos`, `asin`, `atan2`, `tanh`, `log1p`,
`expm1` and `pow`: V8's own fdlibm port, which a platform's libm differs from
in the last bit for a few per cent of arguments — enough to move a sample by
one binary32 ulp here and there. [`src/jsmath.c`](src/jsmath.c) carries those
functions, verified bit-for-bit against V8 over millions of arguments
(`tests/jsmath_check`), so the output does not depend on the target's libm:
the same bits from a desktop, a phone, a console and a browser.

`pow` follows V8's copy exactly, including the one place it groups the final
step differently from Sun's original, which moves about 4% of results by an
ulp. (Current V8 — 14.x, as in Node 26 — calls the platform's `std::pow`
instead unless run with `--no-use-std-math-pow`; `tests/jsmath_check` runs Node
with that flag.)

`-DTAUD_LIBM_POW` (CMake: `-DTAUDPLAY_LIBM_POW=ON`) uses the platform's `pow`
instead, at the cost of the output's platform independence: such a build is no
longer bit-identical to the tracker.

## Verification

`make check` (needs Node ≥ 22 and a checkout of the JS library at
`../microtone-taudplay`, or `JS=path`) compares this build with the JS
library, which runs the same sources as WebAssembly: what it proves is that
your compiler and your platform produce the engine's bits. It runs:

- `tests/jsmath_check` — the math functions against V8.
- `tests/wavcheck` — the WAV encoder, clean and gritty, at the engine rate and
  resampled, against the JS library's `encodeWav`, byte for byte.
- `tests/run.sh` — every song of every corpus file, stereo fold-down and
  binaural, JS vs C, sample for sample. Also takes any files:
  `sh tests/run.sh path/to/*.taud`.
- `tests/scenario.sh` — the public API against the JS `TaudRenderer`: faders
  and fades, master volume, seeking, song switching, stop and replay, probes,
  interrupts, the dithered 8-bit output, and non-48 kHz engine rates. Audio and
  every probe reading (as raw bits) must match.
- `tests/fuzz.sh` — differential fuzzing: `tests/gen_fuzz.mjs` writes random but
  valid songs (both cell formats — the wide cell's notes with their sixteenths,
  65536-TET, on half the cells — 32/64 lanes, stereo/planar/spatial, envelopes
  in 6 and 8 bits,
  IT/SF2 filters, NNA and duplicate checks including the lane check, Ixmp
  patches including stereo ones, layered and FM metainstruments with chip
  operators among them, table kinds the engine does not know, OPL channels of
  every kind, PSG pairs of every waveform with their `S $Axxx` register writes,
  formant instruments with every consonant manner — syllabic, nasalised,
  glottalised, breathy, aspirated, sung from a sample or from a chip, with
  their `S $9xxx` nasalisation, r-colouring, held codas and glide times — FM
  racks with DSP-only principals and programs filling their area to byte 253,
  every effect command, mastering chains, and instrument mixing chains of
  every piece, in both pipeline orders, separate and bus mode, with unknown
  pieces and randomly corrupted blocks among them, and lyric instruments —
  `ILyr` entries with recipes shared, missing and repeated, on formant
  instruments and on others, with empty, short and duplicate entries among
  them), and both implementations must render them identically. `FUZZ_IMIX=1` puts chains in every song,
  `FUZZ_IMIX=0` in none, and `FUZZ_IMIX_KINDS=tsh1,expd` (any FourCCs) draws
  their equipment from those pieces alone.
  `FUZZ_LYRIC=1` gives every song lyrics, `FUZZ_LYRIC=0` none.
  `FUZZ_IPIT=1` gives every song pitch extensions (`IPit`: base fields, Ixmp
  patches, layer and rack tables; fractions, sixteen-octave steps out of a
  note value's range, stale guards, groups that do not fit, a reserved flag
  and a cut-off group), `FUZZ_IPIT=0` none.
  `FUZZ_S9=1` makes `S $9xxx` half of every effect command, on most rows;
  `FUZZ_CUTS=1` does the same for `S $70`…`S $76` (past-note actions, New Note
  Action overrides), best with `FUZZ_INSTS=layered`.
  `FUZZ_GLIDES=1` puts tone portamentos (`G`, `L`) on lanes that keep going
  silent, half of them delayed by `S $Dx` in a wide-cell song — a lyric
  instrument's glide with nothing sounding — best with
  `FUZZ_LYRIC=1 FUZZ_INSTS=formant`.
  `FUZZ_REPEAT=1` plays four pitches only, each with its own sixteenths, on
  three instruments whose Duplicate Check is the same-note kind, with
  glissando (`S $1x`) switched on and off — the same pitch keeps coming back,
  one of them just past a semitone's rounding edge.
  `FUZZ_INSTS=opl`, `psg`, `formant`, `fm` or `layered` narrows the patterns to one kind;
  `FUZZ_WILD=1` fills the OPL, PSG and formant blocks with random bytes.

At the time of writing all of it is bit-identical against Node 26 (V8 14.6):
the seventeen files of the Microtone corpus and demos (every song, both monitor
modes), and hundreds of fuzzed songs.

## Relationship to Microtone and the JS library

These sources are `core/engine/c/` of the
[Microtone repository](https://github.com/Microtone-project/Microtone.js),
copied (`tools/make-taudplay-c.js` there); the engine is developed in that
tree, and the tracker's golden renders are its contract. Up to version 4.10
the engine was JavaScript and this library was its port, file for file —
`sampler.c` was `sampler.js`, `tick.c` was `tick.js` — which is where the
layout and the function names come from.

The library's surface is `include/taudplay.h` and `src/player.c`: the
`TaudRenderer` half of the JS library. Its browser half (`TaudPlayer`, the
AudioWorklet) has no counterpart here: its job, pulling blocks from an audio
callback, is `taud_render`.

`src/` also holds what only the WebAssembly build is made of, which the
library does not compile: `wasm_api.c` (the calls a browser host makes —
uploads, jam voices, meters), `snapshot.c` (the state block the tracker's
display reads) and `wasm_parts.c` (hooks for the tracker's tests).

In step with taudplay 5.0.0 (Microtone.js 5.0.0, engine 74c9468d1933).

## Licence

LGPL-3.0-or-later — see `COPYING.LESSER` (and `COPYING` for the GPL text it
builds on). You may link this library into a proprietary program; changes *to
the library itself* must be shared.

Where the LGPL does not fit — a console title, a static build, changes to the
library you cannot publish — taudplay-c is also available under a proprietary
licence: write to <microtone@curioustorvald.com>. It is offered beside the
LGPL, never instead of it.

Contributions are welcome once their author has agreed to the
[Microtone Contributor Licence Agreement](https://github.com/Microtone-Project/Microtone.js/blob/master/CLA.md):
you keep your copyright, and what is merged stays LGPL. Changes to how the
engine behaves go to [Microtone.js](https://github.com/Microtone-Project/Microtone.js),
where it is authored; this library follows it.

`src/jsmath.c` is derived from fdlibm, Copyright © 1993 Sun Microsystems, Inc.
("Permission to use, copy, modify, and distribute this software is freely
granted, provided that this notice is preserved"). The binaural filter set in
`src/hrir_sadie.c` is the GoogleVR/SADIE set, © 2017 Google Inc. and the
University of York, Apache-2.0.

---

Songs are made with **Microtone** — a tracker for the notes a piano cannot
play, free and in your browser at [microtone.cc](https://microtone.cc).
