<!-- Generated from index.html — the interactive figures live there. -->

*A digital media primer, for people who write software*

# Digital audio and video represent signals as *arrays of samples*.

A media file stores sampled audio and video together with information needed to interpret and play them. This page describes PCM, bit depth, dither, companding, signal reconstruction, pixel aspect ratio, interlacing, gamma, colour representation, pixel formats, and containers.

Xiph.Org's *A Digital Media Primer for Geeks* by Christopher "Monty" Montgomery (2010), adapted here with interactive figures.

The primer introduces the concepts and engineering constraints behind common audio and video formats.

The page contains twelve interactive figures. Images are generated procedurally, audio is synthesised at runtime, and colour-space calculations run in the browser. Additional derivations appear in **Go deeper** panels, which can be read independently of the main text.

---

*§1*

## Digital signals and repeatable copying

Digital communication predates recorded analogue audio. By the 1860s, telegraph systems were carrying multiplexed digital signals across continents. These systems represented messages using discrete states.

A major advantage of digital storage is **repeatable copying**. In an analogue chain, amplifiers, cables, and recording media add noise and distortion. These errors accumulate during transmission and copying, and separating them from the original signal is difficult.

A digital transmission uses discrete signal levels. A receiver can recover the intended value when noise stays within the decision threshold, allowing each stage to regenerate the data. A file copied without errors remains bit-identical through successive copies. Conversion between analogue and digital signals introduces quantisation and hardware errors; the sample rate also limits bandwidth. [The companion page on sampling](/sampling) explains the conditions for reconstructing a signal from its samples.

> **Data rates and hardware requirements**
>
> Raw HD video can require roughly 700 megabits per second, about 500× the data rate of CD audio. Real-time video processing became practical later than audio processing because it required substantially more computing power, storage, and data-transfer capacity.

---

*§2*

## PCM parameters and byte order

**Pulse Code Modulation** (PCM) represents a signal as measurements taken at fixed intervals. Reading a PCM stream requires its sample rate, sample format, channel count, and byte order.

**Sample rate** is the number of measurements per second. Half the rate is the upper frequency boundary for reconstruction. **Sample format** specifies the bit depth and representation: signed or unsigned integer, or floating point. It determines the available levels and *dynamic range*. **Channel count** specifies the number of simultaneous streams. Interleaved stereo stores left and right samples in alternating order. Values wider than 8 bits also have a **byte order**. Standard WAV PCM is little-endian; AIFF PCM is big-endian. Reading samples in the wrong byte order can produce loud noise.

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
> Sample rate sets the frequency range. Bit depth sets the quantisation step size and affects the noise floor. Adjust each slider to compare changes in the sampled waveform and data size.
>
> *(interactive figure — see the web page)*

### Common sample rates

| Rate | Use and background |
|---|---|
| **8 kHz** | Telephony. Speech is intelligible with about 4 kHz of bandwidth, so 8 kHz is the smallest rate that carries it. |
| **44.1 kHz** | CD audio. The rate comes from early digital recorders that stored audio on video tape, where each video field had to contain a whole number of samples. |
| **48 kHz** | Professional video and general audio production. Its relationship to common frame rates simplifies alignment between audio and picture. |
| **96 / 192 kHz** | Production formats. Higher rates allow wider transition bands for anti-aliasing and reconstruction filters and provide headroom for signal processing. |

> **Sample rate and aliasing**
>
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

> **Figure 2 · live — A signal and its quantisation error**
>
> Top: the signal and its quantised version. Bottom: the rounding error. Lower the signal level to see how the error pattern changes.
>
> At higher levels, the error can resemble random noise. At low levels, a simple signal crosses a small number of quantisation levels, producing a repeating error pattern correlated with the waveform. Section 4 describes how dither affects this pattern.
>
> *(interactive figure — see the web page)*

| Format | Range | Where it appears |
|---|---|---|
| **8-bit linear** | ~50 dB | Legacy audio and sound effects. Quantisation noise can be audible in quiet passages. |
| **8-bit µ-law / A-law** | ~14 bits' worth | Telephony. Non-uniform levels provide finer resolution for quiet signals — see §5. |
| **16-bit signed** | ~96 dB | CD audio and common delivery formats. Full scale is 0 dBFS; lower levels have negative dBFS values. |
| **24-bit signed** | ~144 dB | Recording and production. Additional range for low-level signals and processing. |
| **32-bit float** | very large | Mixing and mastering. ±1.0 corresponds to 0 dBFS. Floating-point storage can retain values beyond that level for later gain reduction. |

**Go deeper: Where 6.02 and 1.76 come from**

