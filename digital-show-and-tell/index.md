<!-- Generated from index.html — the interactive figures live there. -->

*Digital show and tell, for people who write software*

# Quantisation error can be made *independent of the signal*.

Draft. This page collects material from Xiph.Org's *Digital Show and Tell*, the follow-up to *A Digital Media Primer for Geeks*. It currently covers dither.

Xiph.Org's *Digital Show and Tell* by Christopher "Monty" Montgomery (2012), adapted here with interactive figures.

The page contains one interactive figure. Audio is synthesised at runtime in the browser. Additional derivations appear in **Go deeper** panels, which can be read independently of the main text.

---

*§1*

## Dither: adding noise to reduce distortion

Dither is random noise added before quantisation to control the statistical properties of the rounding error. Quantisation error is a second signal whose character depends on the input: noise for a loud or complex signal, distortion for a quiet or simple one, silence below half a step. Dither makes the character independent of the input. With dither, the error is noise in every case, at a fixed level set by the bit depth.

Without dither, a quiet sine crosses the same few quantisation levels in each cycle, so the error repeats with the waveform and produces **harmonic distortion**: additional tones at multiples of the original frequency. Even at full scale, a steady sine's error is periodic, a dense set of small harmonics. Enabling dither changes both cases: the added noise decides each rounding at random, the output flickers between neighbouring levels with probabilities that follow the input, and the error loses its relation to the waveform.

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

Dither replaces signal-correlated distortion with broadband noise at a fixed level. It also allows information about signals *smaller than a single step* to remain in the output. Small changes in the input alter the probability of rounding to each neighbouring level, so the output statistics retain information about the signal. Under suitable listening conditions, a tone can remain audible below the noise floor. The cost is a noise floor about 4.8 dB above the undithered quantisation-noise estimate, for the triangular dither used here.

> **Figure 1 · audio + live — A tone below one quantisation step**
>
> A single sine at the selected level, quantised to the selected bit depth, shown as a spectrum. Undithered, a steady sine's error is periodic at any level and appears as harmonic peaks. Dithered, the error is noise and appears as a flat floor. Begin playback at a low volume and increase it gradually.
>
> Lower the tone level below one step and compare the two versions. The undithered tone develops artefacts and eventually rounds to silence. With dither, its level decreases continuously into the noise floor and remains audible below it.
>
> *(interactive figure — see the web page)*

> Dither also applies when reducing image precision. Quantising a gradient to a limited palette can produce visible bands. Adding noise before quantisation replaces these regular boundaries with a fine-grained pattern. Image formats such as GIF use dithering to represent intermediate colours with a limited palette.

Rectangular one-LSB dither decorrelates the error's *mean*. Its variance remains signal-dependent, so the noise level can vary with the input. Adding two independent rectangular sources gives a **triangular** distribution spanning ±1 LSB (TPDF), which decorrelates both mean and variance. The example above uses two random values for this reason. TPDF dither raises noise power by 4.77 dB relative to the uniformly distributed undithered quantisation-error model.

**Noise shaping** changes the distribution of quantisation noise across frequency. In audio, it can reduce noise where hearing is most sensitive, roughly 2–5 kHz, while increasing it at higher frequencies. The perceptual benefit depends on the filter and playback conditions. Noise shaping is used in 16-bit delivery and in delta-sigma converters.

An interactive companion to Xiph.Org's *Digital Show and Tell* by Christopher "Monty" Montgomery (Xiph.Org and Red Hat, 2012). Figures run in the browser using synthesised audio passed through a quantiser.

Text adapted from Wikipedia (CC BY-SA 4.0) and Xiph.Org’s *A Digital Media Primer for Geeks* (CC BY-SA 3.0). This page is licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); see [LICENSE](https://github.com/ssemakov/digital-media-study/blob/main/LICENSE).
