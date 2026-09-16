<!-- Generated from index.html — the interactive figures live there. -->

*A digital media primer, for people who write software*

# Everything in a media file is *numbers on a grid*. The remainder is metadata.

Digital audio and video are often regarded as a specialist field. They are not. There is one fundamental idea — the sampling theorem — and then a long series of engineering decisions, most of which are historical accidents that became standards. This page covers the full stack: what a PCM sample is, why adding noise deliberately makes audio *better*, why video pixels are often not square, and why the position of a chroma sample is not standardised.

Xiph.Org's *A Digital Media Primer for Geeks* by Christopher "Monty" Montgomery (2010) — rebuilt as a set of interactive figures rather than a video.

Monty's framing is worth restating: this field appears difficult because the equipment used to be expensive, not because the concepts are harder than anything else in computer science.

Twelve live figures. Everything is computed in the browser — the images are generated procedurally, the audio is synthesised at runtime, and the colour-space calculations run on real pixels. Heavier material sits in **Go deeper** panels that can be skipped without loss of continuity.

---

*§1*

## Digital did not prevail because of higher accuracy

Begin with a fact that is contrary to common intuition: **digital came first.** The telegraph was carrying multiplexed digital signals across continents by the 1860s, decades before anyone recorded an analogue waveform. Analogue audio and video are the *newer* technology.

Digital also did not prevail on fidelity. A good analogue chain can be extremely accurate. It prevailed on a more mundane and more important property: **copies**. Every analogue component a signal passes through — every amplifier, every metre of cable, every tape generation — adds its own noise and distortion, permanently, and those errors accumulate. There is no way to remove them later, because nothing distinguishes the noise from the signal.

A digital signal has one property that changes everything: the value is quantised, so as long as the noise is smaller than half a step, you can round it away completely. That covers the amplitude axis; the time axis loses nothing either, for a reason that is the entire subject of [the companion page on sampling](/sampling). Copy a digital file a thousand times and the thousandth copy is bit-identical to the first. That is the entire argument. Conversion in and out is lossy in principle, but modern converters are so far below the threshold of human perception that in practice the ends of the chain stopped being the problem decades ago.

> **What actually made this an elite field**
>
> Not the mathematics. A single second of raw HD video is roughly 700 megabits — about 500× the data rate of CD audio. Computers could manipulate raw audio in real time about fifteen years before they could do the same for raw video, and the equipment required in the interim was prohibitively expensive. The concepts were always accessible; the hardware was not.

---

*§2*

## PCM is three numbers and a byte order

Raw digital audio — **Pulse Code Modulation** — is the simplest thing it could be: measure the signal at fixed intervals, write each measurement down as an integer. To read or write a PCM stream you need exactly three parameters, plus one additional detail.

**Sample rate** — how many measurements per second. This sets the highest frequency you can represent: half the rate, and not a hertz more. **Sample format** — how each measurement is stored: how many bits, signed or unsigned, integer or float. This sets the *dynamic range*. **Channel count** — how many simultaneous streams, stored interleaved: left, right, left, right, and so on. And the additional detail: anything wider than 8 bits has a **byte order**. WAV is little-endian, AIFF is big-endian, and getting it wrong produces a loud broadband noise.

That's the entire format. Multiply the three numbers together and you have the bitrate:

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

> **Figure 1 · live — The three numbers, and what they cost**
>
> The sliders show the waveform being sampled and rounded, and the resulting size in bytes.
>
> Note how differently the two sliders behave. Sample rate changes *which frequencies survive* — a horizontal limit. Bit depth changes *how quiet a sound can be before it disappears into the rounding* — a vertical limit. They are not interchangeable, and neither makes the waveform "steppier" in any audible way.
>
> *(interactive figure — see the web page)*

### The common rates, and why they exist

| Rate | Why it exists |
|---|---|
| **8 kHz** | Telephony. Speech is intelligible with about 4 kHz of bandwidth, so 8 kHz is the smallest rate that carries it. |
| **44.1 kHz** | CD. The oddly specific number comes from early digital recorders that stored audio on video tape — the rate had to fit a whole number of samples into a video field. It had no prior use and no successor. |
| **48 kHz** | The professional video standard, and the standard default. Divides neatly by common frame rates, which matters when audio must line up with picture. |
| **96 / 192 kHz** | Not for extending the audible range — human hearing does not extend above 20 kHz. They exist to give anti-aliasing and reconstruction filters a gentler roll-off, and to give processing headroom during production. |