With b bits the step size is q=2/2b for a signal spanning −1 to +1. Assume the rounding error is uniformly distributed across ±q/2 as an approximation for signals that cross many levels. A uniform distribution of width q has variance q2/12, so the noise power is q2/12.

A full-scale sine has power 1/2. The ratio is q2/121/2=2q212=23⋅22b, and 10log10(2322b)=20blog102+10log101.5=6.02b+1.76 dB. The constant 1.76 equals 10log10(3/2), using the sine's RMS power as the signal reference.

This estimate assumes uniformly distributed error that is uncorrelated with the signal. Quiet or simple signals can produce repeating error patterns, limiting the estimate's usefulness for low-level behaviour. Appropriate dither removes the signal dependence of the error's mean and variance.

---

*§4*

## Dither: adding noise to reduce distortion

Dither is random noise added before quantisation to control the statistical properties of the rounding error.

Figure 2 shows how a quiet sine wave crosses the same few quantisation levels in each cycle. The resulting error repeats with the waveform and produces **harmonic distortion**: additional tones at multiples of the original frequency. These harmonics can be audible, particularly in simple or low-level signals.

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

Dither reduces signal-correlated distortion and introduces broadband noise. It also allows information about signals *smaller than a single step* to remain in the output. Small changes in the input alter the probability of rounding to each neighbouring level, so the output statistics retain information about the signal. Under suitable listening conditions, a tone can remain audible below the noise floor.

> **Figure 3 · audio + live — A tone below one quantisation step**
>
> A sine at the selected level, quantised to the selected bit depth. Harmonic peaks show distortion; the broadband floor shows noise. Begin playback at a low volume and increase it gradually.
>
> Lower the tone level below one step and compare the two versions. The undithered tone develops artefacts and eventually rounds to silence. With dither, its level decreases continuously into the noise floor.
>
> *(interactive figure — see the web page)*

> **Dither in images**
>
> Dither also applies when reducing image precision. Quantising a gradient to a limited palette can produce visible bands. Adding noise before quantisation replaces these regular boundaries with a fine-grained pattern. Image formats such as GIF use dithering to represent intermediate colours with a limited palette.

**Go deeper: Why triangular noise, and what noise shaping adds**

Rectangular one-LSB dither decorrelates the error's *mean*. Its variance remains signal-dependent, so the noise level can vary with the input. Adding two independent rectangular sources gives a **triangular** distribution spanning ±1 LSB (TPDF), which decorrelates both mean and variance. The example above uses two random values for this reason. TPDF dither raises noise power by 4.77 dB relative to the uniformly distributed undithered quantisation-error model.

**Noise shaping** changes the distribution of quantisation noise across frequency. In audio, it can reduce noise where hearing is most sensitive, roughly 2–5 kHz, while increasing it at higher frequencies. The perceptual benefit depends on the filter and playback conditions. Noise shaping is used in 16-bit delivery and in delta-sigma converters.

---

*§5*

## Companding and non-uniform quantisation

Linear 8-bit audio provides about 50 dB of range. Telephony uses 8-bit **companding** formats to represent a wider range of speech levels by varying the spacing between quantisation levels.

Perceived loudness is approximately logarithmic. Equal amplitude ratios correspond to similar changes in perceived level: both 0.001 to 0.002 and 0.5 to 1.0 are doublings. Quantisation with finer steps near silence can therefore reduce audible error in quiet signals.

**µ-law** (North America and Japan) and **A-law** (elsewhere) use approximately logarithmic spacing: fine steps near silence and coarse steps near full scale. Each sample still occupies one byte, with 256 possible values. The smallest steps provide low-level resolution comparable to roughly 13–14 bits of linear PCM.

> **Figure 4 · live — Even spacing versus logarithmic spacing**
>
> Left: the positions of quantisation levels. Right: the step size at each amplitude. Smaller steps reduce rounding error; the step size relative to the signal determines the signal-to-noise ratio.
>
> The "quality vs level" view compares relative step size. Linear coding has finer resolution for loud signals, while µ-law has finer resolution for quiet signals. "Equivalent bits" compares µ-law's *smallest* step with a linear quantiser's step. For the curve used here, the difference is about 5.5 bits, giving 8-bit µ-law low-level resolution comparable to 13–14-bit linear PCM.
>
> *(interactive figure — see the web page)*

Companding changes the mapping between sample values and amplitudes while keeping the sample count and byte count fixed. It is a form of **perceptual coding**: precision is allocated according to hearing sensitivity. Gamma encoding and chroma subsampling apply related principles to images.

