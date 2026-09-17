<!-- Generated from index.html — the interactive figures live there. -->

*A digital media primer, for people who write software*

# Digital audio and video represent signals as *arrays of samples*.

A media file stores sampled audio and video together with information needed to interpret and play them. This first part describes PCM parameters, byte order, bit depth, quantisation error and dither. Companding, reconstruction and the whole video half — pixel aspect ratio, interlacing, gamma, colour representation, pixel formats and containers — are listed at the end and will follow.

Xiph.Org's *A Digital Media Primer for Geeks* by Christopher "Monty" Montgomery (2010), adapted here with interactive figures.

The primer introduces the concepts and engineering constraints behind common audio and video formats.

This part contains three interactive figures. Images are generated procedurally, audio is synthesised at runtime, and colour-space calculations run in the browser. Additional derivations appear in **Go deeper** panels, which can be read independently of the main text.

---

*§1*

## Digital signals and repeatable copying

Digital communication predates recorded analogue audio. By the 1860s, telegraph systems were carrying multiplexed digital signals across continents. These systems represented messages using discrete states.

A major advantage of digital storage is **repeatable copying**. In an analogue chain, amplifiers, cables, and recording media add noise and distortion. These errors accumulate during transmission and copying, and separating them from the original signal is difficult.

A digital transmission uses discrete signal levels. A receiver can recover the intended value when noise stays within the decision threshold, allowing each stage to regenerate the data. A file copied without errors remains bit-identical through successive copies. Conversion between analogue and digital signals introduces quantisation and hardware errors; the sample rate also limits bandwidth. [The companion page on sampling](/sampling) explains the conditions for reconstructing a signal from its samples.

> Raw HD video can require roughly 700 megabits per second, about 500× the data rate of CD audio. Real-time video processing became practical later than audio processing because it required substantially more computing power, storage, and data-transfer capacity.

---

*§2*

## PCM parameters and byte order

**Pulse Code Modulation** (PCM) represents a signal as measurements taken at fixed intervals. Reading a PCM stream requires its sample rate, sample format, channel count, and byte order.

The name comes from 1930s telephony, where it described a way of sending a signal down a line rather than a way of storing it. Related methods carried the signal on a train of pulses by varying one property of each pulse: its height in pulse-amplitude modulation, its width in pulse-width modulation, its timing in pulse-position modulation. PCM instead converted each measurement to a binary *code* and sent that code as a group of on-or-off pulses. The signal *modulates* the pulse train, the pulses carry a *code*, hence the name. The transmission part is no longer relevant, and the term now refers to the representation itself: one integer per sample, uncompressed.

**Sample rate** is the number of measurements per second. Half the rate is the upper frequency boundary for reconstruction. **Sample format** specifies the bit depth and representation: signed or unsigned integer, or floating point. It determines the available levels and *dynamic range*. **Channel count** specifies the number of simultaneous streams. Interleaved stereo stores left and right samples in alternating order. Values wider than 8 bits also have a **byte order**. Standard WAV PCM is little-endian; AIFF PCM is big-endian. Reading samples in the wrong byte order can produce loud noise.

> A 16-bit sample occupies two bytes, and a format must state which byte comes first. **Little-endian** stores the least significant byte first; **big-endian** stores the most significant byte first. The value 4660, or `0x1234`, is stored as the bytes `34 12` in little-endian order and `12 34` in big-endian order. x86 and most ARM systems are little-endian, as is WAV. Network protocols and AIFF are big-endian. The names come from *Gulliver's Travels*, where two factions dispute which end of an egg to open. Reading with the wrong order swaps the two bytes of every sample, so the low byte, which changes from one sample to the next, becomes the high byte. The waveform becomes a sequence of large jumps, heard as loud broadband noise. Single-byte formats have no byte order.

Multiplying sample rate, bits per sample, and channel count gives the bitrate. Dividing by eight gives bytes per second:

```go
package main

import "fmt"

func main() {
	sampleRate := 44100                                  // samples per second: CD audio
	bits := 16                                           // bits per sample
	channels := 2                                        // stereo
	bytesPerSecond := sampleRate * (bits / 8) * channels // bits/8 = bytes per sample
	fmt.Println(bytesPerSecond, "B/s")                   // 176400 B/s  = 1.41 Mbit/s, or 10 MB per minute
}
```