> **One line on sample rate and aliasing**
>
> If frequencies above half the sample rate reach the converter, they are not discarded — they *fold down* into the audible range as new tones that were never in the source. The result is audible distortion. The fix is a filter before the converter, and it is the one step that cannot be postponed. ([The companion explainer on the sampling theorem](/sampling) covers this in detail; here it is taken as given, and the focus is on the other two numbers.)

---

*§3*

## Bit depth is dynamic range, not "accuracy"

This is the most common misunderstanding in digital audio. People assume more bits means each sample is "closer to the true value," so 16-bit audio is a slightly blurry version of the real thing and 24-bit is sharper. That is not what happens.

Rounding each sample to the nearest level introduces an error, and that error is bounded by half a level. The signal you can store gets no bigger — full scale is full scale. What changes is how far *below* full scale you can still hear something before it vanishes into the rounding. Bit depth sets the floor, not the ceiling. Each extra bit halves the step size, which lowers the noise floor by about 6 dB:

**dynamic range≈6.02b+1.76 dB**

- **b** — bits per sample.

- **6.02** — one extra bit doubles the number of levels; doubling is 6.02 dB.

- **1.76** — a correction because rounding error is spread evenly rather than sitting at its worst case.

So 8-bit gives you about 50 dB of usable range, 16-bit about 96 dB, and 24-bit about 144 dB — which is more range than any microphone or room can actually deliver, and comfortably past the point where the air itself is the limiting factor. That's why 24-bit is a *production* format: the headroom is there so that thirty stages of gain changes and mixing don't accumulate into anything audible, not because the final result needs it.

> **Figure 2 · live — What rounding actually does to a signal**
>
> Top: the signal and its quantised version. Bottom: the rounding error. Note how the error's *shape* changes as the signal gets quieter.
>
> At a healthy level the error appears as random noise — harmless. Turn the signal down and the error becomes *structured*, locked to the waveform. That structure is the real problem with quantisation, and it is what §4 addresses.
>
> *(interactive figure — see the web page)*

| Format | Range | Where it appears |
|---|---|---|
| **8-bit linear** | ~50 dB | Effectively extinct. Audibly noisy on anything with quiet passages. |
| **8-bit µ-law / A-law** | ~14 bits' worth | Telephony. Same byte count, far more usable range — see §5. |
| **16-bit signed** | ~96 dB | CD, and the delivery format for almost everything. Full scale is 0 dBFS; everything else is negative. |
| **24-bit signed** | ~144 dB | Recording and production. Headroom for processing, not for listening. |
| **32-bit float** | very large | Mixing and mastering. ±1.0 is 0 dBFS, but going over does not clip — the level can be reduced later without damage, which is the purpose. |

**Go deeper: Where 6.02 and 1.76 come from**

With b bits the step size is q=2/2b for a signal spanning −1 to +1. Assume the rounding error is uniformly distributed across ±q/2 — true when the signal is busy enough to wander across many levels. A uniform distribution of width q has variance q2/12, so the noise power is q2/12.

A full-scale sine has power 1/2. The ratio is q2/121/2=2q212=23⋅22b, and 10log10(2322b)=20blog102+10log101.5=6.02b+1.76 dB. The 1.76 is just 10log10(3/2) — the gain from measuring against a sine's RMS rather than its peak.

The assumption is worth examining. "Uniformly distributed error, uncorrelated with the signal" is exactly what stops being true for quiet or simple signals — which is why the formula describes the noise floor well in general and describes low-level behaviour badly. Dither is how you make the assumption true again.

---

*§4*

## Dither: adding noise to reduce distortion

This is the most counter-intuitive idea in digital audio.