---

*§6*

## Reconstructing a signal from samples

A sample records the signal's value at one instant. A plot can display samples as points, connect them with lines, or hold each value until the next sample. These are different interpolation rules applied to the same data.

For a signal bandlimited to below half the sample rate, ideal reconstruction produces the unique smooth curve consistent with the samples and that bandwidth limit. A digital-to-analogue converter approximates this reconstruction using interpolation and filtering.

> **Figure 5 · live — Three ways to draw the same samples**
>
> The same samples are shown with zero-order hold, linear interpolation, and ideal bandlimited reconstruction.
>
> Lower the rate to just above two samples per cycle. Ideal bandlimited reconstruction preserves the sine wave's frequency and amplitude. The staircase and straight-line plots introduce additional high-frequency components through their discontinuities or changes in slope.
>
> *(interactive figure — see the web page)*

> **The hold stage in a converter**
>
> A converter's hold stage can maintain each sample value until the next update, producing a staircase waveform internally. A reconstruction filter attenuates its high-frequency components. Oversampling moves these components farther above the audio band and simplifies filtering.

**Go deeper: Bandlimited reconstruction**

Given samples spaced T apart, the reconstruction is x(t)=∑nx(nT)sinc((t−nT)/T) with sinc(u)=sin(πu)/(πu). Under the sampling theorem's assumptions, this gives the unique signal consistent with every sample and bandlimited to below 1/2T. Other interpolation rules, including the staircase, introduce components above that bandwidth limit.

A staircase has discontinuities, whose spectra extend to arbitrarily high frequencies. Low-pass filtering suppresses these components as part of reconstruction.

---

*§7*

## Video sampling in space and time

Audio is sampled along the time axis. Video is sampled along time and the two spatial axes of the picture. The sampling theorem applies to each axis. Insufficient horizontal sampling can turn fine vertical stripes into moiré; insufficient temporal sampling can make a spinning wheel appear to rotate backwards.

Video requires substantially higher data rates than audio. Raw CD audio is about 1.4 megabits per second. Raw 1080i video can exceed **700 megabits per second**, roughly 500 times as much. Storage and transmission requirements motivated many of the video representations described in the following sections.

> **Figure 6 · calculator — Raw video data rate**
>
> The calculator multiplies width, height, frame rate, and stored bits per pixel to obtain the uncompressed data rate.
>
> At the same bit depth, 4:2:0 chroma subsampling halves the data rate relative to 4:4:4. Section 11 shows how reducing colour resolution affects the image.
>
> *(interactive figure — see the web page)*

---

*§8*

## Pixel aspect ratio

An image's displayed shape depends on its pixel dimensions and its **pixel aspect ratio**. Computer graphics commonly use square pixels. Several broadcast and disc-video formats use rectangular pixels.

Analogue television scanned in *lines*. The standard fixed the vertical line count, and each line carried a continuous horizontal signal. Digitisation chose a horizontal sample count based on that signal's bandwidth. The resulting samples can correspond to rectangular pixels, depending on the format.

For example, a 4:3 NTSC DVD can store **704×480** pixels with a pixel aspect ratio of **10:11**. Applying that ratio gives a displayed width of 640 at a height of 480. Displaying the stored grid with square pixels changes the image's proportions.

> **Figure 7 · live — Stored shape versus displayed shape**
>
> Toggle pixel aspect correction to compare the stored grid with the intended display proportions.
>
> An anamorphic 16:9 image can use the same 704×480 grid with a different declared aspect ratio. The player uses that information to display a widescreen picture. Incorrect aspect-ratio metadata stretches or compresses the displayed image.
>
> *(interactive figure — see the web page)*

---

*§9*

## Interlacing and field timing

Early television systems used interlacing to balance refresh rate and transmission bandwidth. Dividing the picture into alternating sets of scanlines allowed more frequent updates within the available bandwidth.

One pass carries the even-numbered scanlines and the next carries the odd-numbered lines. Each pass is a **field**. A system sending 60 fields per second carries the line count of 30 full frames per second while updating part of the picture every 1/60 second.

Successive fields are captured at **different moments**. Combining them into a frame places alternating lines from those moments in the same image. Objects that move between fields appear at different positions on adjacent lines, producing **combing**. Deinterlacing estimates a complete image at a chosen time from the available fields.

> **Figure 8 · live — Two moments in one frame**
>
> A shape moves from left to right and is captured in successive fields. Compare how the deinterlacing methods handle motion and vertical detail.
>
> Set the speed to zero to examine a stationary image, then increase it to see combing in moving regions. Motion-adaptive deinterlacing weaves stationary regions to preserve detail and interpolates regions where the fields differ.
>
> *(interactive figure — see the web page)*

