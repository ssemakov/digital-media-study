<!-- Generated from index.html — the interactive figures live there. -->

*Sampling, for people who write software*

# Sampling at twice the highest frequency is *sufficient* to reconstruct a signal.

Measurements taken at a fixed interval produce an array of numbers. The sampling theorem answers a specific question: **when is that array a complete record, and when has it discarded information?** The answer is exact, not approximate, and the failure mode is named *aliasing*.

If no component of the signal oscillates faster than B times per second, then measuring it more than 2B times per second captures it *perfectly* — the original can be rebuilt from the measurements with zero error.

Claude Shannon, 1949. "Perfectly" is literal here: the reconstruction is exact, and the formula that performs it is built in [the section on rebuilding the original](#recon).

This page contains twelve live figures; each claim in the text has an associated slider that can be adjusted. Every equation is explained symbol by symbol directly beneath it, and the more advanced material is placed in **Go deeper** panels that can be skipped without loss of continuity.

> fs is the **sample rate** — measurements per second (44100 for CD audio, 60 for a 60 fps game loop, 1 for a metric scraped every second). T=1/fs is the gap between measurements. B is the **bandwidth**: the highest frequency present in the measured signal. That is the complete set of terms.

---

*§1*

## Infinitely many curves fit your data

Begin with the core definition. Sampling is the following loop, and nothing more:

```go
package main

import (
	"fmt"
	"math"
)

// this is all "sampling" means
func main() {
	fs := 4.0                     // sampling frequency: measurements per second, in hertz
	N := 8                        // how many samples to take
	T := 1 / fs                   // sampling period: seconds between measurements
	samples := make([]float64, N) // the recording: N numbers, nothing else
	for n := 0; n < N; n++ {
		samples[n] = measure(float64(n) * T) // read the signal at t = n*T
	}
	fmt.Println(samples)
}

// a 1 Hz tone, standing in for whatever is being measured
func measure(t float64) float64 {
	return math.Sin(2 * math.Pi * t) // one full cycle per second
}
```

The array `samples` now holds the data, and the continuous signal that was measured is gone. The difficulty: **infinitely many different signals produce that exact array.** Plotting the samples as points on a graph, many curves can be drawn through them. The data alone does not determine which one is correct.

The theorem resolves this with an additional constraint: "bandlimited". A signal is bandlimited to B means it contains no oscillation faster than B cycles per second — no sudden jumps, no infinitely sharp corners, and no variation that passes between measurements and returns before the next one. With that constraint, out of the infinitely many curves through the points, **exactly one** is smooth enough to qualify. The array plus the condition "nothing oscillates faster than B" determines the signal uniquely.

The intuition underlies everything that follows. A slow-moving signal cannot change much between measurements. If the sampling rate is high enough relative to how fast the signal can change, the signal is fully determined by the samples.

> One cycle of a wave has a peak and a trough. To detect a wave at all requires at least one measurement while it rises and one while it falls — two per cycle. With fewer, a fast wave becomes indistinguishable from a slow one, which is what Figure 2 shows. That is the 2B; the remainder of this page follows from that observation.

The set of signals bandlimited to B with finite energy is called the Paley–Wiener space PWB. Its defining feature: a signal in it is the inverse Fourier transform of something supported on a finite interval, which by the Paley–Wiener theorem makes it an *entire function* of exponential type 2πB — analytic on the whole complex plane.

Analytic functions are rigid. They cannot vary locally without that variation being visible in their global behaviour, which is the formal version of the earlier intuition. More importantly, PWB is a *reproducing-kernel Hilbert space*: evaluating a signal at a point is a well-behaved (bounded) operation, which is not true for arbitrary square-integrable functions, where "the value at a single instant" is meaningless. The kernel that performs the evaluation is a sinc, which is why the same function turns up again as the reconstruction weights in [the section on rebuilding the original](#recon) — one function in two roles.

---

*§2*

## Sampling makes copies of the spectrum

To determine when sampling loses information, consider the signal's **[spectrum](../waves/#spectrum)** rather than its waveform — the amount of energy at each frequency. This is the same representation produced by an audio visualiser or an FFT.

The central fact of the theory is the following. Sampling in time *duplicates* the spectrum in frequency. Not approximately: it makes exact copies, spaced exactly fs apart, and adds them all together:

**Xs(f)=T1k=−∞∑∞X(f−kfs)**

- **X(f)** — the spectrum of the original signal — what was actually present.

- **Xs(f)** — the spectrum the samples represent.

- **∑k** — sum over every integer k, including negatives.

- **X(f−kfs)** — the original spectrum, shifted by k sample rates.

- **T** — the sampling period: the time between one measurement and the next, equal to 1 divided by the sample rate.

In words: **the samples do not contain X. They contain X plus every copy of X shifted by a multiple of the sample rate.** If those copies do not overlap, the middle one can be cropped out and the original recovered exactly. If they overlap, they have been *summed*, and a sum of two numbers cannot be separated.

That gives the condition immediately. The original spans −B to B. Its neighbour is shifted to fs−B up to fs+B. They do not overlap when B<fs−B, which rearranges to fs>2B. That is the theorem in full. Adjust the sliders below until the copies overlap.

> **Figure 1 · live — The samples represent the spectrum, plus every shifted copy of it**
>
> The blue region is the original spectrum. The orange regions are the copies added by sampling. Raise the bandwidth or lower the sample rate until they overlap.
>
> The dashed yellow box is the ideal crop: keep everything below half the sample rate and discard the rest. It works exactly until the copies overlap, after which it cannot recover the original, because the region being cropped is already a sum.
>
> *(interactive figure — see the web page)*

Sampling *faster* than the minimum produces a gap. At exactly fs=2B the copies are touching, so the crop must be infinitely sharp — a filter that passes everything below the line and nothing above it, with no transition. Such a filter is not realisable in hardware and is expensive in software. Sampling at, for example, 4× opens an adequate gap for a practical filter to roll off through. **Oversampling does not improve the theorem; it makes the engineering feasible.** This is why 44.1 kHz audio was considered tight and 96 kHz production rates exist.

Model ideal sampling as multiplying the signal by a Dirac comb — an infinite train of impulses spaced T apart. The Fourier transform of a comb with spacing T is another comb with spacing 1/T=fs. Multiplication in the time domain is convolution in the frequency domain, and convolving anything with a comb of impulses produces a copy of it at every tooth. Hence the shifted copies, spaced fs apart, scaled by 1/T.

The identity has a name — the *Poisson summation formula* — and its other side is worth knowing: that same sum equals ∑nx(nT)e−i2πfnT, which is the discrete-time Fourier transform of the sample sequence. So the left side is "the true spectrum, periodised" and the right side is "everything a computer can compute from the samples." The theorem is the observation that periodising is reversible exactly when the copies don't overlap.

---

*§3*

## Aliasing: high frequencies appearing as low frequencies

When the copies overlap, a high frequency coincides with a low one and becomes *indistinguishable* from it. It is not attenuated or blurred; it is relabelled. A 3.2 kHz tone sampled at 4 kHz does not return as a quiet 3.2 kHz tone; it returns as an ordinary 800 Hz tone that was never present.

The arithmetic is short enough to write as a function:

```go
package main

import "fmt"

// aliasOf reports where a tone at f hertz appears after sampling at fs hertz.
func aliasOf(f, fs int) int {
	r := ((f % fs) + fs) % fs // wrap into [0, fs)
	if r <= fs/2 {
		return r // below half the sample rate: representable as-is
	}
	return fs - r // above it: mirrored back down into the band
}

func main() {
	fmt.Println(aliasOf(3200, 4000))  // -> 800
	fmt.Println(aliasOf(4800, 4000))  // -> 800  (a different tone, same result)
	fmt.Println(aliasOf(11200, 4000)) // -> 800  (and another)
}
```

Texture addressing provides a useful analogy: frequency is not clamped at the limit and does not wrap around; it uses **mirrored repeat**. Past the edge, the value reflects back. This reflection is why the mapping is a triangle wave rather than a sawtooth, and why three different input tones above can all map to 800 Hz.

Written as math, that function is:

**falias=f0−fs⋅round(f0/fs)**  
*the same thing, in symbols*

That reflection gives the frequency axis a repeating structure worth naming, because the rest of this page uses it. Sampling at a rate divides the whole axis into consecutive slices, each one half the sample rate wide: 0 to half, half to one whole rate, one whole rate to one and a half, and so on upwards without end. These slices are the **zones**, numbered from 1. Zone 1 has a name of its own: the **baseband**. It runs from 0 to half the sample rate, and it is the only stretch of the frequency axis that a set of samples can express at all. Every other zone is folded down onto it, which is why a measurement can only ever report a frequency in that range.

Folding maps each zone onto the baseband in turn. Odd-numbered zones arrive **upright**, the right way up. Even-numbered zones arrive **mirrored**, low frequencies and high frequencies swapped, which is the phase flip described below. Two frequencies in the same zone stay distinct after folding; two in different zones can land on the same place and become impossible to tell apart.

Two frequencies are aliases of each other exactly when their sum or difference is a multiple of the sample rate. On the reflected segments there is an additional effect: the wave returns **phase-flipped**, so a rising sweep is heard as a falling one. This is why an undersampled radio band arrives with its spectrum mirrored, and it can be heard directly in [the audible demonstration](#hear).

> **Figure 2 · live — Two different signals, the exact same measurements**
>
> Increase f0 past half the sample rate. The samples can no longer distinguish the two signals, and the orange curve is the one the system will reproduce.
>
> Enable "staircase output" to see the behaviour of low-cost playback: each value is held until the next arrives. The staircase follows the *orange* curve. No downstream processing can undo an alias; the incorrect result was fixed at the moment of measurement.
>
> *(interactive figure — see the web page)*

> **Figure 3 · live — Every input frequency, and where it ends up**
>
> Input frequency along the bottom, output frequency up the side. Everything above half the sample rate reflects back down. Hover anywhere to trace a value.
>
> This is `aliasOf` plotted. The reflection points sit at multiples of fs/2; the shaded zones are those that return phase-flipped. Deliberately selecting which zone the signal lands in is an established technique, covered in [undersampling on purpose](#bandpass).
>
> *(interactive figure — see the web page)*

> Aliasing is not noise that can be filtered out later. It is *valid-looking data at the wrong frequency*, permanently mixed into the real data. Every remedy must occur before or during measurement, never after.

---

*§4*

## Rebuilding the original: an interpolation kernel

The samples contain all the information. To recover the signal: crop the spectrum to the middle copy and transform back to the time domain. The crop, viewed in the time domain, is a specific weighting function. The result:

**x(t)=n∑x(nT)⋅sinc(Tt−nT),sinc(u)=πusinπu**

The sum is an interpolation loop, of the same form as interpolation code written with a different kernel:

```go
package main

import (
	"fmt"
	"math"
)

// sinc is the ideal interpolation kernel: 1 at its centre, 0 at every other sample instant.
func sinc(u float64) float64 {
	if u == 0 {
		return 1 // the centre of the kernel
	}
	return math.Sin(math.Pi*u) / (math.Pi * u)
}

// the value at ANY time t, exactly, from the samples alone
func valueAt(t float64, samples []float64, T float64) float64 {
	sum := 0.0
	for n := 0; n < len(samples); n++ {
		sum += samples[n] * sinc((t-float64(n)*T)/T) // each sample, weighted by its distance from t
	}
	return sum
}

func main() {
	T := 1.0                                       // sampling period: seconds between samples
	samples := []float64{0, 1, 0, -1, 0, 1, 0, -1} // a sampled sine, four samples per cycle
	fmt.Println(valueAt(2.5, samples, T))          // reconstruct halfway between two samples
}
```

Replacing `sinc` with a box gives nearest-neighbour interpolation. Replacing it with a triangle gives linear interpolation. Replacing it with a cubic gives bicubic. **Sinc is the kernel that is exactly correct**; the others are approximations to it. Truncating the loop to a few neighbours and tapering the ends gives **Lanczos**, the resampling filter used in most image libraries.

Two properties make it work: sinc(0)=1, and sinc is exactly zero at every other integer. Each sample's kernel contributes its full value at its own instant and *zero* at every other sample instant. The curve passes through every sample point exactly, and the kernels determine only the values in between.

> **Figure 4 · live — The waveform built from kernels**
>
> Each measurement contributes one scaled sinc. Summing enough of them reconstructs the original. Enable "show kernels" to see the individual contributions.
>
> Reducing the kernel count makes the error appear at the *edges* first; the middle remains accurate longest. This is why practical resamplers use a few dozen taps around each output point rather than the full infinite sum, and why they remain accurate.
>
> *(interactive figure — see the web page)*

> The ideal kernel is infinitely wide and requires samples from the future, so it is not implemented directly. The theorem guarantees that **no information was destroyed**, so the difference between a finite, practical interpolator and the ideal one is a controllable error budget rather than an unrecoverable loss. That distinction is the key point.

Set φn(t)=T−1/2sinc((t−nT)/T). These functions are *orthonormal*: ∫φnφmdt=δnm, because the sincs are mutually zero at each other's sample points and their overlaps cancel exactly. They also *span* the whole space of signals bandlimited to 1/2T. Together that makes {φn} an orthonormal basis, and the reconstruction formula is nothing but the standard "expand a vector in an orthonormal basis" — the samples *are* the coordinates.

One consequence: Parseval's identity gives ∫∣x(t)∣2dt=T∑n∣x(nT)∣2. The energy in the samples equals the energy in the waveform, exactly. Sampling is not a lossy summary that is merely good enough; it is a rotation into a different coordinate system, and rotations preserve energy. Under undersampling, this identity is the first to break.

---

*§5*

## An off-by-one error in the textbook statement

Many references state the rule as fs≥2B. It should be fs>2B, strictly. The counterexample is one line.

Sample a sine wave at exactly twice its frequency — two measurements per cycle, the theoretical minimum. If the measurements land on the zero crossings, every one reads **zero**. The array is all zeros. The signal is indistinguishable from silence, and from the same sine at ten times the amplitude.

**x(t)=sin(2πBt),x(nT)=sin(2πB⋅n/2B)=sin(πn)=0 for all n**

Shifting the phase recovers the amplitude partially. Adjust the slider below: at one phase the full amplitude is captured, at another nothing is captured, and in between an arbitrary fraction is captured. A wave at exactly half the sample rate has its amplitude determined by *where the measurements fall*, which is not a property of the signal.

> **Figure 5 · live — Exactly two samples per cycle, as the phase rotates**
>
> The blue wave has constant amplitude. Observe the samples and the reconstructed output.
>
> Start the rotation. The signal does not change; only the alignment between the wave and the measurement clock changes. Depending on that alignment is not acceptable, which is why the rule is a strict inequality.
>
> *(interactive figure — see the web page)*

In practice this rarely occurs, because a real filter has already attenuated anything at the limit. It is worth knowing for the same reason as whether a bounds check uses `<` or `<=`: it indicates that the boundary is genuinely excluded, not merely marginal.

> The **Nyquist rate** is 2B — a property of your *signal*, the rate you'd need. The **Nyquist frequency** is fs/2 — a property of your *sampler*, the highest frequency it can represent. They're different numbers that happen to be equal at the minimum rate, which is exactly why people mix them up. If someone says "sample above the Nyquist frequency," they mean the rate.

A real sinusoid has two degrees of freedom — amplitude and phase, or equivalently an in-phase and a quadrature component. At exactly fs/2 the frequency folds onto *itself*, and because folding conjugates phase, the quadrature component cancels against its own image. The sampler measures Acosϕ and is structurally blind to Asinϕ: one degree of freedom out of two, gone.

With more machinery: the sampling operator restricted to PWfs/2 has a one-dimensional kernel spanned by sin(2π(fs/2)t). Any signal can absorb an arbitrary multiple of that function without a single sample changing, so the map isn't injective and there's no hope of inverting it. Demanding B<fs/2 — no energy *at* the boundary — is exactly the condition that kills the kernel.

---

*§6*

## An audible demonstration

Aliasing is often clearer through a speaker than on a screen. Below, a tone sweeps upward from 200 Hz to 7 kHz. It is decimated to the selected rate and reconstructed correctly; the only variable is whether a filter runs *before* the decimation.

With the filter **on**, the sweep climbs and then stops. With it **off**, it climbs, reaches a ceiling, and descends *back down* while the real tone continues upward. That descending sound is not present anywhere in the input; it is produced by the sampler.

> **Figure 6 · audio — A rising sweep, sampled too slowly**
>
> This uses a real windowed-sinc filter, integer decimation, and bandlimited reconstruction. Use headphones or speakers, then press play.
>
> Compare the two. With the filter on, the top of the sweep is lost — a known, bounded loss. With it off, data is retained but incorrect. Between missing data and confidently incorrect data, missing data is preferable.
>
> *(interactive figure — see the web page)*

---

*§7*

## Reducing a rate in code

Everything so far has concerned the moment of measurement, where a converter or a sensor turns a continuous quantity into an array. Most programmers never write that code. What they do write, routinely, is code that takes an array which already exists and produces a smaller one — and that operation is a second act of sampling, governed by the same theorem.

**Downsampling** means lowering the rate of data already captured, by keeping one of every N values and discarding the rest. It is done constantly and for ordinary reasons: storage and bandwidth, thumbnails generated from full-resolution images, metrics rolled up from per-second to per-minute, audio converted between 48 kHz and 44.1 kHz.

The trap is that dropping samples *is sampling again*, at a rate N times lower, so the theorem applies a second time. The limit moves down with the rate, and anything above the new half-rate is reflected back down and reappears as a lower frequency, which is the folding described in [the section on aliasing](#alias). The difference is that here the original is already on disk, and the code doing the reading is what introduces the fold.

Preventing it requires removing those components first. A **low-pass filter** passes everything below a chosen cutoff and removes everything above it. That is the same operation as cropping the spectrum back to a single copy in [the section on spectral copies](#copies), applied to an array instead of a spectrum. Set the cutoff at the new half-rate and the values that remain are genuinely redundant, so discarding them costs nothing. The order is the whole point: filter, then drop. Dropping first destroys the information the filter needed.

```go
package main

import "fmt"

// a plain moving-average low-pass: crude, but it removes the fast wiggle
func lowPass(x []float64, width int) []float64 {
	out := make([]float64, len(x))
	for i := range x {
		sum, count := 0.0, 0
		for j := i - width; j <= i+width; j++ { // average the 2*width+1 samples around i
			if j >= 0 && j < len(x) {
				sum += x[j]
				count++
			}
		}
		out[i] = sum / float64(count)
	}
	return out
}

func main() {
	input := make([]float64, 32) // the signal at the original rate
	for i := range input {
		input[i] = float64(i % 2) // fastest possible wiggle: 0,1,0,1,...
	}

	keep := 4                                 // keep every 4th sample: the rate drops by 4
	wrong := make([]float64, len(input)/keep) // samples dropped straight away
	right := make([]float64, len(input)/keep) // filtered first, then dropped

	// WRONG — this is exactly the "filter off" button above
	for n := range wrong {
		wrong[n] = input[n*keep]
	}
	// anything above the NEW half-rate is now permanently disguised as something slower

	// RIGHT — remove what the new rate can't represent, THEN drop samples
	filtered := lowPass(input, 2) // cutoff near the new half-rate
	for n := range right {
		right[n] = filtered[n*keep]
	}
	// now the discarded samples really were redundant

	fmt.Println("wrong:", wrong) // all 0s: the wiggle became a false DC level
	fmt.Println("right:", right) // ~0.5: reports "too fast to represent"
}
```

The same error appears when shrinking an image by taking every N-th pixel instead of area-averaging, when thinning a metrics series by keeping every N-th point instead of averaging the bucket, and when animating a spinning wheel by evaluating its angle once per frame. These are all the same bug.

In none of those three is there an obvious signal, a sample rate, or anything a programmer would call a filter — there is a frame loop, an image resize, and a rollup query.

---

*§8*

## Undersampling on purpose

"Sample at twice the highest frequency" is the commonly cited form of the rule, and it is stricter than necessary. What matters is that the copies do not overlap, which depends on how *wide* the signal's band is, not on how high it sits.

Take FM radio between 100 and 102 MHz. The highest frequency present is 102 MHz, so the familiar rule asks for 204 million measurements per second. But the signal only occupies 2 MHz of the axis; everything below 100 MHz and above 102 MHz is empty. Sampling folds the whole axis down into the baseband — the stretch from 0 to half the sampling rate, which is all a set of samples can express — and if the fold happens to place this 2 MHz of content where nothing else lands, none of it is lost. Sampling at 4 million measurements per second does exactly that.

The saving is in the converter alone. A part that measures 4 million times a second is far slower, cheaper and lower-powered than one that measures 204 million times a second, and that is the entire point of the technique. Nothing else gets easier: the analogue input must still respond accurately to a 102 MHz wave, and the filter in front of it must now pass 100 to 102 MHz while rejecting everything else, which is harder to build than a filter that simply removes everything above a limit. Software-defined radios and instrumentation front-ends are built this way.

### Why only some rates work

Recall that a sampling rate cuts the frequency axis into zones, each half the rate wide, and that every zone folds down onto the baseband. For this to be usable, the band has to sit *inside a single zone*. If it does, the whole band folds down together and keeps its internal spacing, so it can be recovered. If instead the band lies across the boundary between two zones, the part below the boundary and the part above it fold down on top of each other and are added together. That is the case the figure calls **straddling**, and it destroys the signal: the two halves are summed into one set of numbers with no way to separate them again.

With the band at 100 to 102 MHz and a rate of 4 MHz, the zones are 2 MHz wide, so their boundaries fall at 0, 2, 4 and so on — including at exactly 100 and 102. The band fills one zone precisely and nothing straddles. Change the rate to 4.4 MHz and the zones become 2.2 MHz wide, with boundaries at 99 and 101.2. Now the band lies across 101.2, the two parts fold onto one another, and the signal is gone. The band did not move; only the spacing of the boundaries did.

This is why raising the rate does not always help and lowering it does not always hurt. Workable rates come in separate ranges with unusable gaps between them, given by:

**n2f2≤fs≤n−12f1,n=1,2,…,⌊f2−f1f2⌋**

- **f1,f2** — the bottom and top of the signal's band.

- **n** — the number of the zone the band folds down from. n=1 is ordinary sampling.

- **⌊ ⌋** — round down to an integer.

Rather than evaluate that by hand, use the figure. It has two parts, and they answer two different questions about the same choice of rate.

The **upper bar** is a survey: it runs along every sampling rate the slider can reach, from left to right, and colours each one according to whether a band of the chosen position and width would survive it. Green means the band falls inside a single zone at that rate; orange means it would straddle a boundary. The dashed marker shows which rate is currently selected. The green stretches are labelled with the number of the zone the band ends up in.

The **lower bar** zooms in on that one selected rate and shows the outcome: the baseband, from 0 to half the selected rate, with the band drawn where it actually arrives after folding. When the selected rate is a green one, a solid block shows the stretch of the baseband the band now occupies, and everything else in the baseband is free. When the rate is an orange one, there is no meaningful position to draw, because the band has been added to itself.

So the upper bar answers "which rates are usable at all", and the lower bar answers "if this rate is used, where does the signal end up". Moving the rate slider moves the marker in the upper bar and redraws the lower one.

> **Figure 7 · live — Which sampling rates work for a band, and where it ends up**
>
> Upper bar: every rate, coloured by whether it works. Lower bar: for the selected rate, where the band arrives in the baseband. The workable ranges get narrower as the rate falls, which means a lower rate has to be held to a tighter accuracy to stay inside one.
>
> Widening the band, or moving it, redraws the whole upper bar: which rates work is a property of the band, not a fixed list. Two costs are unchanged by any of this. The input circuit must still respond accurately to the highest frequency in the band, 102 MHz here, because that is the wave physically arriving at it. And the filter in front must now pass only the band itself, rejecting everything above and below, since any content in another zone folds down onto the signal just as readily. The saving is in how often the converter has to measure, and nowhere else.
>
> *(interactive figure — see the web page)*