> **Figure 1 · live — PCM parameters and data size**
>
> The sliders show the waveform being sampled and rounded, and the resulting size in bytes.
>
> Sample rate sets the frequency range. Bit depth sets the quantisation step size and affects the noise floor. "Distinct levels" is the number of values the sample integer can hold, 2 to the power of the bit depth: 256 at 8 bits, 65,536 at 16 bits, about 16.8 million at 24 bits. Every stored value is one of these levels. Adjust each slider to compare changes in the sampled waveform and data size.
>
> *(interactive figure — see the web page)*

### Common sample rates

| Rate | Use and background |
|---|---|
| **8 kHz** | Telephony. Speech is intelligible with about 4 kHz of bandwidth, so 8 kHz is the smallest rate that carries it. |
| **44.1 kHz** | CD audio. The rate comes from early digital recorders that stored audio on video tape, where each video field had to contain a whole number of samples. |
| **48 kHz** | Professional video and general audio production. Its relationship to common frame rates simplifies alignment between audio and picture. |
| **96 / 192 kHz** | Production formats. Higher rates allow wider transition bands for anti-aliasing and reconstruction filters and provide headroom for signal processing. |

> Frequencies above half the sample rate can *alias* into the representable band, producing tones at different frequencies. An anti-aliasing filter limits the input bandwidth before sampling. [The companion explainer on the sampling theorem](/sampling) covers this process in detail.

---

*§3*

## Bit depth and quantisation error

Bit depth specifies how many levels an integer sample can represent. At a fixed full-scale amplitude, increasing the bit depth places those levels closer together and reduces the error introduced by rounding.

The **quantisation step** is the distance between two neighbouring levels. For a range of −1 to +1 and *b* bits it is 2 ÷ 2*b*: 0.125 at 4 bits, about 0.00003 at 16 bits. It is also called one LSB, for least significant bit, since it is the change produced by flipping the lowest bit of the sample. Rounding a sample to the nearest level introduces an error bounded by half a step. Each additional bit halves the step size. When the rounding error behaves as uniformly distributed noise, this lowers the noise floor by about 6 dB per bit:

**dynamic range≈6.02b+1.76 dB**

- **b** — bits per sample.

- **6.02** — one extra bit doubles the number of levels; doubling is 6.02 dB.

- **1.76** — a constant arising from the power ratio between a full-scale sine and uniformly distributed rounding error.

The approximate range is 50 dB for 8-bit audio, 96 dB for 16-bit audio, and 144 dB for 24-bit audio. Recording and production commonly use 24-bit samples to accommodate low recording levels and repeated processing. The usable range of a recording also depends on microphone noise, converter performance, and the recording environment.

> Signal levels on this page are given in **dBFS**, decibels relative to full scale. Full scale is the largest value the sample format can hold, ±1.0 in the figures, and a signal whose peaks reach it is at 0 dBFS. Quieter signals have negative values: the level is 20·log10(amplitude ÷ full scale), so −6 dBFS is half the amplitude, −20 dBFS one tenth, and −60 dBFS one thousandth. The unit makes levels and bit depth directly comparable. Each bit adds about 6 dB, so the quantisation noise of a 16-bit format sits near −96 dBFS, and a tone at −90 dBFS is about 6 dB above it.

### Quantisation error as an added signal

Bit depth is sometimes described as the precision of each sample, as if a 24-bit recording were a sharper copy of a 16-bit one. A more useful description treats the rounding error as a second signal added to the first: the stored value minus the true value, which is the orange trace in Figure 2. Its size is always within half a step, so bit depth alone does not determine how audible it is. Its *character* does, and the character depends on the signal.

A loud or complex signal crosses many levels in an irregular sequence, so the error at each sample is effectively random. Random error is noise. It sits at a fixed level, about 6 dB lower for each additional bit, and is heard as faint hiss. This is the usual case, and it is what "bit depth sets the noise floor" refers to. A quiet or simple signal crosses the same one or two levels in the same way on every cycle, so the error repeats with the signal. Repeating error is distortion: tones at multiples of the signal's frequency that were not in the source, which hearing detects more readily than noise of the same power. Figure 2 shows this near −20 dBFS. A signal smaller than half a step rounds to zero at every sample and is removed entirely.

Bit depth changes only the step size. Finer steps reduce the maximum error and lower the noise floor, while full scale stays where it is. Bit depth therefore sets the floor rather than the ceiling, and dynamic range is the distance between the loudest representable signal and the rounding noise beneath it. Low bit depths are exposed by quiet passages rather than loud ones.