> **Interlaced and progressive notation**
>
> **1080i** denotes 1080 lines with interlaced scanning. At 60 fields per second, each field contains 540 lines. **1080p** denotes 1080 lines with progressive scanning, where each update carries a complete frame. "480i60" denotes 480 lines at 60 interlaced fields per second, equivalent to 30 full frames' worth of lines per second. Interlaced material remains common in archives.

---

*§10*

## Gamma encoding and brightness

A cathode ray tube has a nonlinear brightness response, approximately proportional to the input voltage raised to the power of **2.5**. An encoding curve compensates for this response to produce the intended brightness.

Television systems applied the correction in the *camera*, reducing the correction circuitry needed in each receiver. A camera curve of roughly 1/2.2 combined with a CRT response near 2.5 produces an approximately linear round trip with a small contrast increase suited to dim-room viewing.

Gamma encoding also aligns with human brightness sensitivity. Vision distinguishes finer brightness differences in dark regions than in bright regions. A gamma curve allocates more stored levels to those dark regions, reducing visible quantisation at a given bit depth. This perceptual benefit remains useful with modern displays.

> **Figure 9 · live — Linear and gamma encoding at the same bit depth**
>
> The same 8-bit budget, allocated two ways. The difference appears at the dark end of each ramp.
>
> When the encode and decode exponents differ, the round-trip curve bends away from the diagonal and changes image brightness. sRGB uses a transfer curve close to a 2.2 power law, with a linear segment near black.
>
> *(interactive figure — see the web page)*

> **Arithmetic in linear light**
>
> Averaging gamma-encoded values can make scaling, blurring, and alpha-blending results too dark, especially around high-contrast detail. For these operations, decode the values to linear light, perform the arithmetic, and re-encode the result.

---

*§11*

## Luma, chroma, and colour resolution

Human vision uses three types of cone receptors. Displays combine red, green, and blue primaries in different proportions to reproduce colours within their gamut. RGB describes the contribution of each primary.

Human vision resolves finer detail in *brightness* than in *colour*. Video commonly uses Y′CbCr to represent these separately: one channel for **luma** (Y′), and two colour-difference channels for **chroma** (Cb and Cr):

**Y′=0.299R′+0.587G′+0.114B′,Cb=1.772B′−Y′,Cr=1.402R′−Y′**

- **R′,G′,B′** — gamma-encoded values from §10, indicated by the prime marks.

- **Y′** — luma: a weighted combination of the gamma-encoded channels, with the largest weight on green.

- **Cb,Cr** — blue-difference and red-difference chroma. Both are zero for a neutral grey.

Cb and Cr can be stored at *lower resolution* than Y′ with limited perceptual loss in many images. This is **chroma subsampling**. In 4:2:0, each chroma plane has half the width and half the height of the luma plane, reducing the total sample count by half relative to 4:4:4.

> **Figure 10 · live — Comparing luma and chroma subsampling**
>
> A generated test image is converted to Y′CbCr, subsampled, and converted back to RGB. Compare the effects of subsampling "the colour" and "the brightness".
>
> Select "the brightness" to compare the loss of fine detail. The same subsampling factor removes fewer samples from a single luma plane than from two chroma planes. Compare the visible changes with the measured error to see how spatial detail and colour affect perception.
>
> *(interactive figure — see the web page)*

### Chroma sample positions

Chroma siting specifies the position of each chroma sample relative to the luma grid. For 4:2:0, a chroma sample may be centred within a 2×2 luma block or aligned horizontally with a luma column, depending on the format.

MPEG-1, JPEG, Theora, and WebM use chroma centred horizontally and vertically. MPEG-2 uses vertical centring with horizontal alignment to every other luma column. PAL-DV alternates the chroma channels between lines. These layouts share the 4:2:0 sample ratio.

> **Figure 11 · live — Chroma siting layouts**
>
> Large dots show luma samples; rings show chroma samples. Compare the 4:2:0 layouts, with 4:2:2 included as a reference.
>
> Interpreting one siting layout as another can shift colour by half a pixel relative to luma. This is most visible at sharp colour boundaries, such as subtitles or a red logo on white. A conversion needs the source and destination siting information.
>
> *(interactive figure — see the web page)*

---

*§12*

## Pixel formats, fourccs, and containers

A pixel format specifies how samples are arranged in memory. **Packed** formats interleave channel values, such as Y, Cb, Y, Cr. **Planar** formats store each channel in a separate contiguous block, allowing the planes to be processed independently.