The problem from Figure 2: when a signal is quiet, or simple, the rounding error stops being random. A quiet sine wave crosses the same few levels in the same pattern every cycle, so the error repeats at the signal's own frequency. Repeating error isn't noise — it's **distortion**, and distortion generates harmonics: new tones at multiples of the original that were never in the source. Your ear is extremely good at picking those out, because they're musically related to what you're listening to.

The fix is to add a small amount of random noise *before* rounding — about one level's worth:

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

	// naive: round to the nearest step; error is correlated, audible distortion
	out := math.Round(x*levels) / levels

	// dithered: error becomes uncorrelated noise, and the signal survives below one step
	d := (rand.Float64() + rand.Float64() - 1) / levels // triangular noise, plus or minus 1 LSB
	outDithered := math.Round((x+d)*levels) / levels

	fmt.Println(out, outDithered)
}
```

The noise decorrelates the error from the signal, which converts distortion into a constant, featureless hiss. And it does something better than that: because the noise pushes the signal across a rounding boundary more often when the signal is slightly higher, information about signals *smaller than a single step* survives in the statistics. A tone quieter than the smallest number the format can represent remains audible. It is below the noise, but the ear integrates over time and recovers it, which is why a conversation remains intelligible in a noisy room.

> **Figure 3 · audio + live — A tone quieter than one bit**
>
> A sine at the level shown, quantised to the depth shown. The spectrum shows the difference: spikes indicate invented harmonics; a flat floor indicates broadband noise. Raise the volume gradually before playing.
>
> Drag the tone level below one step and listen to the undithered version: the tone does not fade out smoothly; it breaks up into audible artefacts and eventually cuts to silence. The dithered version simply gets quieter, staying recognisable well past the point where the undithered one has broken down.
>
> *(interactive figure — see the web page)*

> **Why this applies beyond audio**
>
> The same trick applies anywhere you reduce precision. Reducing an image to 256 colours produces visible banding in gradients; adding a little noise before quantising converts the bands into a fine grain your eye ignores. That's what "dithering" means in a GIF export dialog, and it's the same mathematics.

**Go deeper: Why triangular noise, and what noise shaping adds**

Flat (rectangular, one-LSB) dither decorrelates the error's *mean* but leaves its variance modulated by the signal — you get a hiss that pumps slightly with the music. Adding two independent rectangular sources gives a **triangular** distribution spanning ±1 LSB (TPDF), which decorrelates both mean and variance. That's why the snippet above adds two calls to `random()`. The price is 4.77 dB more noise power than undithered quantisation — a trade that is almost always accepted.

**Noise shaping** goes further. The total noise power is fixed, but its distribution across frequency is not, so you can push it out of the band where hearing is most sensitive (roughly 2–5 kHz) and pile it up near 20 kHz where you can't hear it. The measured noise gets worse; the *perceived* noise drops by 10–15 dB. This is how 16-bit delivery achieves what sounds like considerably more than 96 dB of usable range, and it's the same principle a delta-sigma converter uses to get 20 bits out of a 1-bit comparator.

---

*§5*

## Companding: 8 bits that behave like 14

Linear 8-bit audio has poor quality — about 50 dB of range, which is not enough for speech with any variation in loudness. Yet telephony ran on 8 bits per sample for decades and sounded acceptable. The reason is that the levels do not have to be evenly spaced.

Human hearing is roughly logarithmic: a change in loudness is perceived as a *ratio*, not a difference. Going from 0.001 to 0.002 is as perceptually significant as going from 0.5 to 1.0. Even spacing wastes most of its levels on loud sounds where the differences are imperceptible.

So **µ-law** (North America and Japan) and **A-law** (elsewhere) space the levels logarithmically: fine steps near silence, coarse steps near full scale. Same 256 values, same one byte per sample, but the quiet end gets the resolution it needs. The result behaves like roughly 14 bits of linear range.

> **Figure 4 · live — Even spacing versus logarithmic spacing**
>
> Left: where the levels sit. Right: the step size at each amplitude — smaller is better, and what matters is the step size *relative* to the signal.
>
> Switch to "quality vs level" and the trade is clear: linear coding is better than µ-law for loud signals and substantially worse for quiet ones. µ-law gives up a little at the top — where the difference is imperceptible — to hold quality roughly constant all the way down. "Equivalent bits" compares µ-law's *smallest* step against a linear coder's: the ratio is a constant 5.5 bits, which is why 8-bit µ-law is usually quoted as behaving like 13–14 bits.
>
> *(interactive figure — see the web page)*

Note what this is *not*: compression in the file-size sense. There is no modelling, no prediction, no entropy coding — just a different mapping from numbers to amplitudes. It is the simplest form of perceptual coding, and the direct ancestor of the idea that every modern codec is built on: **spend bits where perception is sharp, save them where it is dull.** The video half of this page applies that same principle three more times.

---

*§6*

## Digital audio is not a staircase

Searches for "digital audio" return many diagrams showing a smooth analogue curve next to a blocky digital staircase, usually to argue that analogue is smoother and therefore better. The staircase is **incorrect** — not a simplification. No digital system produces it, and nothing in the theory suggests it.

The confusion comes from how samples are drawn. A sample is a single measurement at a single instant: a point, with no width. Drawing a flat step between two points is a drawing convention, not a claim about the signal. The question "what was the signal doing between these two points?" is not answered by "holding flat" — the answer is the unique smooth curve that passes through every point without varying faster than the rate allows, which is what a converter actually reconstructs.

> **Figure 5 · live — Three ways to draw the same samples**
>
> Same dots in all three. Only one of them is what comes out of a converter.
>
> Drop to barely more than two samples per cycle — where the staircase appears most exaggerated — and the real output is still a clean, smooth wave of exactly the right frequency and amplitude. The staircase's sharp corners would contain frequencies far above what the format can represent, which indicates that it cannot be the real signal.
>
> *(interactive figure — see the web page)*

> **Where the staircase does briefly exist**
>
> A low-cost converter does emit a staircase for a moment, and then a filter smooths it away — that is the "hold" stage, and its purpose is to be temporary. Modern converters oversample so heavily that the steps are far above the audible range before the filter even sees them. So the staircase exists for microseconds inside a chip, never in the stored data and never at the speaker.

**Go deeper: Why the smooth curve is the only possible answer**

Given samples spaced T apart, the reconstruction is x(t)=∑nx(nT)sinc((t−nT)/T) with sinc(u)=sin(πu)/(πu). This is not one option among many — it is the *only* signal that both passes through every sample and contains no frequency above 1/2T. Any other curve through those points, the staircase included, necessarily contains higher frequencies, and those frequencies could not have survived the sampling process in the first place.

The staircase specifically has discontinuities, and a discontinuity has energy at every frequency out to infinity. Reading a staircase as "what digital audio looks like" is therefore backwards: it's the one shape the format provably cannot contain.

---

*§7*

## Video is the same theorem, three times

Audio is sampled along one axis: time. Video is sampled along three — time, and the two spatial axes of the picture. Everything from the first half applies to each of them independently. Sample the horizontal axis too coarsely and fine vertical stripes alias into moiré. Sample time too coarsely and a spinning wheel appears to rotate backwards. Same theorem, different units.

What distinguishes video from audio is not the theory but the volume of data. Raw CD audio is about 1.4 megabits per second. Raw 1080i video is over **700 megabits per second**: roughly 500 times as much. That single ratio explains most of the history. Computers could edit raw audio in real time about fifteen years before they could do the same for video, and every design decision in the next few sections was made under bandwidth pressure that audio never experienced.

> **Figure 6 · calculator — What raw video actually costs**
>
> No compression — just width × height × frames × bits. The results are larger than most people expect.
>
> Chroma subsampling alone — covered in §11 — cuts this in half before any codec is involved, and costs almost nothing perceptible. It is the most efficient reduction in the pipeline.
>
> *(interactive figure — see the web page)*

---

*§8*

## Pixels are often not square

On a computer, a pixel is a square and an image's shape is just its pixel count. Broadcast video never worked that way, and the assumption still breaks things today.

Analogue television scanned in *lines*: the vertical resolution was fixed by the standard, but horizontally the picture was a continuous signal with no inherent pixel count at all. When it came time to digitise, the horizontal sample count was chosen from the channel's bandwidth — not to make the samples come out square. So a pixel ended up taller than it is wide, or wider than it is tall, depending on the standard.

The practical consequence: a 4:3 NTSC DVD stores **704×480** pixels, which is not a 4:3 ratio. Each stored pixel has a **10:11** shape, and the player stretches them on the way to the screen. Without that correction, the image appears slightly too tall.

> **Figure 7 · live — Stored shape versus displayed shape**
>
> The circle should be a circle. Toggling the correction shows the effect of omitting it.
>
> Anamorphic 16:9 is the extreme case: the same 704×480 grid is declared to represent a widescreen picture, so every pixel is stretched to about 1.46× its stored width. The file gives no indication of this beyond a flag — which is why a mislabelled flag makes the picture appear either stretched or squashed.
>
> *(interactive figure — see the web page)*

---

*§9*

## Interlacing: reducing bandwidth before compression existed

Early television engineers faced a dilemma. Low frame rates flicker badly and make motion stutter. High frame rates need bandwidth that was not available. Their solution was effective and has caused difficulties ever since.

Instead of sending whole frames, send **half** of each one: all the even-numbered scanlines in one pass, then all the odd-numbered lines in the next. Each half is a **field**. This provides the temporal smoothness of 60 updates per second at the bandwidth of 30 full frames, and because the eye integrates over the whole screen, the missing lines barely register.

This is the source of the difficulties. Two fields do *not* add up to one frame, because they were captured at **different moments**. Half the original content was never recorded at all. Weave two fields together and anything that moved between them shows up as a comb of interleaved stripes — not a compression artefact, but an accurate record of two different instants stored in one grid.

> **Figure 8 · live — Two moments in one frame**
>
> A shape moving left to right, captured as fields. Each deinterlacing strategy has a different cost.
>
> Set the speed to zero and every method looks perfect — combing only appears where something moved. That's the insight behind motion-adaptive deinterlacing: weave the still parts to keep full detail, and interpolate only where the fields disagree.
>
> *(interactive figure — see the web page)*

> **The naming convention explained**
>
> **1080i** is 1080 lines, interlaced — 60 fields per second, each holding 540 lines. **1080p** is 1080 lines, progressive — whole frames. "480i60" means 480 lines interlaced at 60 fields per second, which is 30 frames' worth of bandwidth. Interlacing is absent from new formats but remains common in archives, which is why deinterlacing code remains necessary.

---

*§10*

## Gamma: an accident that proved beneficial

A cathode ray tube is not a linear device. Double the voltage on the gun and you get much more than double the brightness — the response goes roughly as voltage to the power of **2.5**. Left uncorrected, every image would come out with crushed shadows and blown-out highlights.

Someone had to apply the inverse. The engineers made a decision that appeared to be a cost-saving measure and proved effective: put the correction in the *camera*. There would be a handful of cameras and millions of television sets, so correcting at the source was substantially cheaper. Cameras applied roughly a 1/2.2 power curve, the CRT applied its native 2.5, and the round trip came out close enough to linear. (The mismatch between 2.2 and 2.5 is not an error — it compensates for viewing in a dim room, where the eye requires slightly more contrast.)

It survived the obsolescence of the CRT for the following reason. Human brightness perception is *also* roughly a power law — an exponent near 3, not far from the CRT's 2.5. The eye can distinguish far more shades in dark regions than in bright ones. Storing values with a gamma curve therefore spends bits where the eye is most sensitive and saves them where it is least sensitive. It is, accidentally, a perceptual compression scheme — the same principle as µ-law from §5, arrived at for entirely unrelated reasons.

> **Figure 9 · live — Why 8 bits of linear light isn't enough, and 8 bits of gamma is**
>
> The same 8-bit budget, allocated two ways. The difference appears at the dark end of each ramp.
>
> When the encode and decode exponents differ, the round-trip curve bends away from the diagonal, producing a washed-out or crushed image; this is exactly what happens when software blends pixels without knowing which space they are in. Modern sRGB still uses a curve close to 2.2 for precisely the reason above; the CRT it was designed around is long gone.
>
> *(interactive figure — see the web page)*

> **A common bug this causes**
>
> Averaging two gamma-encoded pixels does *not* give the average brightness. Scaling an image, blurring it, or alpha-blending in gamma space makes results systematically too dark — most visibly when resizing images with fine high-contrast detail. Correct order: decode to linear, do the arithmetic, re-encode. A significant amount of graphics software still handles this incorrectly, which is why a downscaled image can appear darker than the original.

---

*§11*

## Colour: exploiting the limits of human vision

The eye has three colour receptors, so displays use three primaries — red, green, blue — and adding them in different proportions reproduces most visible colours. RGB is the natural format for a display, and a poor format for storage.

The reason: RGB spreads the picture's information evenly across three channels, but human vision does not treat those channels evenly. The eye resolves fine detail in *brightness* very well and fine detail in *colour* poorly. Video therefore converts to a different arrangement — one channel of brightness (**luma**, Y′), and two channels describing only how the colour departs from grey (**chroma**, Cb and Cr):

**Y′=0.299R′+0.587G′+0.114B′,Cb=1.772B′−Y′,Cr=1.402R′−Y′**

- **R′,G′,B′** — the gamma-encoded values from §10 — the prime marks matter.

- **Y′** — luma: the black-and-white picture. Green dominates because your eye is most sensitive to it.

- **Cb,Cr** — how much bluer, and how much redder, than grey. Zero means no colour at all.

Because colour detail is nearly invisible, Cb and Cr can be stored at *lower resolution* than Y′ with little perceptible loss. This is **chroma subsampling**: 4:2:0 halves both chroma dimensions, cutting the data in half before any codec runs.

> **Figure 10 · live — Discarding three-quarters of the colour**
>
> A procedurally generated test frame, put through a real Y′CbCr round trip. Compare "subsample the colour" against "subsample the brightness by the same amount."
>
> Switch to "the brightness" and the degradation becomes obvious — for a comparable measured error, and in fact a *smaller* data saving. That asymmetry is the entire justification for Y′CbCr, and it is why every camera, codec and streaming service ships 4:2:0 by default.
>
> *(interactive figure — see the web page)*

### Where, exactly, is a chroma sample?

This is the part that complicates interoperability. If four luma pixels share one chroma sample, *where does that chroma sample sit?* Centred among the four? Aligned with the left pair? Somewhere else?

Every answer got standardised by somebody. MPEG-1, JPEG, Theora and WebM centre it both horizontally and vertically. MPEG-2 centres it vertically but aligns it horizontally with every other luma column. PAL-DV does something different again, alternating which chroma channel each line carries. All of them are labelled "4:2:0."

> **Figure 11 · live — Four standards, one label**
>
> Large dots are luma samples; rings are chroma. Every layout below is legitimately called 4:2:0 by some standard.
>
> If the siting is wrong, colour shifts by half a pixel against the brightness — subtle on most frames, and clearly visible on hard colour edges like subtitles or a red logo on white. This is a leading cause of transcodes that look slightly off, and it is invisible in the file's metadata.
>
> *(interactive figure — see the web page)*

---

*§12*

## Pixel formats, fourccs, and the box it all ships in

Between "I have Y′CbCr samples" and "I have bytes in a buffer" there is one more decision: layout. **Packed** formats interleave the channels — Y, Cb, Y, Cr, and onward. **Planar** formats keep each channel in its own contiguous block, which is what almost every codec wants, since it can then process the small chroma planes independently.

Multiply that choice by every subsampling scheme and every bit depth and the result is the **fourcc** set: four-character codes like `YV12`, `NV12`, `UYVY`, `I420`. At least fifty exist; roughly fifteen are common. The complication: a fourcc describes the *arrangement* of samples, and generally says nothing about chroma siting or which colourspace matrix was used. `YV12` alone doesn't tell you whether the chroma is sited MPEG-1 style or MPEG-2 style, or whether the matrix was BT.601 or BT.709. That information travels separately, or it doesn't travel at all.

| Fourcc | Layout | What it is |
|---|---|---|
| **I420** | planar | 4:2:0 as Y, then Cb, then Cr. The default format of video codecs. |
| **YV12** | planar | Identical to I420 with Cr and Cb swapped. Confusing the two produces an incorrectly orange-and-blue picture. |
| **NV12** | semi-planar | Y plane, then a single interleaved CbCrCbCr plane. What most hardware decoders emit. |
| **UYVY / YUY2** | packed | 4:2:2 interleaved. Common in capture hardware and older editing pipelines. |
| **P010** | semi-planar | Like NV12 but 10 bits per sample. The usual HDR working format. |

### Containers: the part unrelated to pixels

A raw stream of compressed frames is unusable on its own. The frames vary in size unpredictably, so frame 400 cannot be located by multiplication. There is no way to tell where audio ends and video begins. There are no timestamps, so nothing tells the player how to keep them together.

A **container** — MP4, Matroska, Ogg, AVI, WebM — solves exactly those problems and nothing else. It adds framing so each chunk's boundaries are findable, identification so streams can be told apart, timing so they can be synchronised, and space for metadata like chapters and subtitles. The crucial property is that containers are *generic*: the container is independent of the codec that produced the bytes it carries. That is why "MP4" is not a video format, and why two MP4 files can be completely different inside.

> **Figure 12 · live — Why streams are split and interleaved**
>
> Two streams, one file, one read head. The buffer slider shows why interleaving is necessary.
>
> As the interleaving slider moves right, the streams separate into long runs. Playback still works, but the player must then buffer everything between the audio it is playing and the video it needs — which is why a badly interleaved file stutters when streamed and plays correctly from a local disk.
>
> *(interactive figure — see the web page)*

---

*§13*

## The whole primer on one page

| Thing | What to remember |
|---|---|
| **Why digital won** | Not accuracy — *copyability*. Analogue errors accumulate forever; digital errors round away to nothing. |
| **PCM** | Sample rate, sample format, channel count, byte order. That's the entire format. |
| **Sample rate** | Sets the highest frequency: half the rate. 44.1 kHz is a historical accident; 48 kHz is the standard default. |
| **Bit depth** | Sets the noise floor, not the "accuracy". About 6 dB per bit. 16-bit is for delivery, 24-bit for production. |
| **Dither** | Add ~1 step of noise before rounding. Converts audible distortion into inaudible hiss, and preserves signals smaller than one step. |
| **Companding** | Log-spaced levels give 8 bits the usable range of ~14. The direct ancestor of all perceptual coding. |
| **The staircase** | Doesn't exist. It's a drawing convention, and the one shape the format provably cannot contain. |
| **Video scale** | ~500× the data rate of CD audio. Every video design decision was made under that pressure. |
| **Pixel aspect** | Stored size ≠ displayed shape. 704×480 with a 10:11 pixel is a 4:3 picture. |
| **Interlacing** | Two fields, two *moments*. Combing is not an artefact, it is an accurate record of motion. |
| **Gamma** | A CRT workaround that survived because human vision has the same shape. Determine which space pixels are in before performing arithmetic. |
| **Chroma subsampling** | Half the data, invisible cost — because the eye resolves brightness far better than colour. |
| **Chroma siting** | Four incompatible layouts all called 4:2:0, and the file usually doesn't say which. |
| **Containers** | Framing, identification, timing, metadata. Generic by design — "MP4" tells you almost nothing about the contents. |

> **The principle running through all of it**
>
> Every effective decision on this page is the same one: *identify where human perception is sharp, allocate bits there, and remove them elsewhere.* µ-law does it with loudness, gamma does it with brightness, chroma subsampling does it with colour. Modern codecs apply the same idea more aggressively; once the pattern is visible, their behaviour is straightforward to understand.

An interactive companion to Xiph.Org's [A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) by Christopher "Monty" Montgomery (Xiph.Org and Red Hat, 2010; wiki text CC-BY-SA). Every figure is computed live in the browser — the test images are generated procedurally, the colourspace conversions run on real pixels, and the audio is synthesised at runtime through an actual quantiser.

Monty's follow-up, *Digital Show and Tell*, demonstrates several of these points on real lab equipment.

Text adapted from Wikipedia (CC BY-SA 4.0) and Xiph.Org’s *A Digital Media Primer for Geeks* (CC BY-SA 3.0). This page is licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); see [LICENSE](https://github.com/ssemakov/digital-media-study/blob/main/LICENSE).