Sampling and quantisation are two separate roundings of one waveform. Sampling rounds time, keeping the value only at fixed instants. Quantisation rounds amplitude, keeping only one of the allowed values at each instant. [The sampling page](/sampling) shows that rounding time loses nothing when the rate is sufficient. Rounding amplitude always discards something, and dither, the subject of §4, changes what is discarded from distortion into noise.

> **Figure 2 · audio + live — A signal and its quantisation error**
>
> A sine wave is rounded to the nearest of the 2bits levels. Top: the original and the rounded version, with the levels drawn as dashed lines; the vertical axis zooms in as the signal gets quieter. Bottom: the rounding error, rounded minus original, measured in quantisation steps. Rounding is never off by more than half a step, so the error's size is fixed and its *shape* is what changes. Lower the signal level and watch the shape. The buttons play the signal at the selected level and bit depth, with the dither checkbox applied. Playback gain is normalised so quiet settings remain audible; compare the character of the sound rather than its loudness. A steady sine repeats every cycle, so its rounding error is periodic at any level. The two-tone signal, 440 and 623 Hz, never repeats, so its error can behave as noise.
>
> At 0 dBFS with 4 bits the sine spans 8 steps and crosses every level. The error is a rapid sequence of small ramps, one per level crossing, and resembles random noise. This is the regime the 6 dB-per-bit estimate describes. Near −20 dBFS the sine spans less than one step, the rounded output becomes a two- or three-level square wave, and the error becomes a periodic waveform locked to the signal. That error is distortion: harmonics of the signal that were not in the source. Below about −24 dBFS the amplitude is under half a step, every sample rounds to zero, and the error is the negative of the signal. "Levels used" counts how many levels the rounded output lands on. With dither enabled, random noise is added before rounding, the output flickers between neighbouring levels in proportion to the input, and the error loses its lock to the waveform. With the single sine, the full-scale error is a dense set of harmonics and is heard as a change of timbre. With the two tones, the full-scale error decorrelates and plays as broadband hiss, while the low-level error still collapses into distortion. Section 4 describes dither in detail.
>
> *(interactive figure — see the web page)*

| Format | Range | Where it appears |
|---|---|---|
| **8-bit linear** | ~50 dB | Legacy audio and sound effects. Quantisation noise can be audible in quiet passages. |
| **8-bit µ-law / A-law** | ~14 bits' worth | Telephony. Non-uniform levels provide finer resolution for quiet signals — see §5. |
| **16-bit signed** | ~96 dB | CD audio and common delivery formats. Full scale is 0 dBFS; lower levels have negative dBFS values. |
| **24-bit signed** | ~144 dB | Recording and production. Additional range for low-level signals and processing. |
| **32-bit float** | very large | Mixing and mastering. ±1.0 corresponds to 0 dBFS. Floating-point storage can retain values beyond that level for later gain reduction. |

With b bits the step size is q=2/2b for a signal spanning −1 to +1. Assume the rounding error is uniformly distributed across ±q/2 as an approximation for signals that cross many levels. A uniform distribution of width q has variance q2/12, so the noise power is q2/12.

A full-scale sine has power 1/2. The ratio is q2/121/2=2q212=23⋅22b, and 10log10(2322b)=20blog102+10log101.5=6.02b+1.76 dB. The constant 1.76 equals 10log10(3/2), using the sine's RMS power as the signal reference.

This estimate assumes uniformly distributed error that is uncorrelated with the signal. Quiet or simple signals can produce repeating error patterns, limiting the estimate's usefulness for low-level behaviour. Appropriate dither removes the signal dependence of the error's mean and variance.

---

*§4*

## Dither: adding noise to reduce distortion

Dither is random noise added before quantisation to control the statistical properties of the rounding error. Section 3 described that error as a second signal whose character depends on the input: noise for a loud or complex signal, distortion for a quiet or simple one, silence below half a step. Dither makes the character independent of the input. With dither, the error is noise in every case, at a fixed level set by the bit depth.

Figure 2 shows the undithered case. A quiet sine crosses the same few quantisation levels in each cycle, so the error repeats with the waveform and produces **harmonic distortion**: additional tones at multiples of the original frequency. Even at full scale, a steady sine's error is periodic, a dense set of small harmonics. Enabling dither in Figure 2 changes both cases: the added noise decides each rounding at random, the output flickers between neighbouring levels with probabilities that follow the input, and the error loses its relation to the waveform.