A **fourcc** is a four-character identifier for a format, such as `YV12`, `NV12`, `UYVY`, or `I420`. Pixel-format identifiers describe sample layout, subsampling, and bit depth. Chroma siting and colourspace matrices generally require separate metadata. For example, interpreting a `YV12` buffer also requires its siting convention and matrix, such as BT.601 or BT.709.

| Fourcc | Layout | What it is |
|---|---|---|
| **I420** | planar | 4:2:0 as Y, then Cb, then Cr. Common in software video processing. |
| **YV12** | planar | I420 layout with Cr and Cb swapped. Reading the planes in the wrong order changes the colours. |
| **NV12** | semi-planar | Y plane followed by an interleaved CbCrCbCr plane. Common in hardware decoding. |
| **UYVY / YUY2** | packed | 4:2:2 interleaved. Common in capture hardware and older editing pipelines. |
| **P010** | semi-planar | NV12-style layout with 10-bit samples. Common in HDR video pipelines. |

### Containers and stream metadata

Playback requires information about stream boundaries, frame sizes, and timing. Compressed frames can vary in size, and audio and video streams need timestamps to stay synchronised.

A **container**, such as MP4, Matroska, Ogg, AVI, or WebM, organises encoded streams into a file. It provides chunk boundaries, stream identification, timing, and metadata such as chapters and subtitles. Container formats support particular sets of codecs, and files using the same container can carry different encoded streams. The container and codec together determine how a player reads and decodes the media.

> **Figure 12 · live — Interleaving audio and video streams**
>
> Two streams share one file. Adjust interleaving to see how chunk placement affects the player's buffer requirement.
>
> Moving the interleaving slider right groups each stream into longer runs. A sequential reader must buffer more data to obtain matching audio and video. This increases startup delay and memory requirements during streaming; a local player can seek to the required chunks.
>
> *(interactive figure — see the web page)*

---

*§13*

## Summary

| Topic | Summary |
|---|---|
| **Digital copying** | Error-free digital copies preserve the sample values. Analogue transmission and copying accumulate noise and distortion. |
| **PCM** | Sample rate, sample format, channel count, and byte order specify how to read the samples. |
| **Sample rate** | Half the sample rate is the upper frequency boundary. Common rates include 44.1 kHz for CD audio and 48 kHz for video. |
| **Bit depth** | Each additional bit reduces the quantisation step size by half, lowering the noise floor by about 6 dB. |
| **Dither** | Noise added before rounding reduces signal-correlated distortion and preserves low-level information statistically. |
| **Companding** | Non-uniform levels give 8-bit samples finer resolution near silence. µ-law and A-law apply this to speech. |
| **Reconstruction** | Bandlimited interpolation recovers the continuous signal from samples. A hold stage requires reconstruction filtering. |
| **Video scale** | Raw HD video can require roughly 500× the data rate of CD audio, motivating reductions in storage and bandwidth. |
| **Pixel aspect** | Displayed shape depends on pixel dimensions and pixel aspect ratio. A 704×480 grid with 10:11 pixels displays as 4:3. |
| **Interlacing** | Successive fields record different times. Combining them produces combing where objects move. |
| **Gamma** | Nonlinear encoding allocates more levels to dark regions. Decode to linear light before brightness arithmetic. |
| **Chroma subsampling** | 4:2:0 halves the sample count relative to 4:4:4 by reducing chroma resolution while retaining luma detail. |
| **Chroma siting** | Formats with the same subsampling ratio can use different chroma positions. Conversions need the siting information. |
| **Containers** | Containers organise streams with framing, identification, timing, and metadata. Codec information specifies how to decode them. |

> **Perceptual allocation of precision**
>
> *Perceptual coding allocates precision according to human sensitivity.* µ-law varies audio level spacing, gamma varies brightness spacing, and chroma subsampling reduces colour resolution. These techniques illustrate a principle also used in modern audio and video codecs.

An interactive companion to Xiph.Org's [A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) by Christopher "Monty" Montgomery (Xiph.Org and Red Hat, 2010; wiki text CC-BY-SA). Figures run in the browser using generated test images, colourspace conversions, and synthesised audio passed through a quantiser.

Monty's follow-up, *Digital Show and Tell*, demonstrates several of these concepts using laboratory equipment.

Text adapted from Wikipedia (CC BY-SA 4.0) and Xiph.Org’s *A Digital Media Primer for Geeks* (CC BY-SA 3.0). This page is licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); see [LICENSE](https://github.com/ssemakov/digital-media-study/blob/main/LICENSE).
