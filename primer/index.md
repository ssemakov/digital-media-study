<!-- Generated from index.html — the interactive figures live there. -->

*A digital media primer, for people who write software*

# Digital audio and video represent signals as *arrays of samples*.

A media file stores sampled audio and video together with information needed to interpret and play them. This first part describes PCM parameters, byte order, bit depth and quantisation error. Dither, companding, reconstruction and the whole video half — pixel aspect ratio, interlacing, gamma, colour representation, pixel formats and containers — are listed at the end and will follow.

Xiph.Org's *A Digital Media Primer for Geeks* by Christopher "Monty" Montgomery (2010), adapted here with interactive figures.

The primer introduces the concepts and engineering constraints behind common audio and video formats.

This part contains two interactive figures. Images are generated procedurally, audio is synthesised at runtime, and colour-space calculations run in the browser. Additional derivations appear in **Go deeper** panels, which can be read independently of the main text.

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

Rounding a sample to the nearest level introduces an error bounded by half a step. Each additional bit halves the step size. When the rounding error behaves as uniformly distributed noise, this lowers the noise floor by about 6 dB per bit:

**dynamic range≈6.02b+1.76 dB**

- **b** — bits per sample.

- **6.02** — one extra bit doubles the number of levels; doubling is 6.02 dB.

- **1.76** — a constant arising from the power ratio between a full-scale sine and uniformly distributed rounding error.

The approximate range is 50 dB for 8-bit audio, 96 dB for 16-bit audio, and 144 dB for 24-bit audio. Recording and production commonly use 24-bit samples to accommodate low recording levels and repeated processing. The usable range of a recording also depends on microphone noise, converter performance, and the recording environment.

> Signal levels on this page are given in **dBFS**, decibels relative to full scale. Full scale is the largest value the sample format can hold, ±1.0 in the figures, and a signal whose peaks reach it is at 0 dBFS. Quieter signals have negative values: the level is 20·log10(amplitude ÷ full scale), so −6 dBFS is half the amplitude, −20 dBFS one tenth, and −60 dBFS one thousandth. The unit makes levels and bit depth directly comparable. Each bit adds about 6 dB, so the quantisation noise of a 16-bit format sits near −96 dBFS, and a tone at −90 dBFS is about 6 dB above it.

> **Figure 2 · live — A signal and its quantisation error**
>
> A sine wave is rounded to the nearest of the 2bits levels. Top: the original and the rounded version, with the levels drawn as dashed lines; the vertical axis zooms in as the signal gets quieter. Bottom: the rounding error, rounded minus original, measured in quantisation steps. Rounding is never off by more than half a step, so the error's size is fixed and its *shape* is what changes. Lower the signal level and watch the shape.
>
> At 0 dBFS with 4 bits the sine spans 8 steps and crosses all 16 levels. The error is a rapid sequence of small ramps, one per level crossing, and resembles random noise. This is the regime the 6 dB-per-bit estimate describes. Near −20 dBFS the sine spans less than one step, the rounded output becomes a two- or three-level square wave, and the error becomes a periodic waveform locked to the signal. That error is distortion: harmonics of the signal that were not in the source. Below about −24 dBFS the amplitude is under half a step, every sample rounds to zero, and the error is the negative of the signal. "Levels used" counts how many levels the rounded output lands on. With dither enabled, random noise is added before rounding, the output flickers between neighbouring levels in proportion to the input, and the error loses its lock to the waveform. Section 4 describes this in detail.
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

*To be continued*

## The rest of the primer

This page is being published a part at a time. What is here covers sampled audio up to the point where quantisation error is understood. The sections below are written and will appear here as they are reviewed.

> 1. **Dither** — adding noise on purpose so that quantisation error becomes noise rather than distortion, and why a signal quieter than one step survives it.
> 
> 2. **Companding** — non-uniform quantisation, and how µ-law fits a wider range into the same number of bits.
> 
> 3. **Reconstruction** — rebuilding a continuous signal from samples, and why the stair-step picture of digital audio is wrong.
> 
> 4. **Video sampling** — the same theory applied to space and time rather than time alone.
> 
> 5. **Pixel aspect ratio** — why pixels are not always square, and what that does to stored dimensions.
> 
> 6. **Interlacing** — fields, field order, and the timing that comes with them.
> 
> 7. **Gamma** — why stored brightness values are not proportional to light, and what breaks when that is ignored.
> 
> 8. **Luma and chroma** — separating brightness from colour, and subsampling the colour channels.
> 
> 9. **Pixel formats and containers** — fourccs, plane layouts, and what a container does and does not tell you.

In the meantime, [the sampling theorem page](../sampling/) is complete, and it covers the theory that the reconstruction section here depends on.

An interactive companion to Xiph.Org's [A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) by Christopher "Monty" Montgomery (Xiph.Org and Red Hat, 2010; wiki text CC-BY-SA). Figures run in the browser using generated test images, colourspace conversions, and synthesised audio passed through a quantiser.

Monty's follow-up, *Digital Show and Tell*, demonstrates several of these concepts using laboratory equipment.

Text adapted from Wikipedia (CC BY-SA 4.0) and Xiph.Org’s *A Digital Media Primer for Geeks* (CC BY-SA 3.0). This page is licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); see [LICENSE](https://github.com/ssemakov/digital-media-study/blob/main/LICENSE).