The following example adds triangular noise spanning ±1 quantisation step *before* rounding:

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
)

func main() {
	x := 0.3      // input sample, in the range -1..1
	levels := 8.0 // quantiser step count: how many values the format can store

	// undithered: rounding error can be correlated with the signal
	out := math.Round(x*levels) / levels

	// dithered: error becomes uncorrelated noise, and the signal survives below one step
	d := (rand.Float64() + rand.Float64() - 1) / levels // triangular noise, plus or minus 1 LSB
	outDithered := math.Round((x+d)*levels) / levels

	fmt.Println(out, outDithered)
}
```

Dither replaces signal-correlated distortion with broadband noise at a fixed level. It also allows information about signals *smaller than a single step* to remain in the output. Small changes in the input alter the probability of rounding to each neighbouring level, so the output statistics retain information about the signal. Under suitable listening conditions, a tone can remain audible below the noise floor. The cost is a noise floor about 4.8 dB above the undithered estimate in §3, for the triangular dither used here.

> **Figure 3 · audio + live — A tone below one quantisation step**
>
> A single sine at the selected level, quantised to the selected bit depth, shown as a spectrum. Undithered, a steady sine's error is periodic at any level and appears as harmonic peaks. Dithered, the error is noise and appears as a flat floor. Begin playback at a low volume and increase it gradually.
>
> Lower the tone level below one step and compare the two versions. The undithered tone develops artefacts and eventually rounds to silence, the third regime of §3. With dither, its level decreases continuously into the noise floor and remains audible below it.
>
> *(interactive figure — see the web page)*

> Dither also applies when reducing image precision. Quantising a gradient to a limited palette can produce visible bands. Adding noise before quantisation replaces these regular boundaries with a fine-grained pattern. Image formats such as GIF use dithering to represent intermediate colours with a limited palette.

Rectangular one-LSB dither decorrelates the error's *mean*. Its variance remains signal-dependent, so the noise level can vary with the input. Adding two independent rectangular sources gives a **triangular** distribution spanning ±1 LSB (TPDF), which decorrelates both mean and variance. The example above uses two random values for this reason. TPDF dither raises noise power by 4.77 dB relative to the uniformly distributed undithered quantisation-error model.

**Noise shaping** changes the distribution of quantisation noise across frequency. In audio, it can reduce noise where hearing is most sensitive, roughly 2–5 kHz, while increasing it at higher frequencies. The perceptual benefit depends on the filter and playback conditions. Noise shaping is used in 16-bit delivery and in delta-sigma converters.

---

---

*To be continued*

## The rest of the primer

This page is being published a part at a time. What is here covers sampled audio through dither. The sections below are written and will appear here as they are reviewed.

> **Still to come**
>
> 1. **Companding** — non-uniform quantisation, and how µ-law fits a wider range into the same number of bits.
> 2. **Reconstruction** — rebuilding a continuous signal from samples, and why the stair-step picture of digital audio is wrong.
> 3. **Video sampling** — the same theory applied to space and time rather than time alone.
> 4. **Pixel aspect ratio** — why pixels are not always square, and what that does to stored dimensions.
> 5. **Interlacing** — fields, field order, and the timing that comes with them.
> 6. **Gamma** — why stored brightness values are not proportional to light, and what breaks when that is ignored.
> 7. **Luma and chroma** — separating brightness from colour, and subsampling the colour channels.
> 8. **Pixel formats and containers** — fourccs, plane layouts, and what a container does and does not tell you.

In the meantime, [the sampling theorem page](../sampling/) is complete, and it covers the theory that the reconstruction section here depends on.


An interactive companion to Xiph.Org's [A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) by Christopher "Monty" Montgomery (Xiph.Org and Red Hat, 2010; wiki text CC-BY-SA). Figures run in the browser using generated test images, colourspace conversions, and synthesised audio passed through a quantiser.

Monty's follow-up, *Digital Show and Tell*, demonstrates several of these concepts using laboratory equipment.

Text adapted from Wikipedia (CC BY-SA 4.0) and Xiph.Org’s *A Digital Media Primer for Geeks* (CC BY-SA 3.0). This page is licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); see [LICENSE](https://github.com/ssemakov/digital-media-study/blob/main/LICENSE).
