<!-- Generated from waves/index.html — the interactive figures live there. -->

*Waves, for programmers*

# A wave is described completely by *three numbers*.

Amplitude, frequency, and phase. Those three terms, plus spectrum, appear early in every signal processing text and are then used on every page that follows. This page defines each one, shows what it controls, and stops where the sampling theorem begins. Every figure is interactive and every code example is a complete, runnable program, provided in Go and JavaScript.

A wave is a quantity that oscillates. Each term below answers one question about it: **amplitude** answers how large the swing is, **frequency** and **period** answer how fast it repeats, **wavelength** answers how far one repetition reaches in space, and **phase** answers where in its cycle the wave currently sits. A **spectrum** answers all of those for every frequency at once.

The examples use audio, because the numbers there are familiar. Nothing on this page is specific to sound. The only assumption is that a sine function takes an input and returns an output.

---

## Amplitude, frequency, period

A pure tone is fully specified by three numbers, and that specification is exact. A sine carries energy at exactly one frequency, and every other signal — however complex — is a sum of sines, each with its own amplitude, frequency, and phase. Because every signal decomposes this way, the sine is the unit the later sections build on: square waves, spectra and spectrograms are all assembled from sines. Here is that single sine, written out:

**x(t)=Asin(2π(ft+φ))**

**Period** is the frequency written the other way round: T=1/f is the time one cycle takes. A 2 Hz wave has a period of 0.5 seconds. Both forms describe the same fact, and each is convenient in different places.

The three controls below are independent of each other. Amplitude scales the wave vertically, frequency changes how many cycles fit in the same span, and phase shifts it sideways without altering its shape.

> **Figure 1 · live — Amplitude, period and phase marked on one wave**
>
> The orange bracket measures amplitude. The green bracket measures one period. The grey dashed curve, which appears once phase is non-zero, is the same wave at φ=0.
>
> Hover the plot to read the value at any instant and the number of cycles elapsed. A phase of 1.00 turns produces the same wave as 0.00, because phase is circular. This is why phase is measured as an angle.
>
> *(interactive figure — see the web page)*

The same three numbers in code. The period is derived rather than stored, since it carries no information that the frequency does not already hold:

```go
package main

import (
	"fmt"
	"math"
)

const (
	amplitude = 1.0 // peak swing away from rest, in whatever unit the wave carries
	frequency = 2.0 // complete cycles per second, in hertz
	phase     = 0.0 // starting position within the cycle, in turns (1.0 = a full cycle)
)

// wave returns the value of the tone at time t, in seconds.
func wave(t float64) float64 {
	return amplitude * math.Sin(2*math.Pi*(frequency*t+phase))
}

func main() {
	period := 1 / frequency // seconds per cycle: the reciprocal of frequency

	fmt.Printf("amplitude %.2f, frequency %.2f Hz, period %.3f s\n",
		amplitude, frequency, period)

	// One full cycle, sampled at eight evenly spaced instants.
	for n := 0; n < 8; n++ {
		t := float64(n) * period / 8
		fmt.Printf("t = %.3f s   value = %+.3f\n", t, wave(t))
	}
}
```

The output shows the wave passing through zero, rising to the amplitude, and returning — one full cycle across eight evenly spaced instants.

> Many texts write sin(ωt), where ω=2πf is called *angular frequency* and is measured in radians per second. It is the same quantity with the factor of 2π already applied, which shortens the formulas that use calculus. To convert back to cycles per second, divide by 2π.

---

## Phase

Phase behaves differently from the other two numbers, because it has no meaning for a single wave considered on its own. Asking for the phase of one isolated sine has no fixed answer: it depends on where t=0 is placed, and moving that origin changes the phase without changing the wave.

Phase becomes measurable when there are two waves. The difference between their phases is a physical property that does not depend on the choice of origin, and it determines the result when the two are combined.

> **Figure 2 · live — Two waves at the same frequency, and their sum**
>
> Both inputs are 2 Hz with amplitude 1 at every slider position. Only the phase difference between them changes.
>
> The same effect appears in noise-cancelling headphones, in the change of tone when a microphone is moved a short distance, and when two recordings of the same part are mixed together.
>
> *(interactive figure — see the web page)*

The figure computed as a program. The two waves keep the same amplitude throughout; only the offset between them changes:

```go
package main

import (
	"fmt"
	"math"
)

const (
	frequency = 2.0  // both waves are 2 Hz; only their phase differs
	steps     = 2000 // how many instants to check when searching for the peak
	span      = 1.0  // seconds to search over: two full cycles at 2 Hz
)

// peakOfSum returns the largest absolute value reached by wave A plus wave B,
// where B is shifted by diff turns relative to A.
func peakOfSum(diff float64) float64 {
	peak := 0.0
	for i := 0; i <= steps; i++ {
		t := span * float64(i) / steps
		a := math.Sin(2 * math.Pi * frequency * t)
		b := math.Sin(2 * math.Pi * (frequency*t + diff))
		if s := math.Abs(a + b); s > peak {
			peak = s
		}
	}
	return peak
}

func main() {
	// Each wave alone peaks at 1.0. Their sum peaks anywhere from 2.0 to 0.0,
	// decided entirely by the phase difference.
	for _, diff := range []float64{0, 0.125, 0.25, 0.375, 0.5} {
		fmt.Printf("phase difference %.3f turns (%3.0f deg)   peak of sum %.3f\n",
			diff, diff*360, peakOfSum(diff))
	}
}
```

At 0 turns the peaks coincide and the amplitudes add to 2.0. At 0.5 turns one wave is positive wherever the other is negative, and the sum is zero everywhere. The values in between follow 2∣cos(πd)∣ for a difference of d turns.

A short summary: amplitude describes how much energy a wave carries, and phase describes how it will combine with other waves. Any system that adds signals together — a mixer, a room with reflecting surfaces, an array of antennas — produces a result that depends on phase.

There is one asymmetry worth recording. Human hearing is largely insensitive to the absolute phase of the components within a steady sound, so two signals with the same spectrum but different phases can sound very similar. This is part of why a magnitude spectrum, which discards phase entirely, remains a useful picture. It is also why recovering a signal from a magnitude spectrum alone is a difficult problem rather than a direct calculation.

---

## Superposition

**Superposition** is the rule that when two waves occupy the same place, the result is their sum at each instant. It is a plain sum: the air pressure at a given point is one number, equal to the total of the contributions from every source present.

Applied in reverse, this rule is the basis of the rest of the subject. If sines added together produce other shapes, then other shapes can be decomposed back into sines. Adding harmonics below shows a square wave being assembled from nothing but smooth curves.

> **Figure 3 · live — A square wave assembled from sines**
>
> Each faint orange curve is one sine. The blue curve is their total. The lower panel lists the amplitude of each component in use.
>
> On the square and sawtooth, the overshoot near each step is the Gibbs phenomenon. It settles at about 9% of the step height and does not shrink further as terms are added, though it does become narrower. The same overshoot appears as ringing near sharp edges in filtered audio and in JPEG images. The triangle behaves differently: its harmonics fall off as 1/k2 rather than 1/k, so it converges quickly and without ringing.
>
> *(interactive figure — see the web page)*

The overshoot is easier to trust as a number than as a picture. This program sums an increasing number of harmonics and reports the peak each time:

```go
package main

import (
	"fmt"
	"math"
)

const (
	fundamental = 1.0  // frequency of the lowest component, in hertz
	steps       = 4000 // instants examined across one period
)

// square sums the first n odd harmonics of a square wave.
// Harmonic k has amplitude 4/(pi*k), and only odd k are present.
func square(t float64, n int) float64 {
	sum := 0.0
	for k := 1; k <= n; k += 2 {
		sum += 4 / (math.Pi * float64(k)) * math.Sin(2*math.Pi*fundamental*float64(k)*t)
	}
	return sum
}

func main() {
	// A true square wave steps from -1 to +1, a jump of 2.0, and holds at 1.0.
	// Each partial sum overshoots that level near the step. The overshoot narrows
	// as terms are added but settles at about 9% of the jump rather than vanishing.
	const step = 2.0 // height of the square wave's jump, from -1 to +1
	for _, n := range []int{1, 3, 9, 33, 129} {
		peak := 0.0
		for i := 0; i <= steps; i++ {
			t := float64(i) / steps / fundamental
			if v := square(t, n); v > peak {
				peak = v
			}
		}
		fmt.Printf("harmonics up to %3d   peak %.4f   overshoot %.2f%% of the jump\n",
			n, peak, (peak-1)/step*100)
	}
}
```

The peak falls from 27% above the target to about 8.9% and then stops. Adding more terms moves the overshoot closer to the step but does not reduce its height.

> The second panel of Figure 3 lists which frequencies are present and how much of each. That is a spectrum, assembled here by choosing the components directly. The next section describes what that picture means in general.

---

## Spectrum

A **spectrum** describes a signal by its ingredients rather than by its behaviour over time: for each frequency, how much of that frequency the signal contains. Both descriptions are complete and both describe the same signal. The **Fourier transform** is the calculation that converts between them.

The horizontal axis carries a different meaning in each picture. On a waveform, moving right means later in time. On a spectrum, moving right means a higher frequency. A tall bar on the right of a spectrum is not an event that happens at the end; it is a component that oscillates quickly and is present throughout.

Most explanations start from a waveform and compute its spectrum. The figure below works in the other direction: set the amount of each frequency, and read off the waveform that those choices produce.

> **Figure 4 · live — Choosing a spectrum, and reading the resulting waveform**
>
> Drag any bar in the lower panel. The waveform above is the sum of the components those bars specify.
>
> Enabling the phase option changes the shape of the waveform while leaving every bar in the lower panel where it is. That difference is the information a magnitude spectrum does not record.
>
> *(interactive figure — see the web page)*

Computing a spectrum from a signal is a direct calculation. The program below builds a signal from two known components and then recovers them, which makes the result checkable against the input:

```go
package main

import (
	"fmt"
	"math"
	"math/cmplx"
)

const (
	sampleRate = 64.0 // measurements per second
	n          = 64   // number of measurements; one full second of signal
)

// dft returns the magnitude of each frequency bin in x.
// Bin k corresponds to k*sampleRate/n hertz.
func dft(x []float64) []float64 {
	mag := make([]float64, len(x)/2)
	for k := range mag {
		sum := complex(0, 0)
		for t, v := range x {
			angle := -2 * math.Pi * float64(k) * float64(t) / float64(len(x))
			sum += complex(v, 0) * cmplx.Exp(complex(0, angle))
		}
		// Scale so a sine of amplitude A reports A in its bin.
		mag[k] = 2 * cmplx.Abs(sum) / float64(len(x))
	}
	return mag
}

func main() {
	// A signal built from two known ingredients, so the answer is checkable.
	signal := make([]float64, n)
	for i := range signal {
		t := float64(i) / sampleRate
		signal[i] = 1.0*math.Sin(2*math.Pi*4*t) + 0.5*math.Sin(2*math.Pi*11*t)
	}

	binWidth := sampleRate / n // hertz covered by each bin
	fmt.Printf("bin width %.2f Hz\n\n", binWidth)

	for k, m := range dft(signal) {
		if m < 0.01 { // skip bins holding no meaningful energy
			continue
		}
		fmt.Printf("%5.1f Hz   magnitude %.3f\n", float64(k)*binWidth, m)
	}
}
```

The output reports 1.000 at 4 Hz and 0.500 at 11 Hz, matching the two components used to build the signal. This is the discrete Fourier transform written directly from its definition. It takes time proportional to n2; the fast Fourier transform computes the same result in time proportional to nlogn, and is what production code uses.

### Bandwidth

Once a signal is described by its spectrum, one number summarises it for most engineering purposes. **Bandwidth** is the width of the frequency region the signal occupies. A signal with no content above 20 kHz has a spectrum that stops at 20 kHz, and that fact alone determines how fast it has to be measured.

This is where the next page continues. The [sampling theorem](../sampling/) is a statement about the spectrum rather than the waveform: measure faster than twice the highest frequency present and the samples are a complete record; measure more slowly and copies of the spectrum overlap, which cannot be undone.

---

## Wavelength

Everything above describes a wave oscillating in time at one fixed point. **Wavelength** describes the other axis: for a wave travelling through a medium, the distance covered by one complete cycle.

**λ=fv**  
*v = speed in the medium*

Wavelength is therefore a property of the signal together with the medium carrying it, rather than of the signal alone. A 440 Hz tone measures about 78 cm in air, 3.4 m in water and 13.5 m in steel. The frequency is the same in all three; the propagation speed is not.

This is why wavelength appears rarely in signal processing code. Once a wave has been recorded into an array of numbers there is no medium and no propagation speed, so wavelength is undefined and only frequency remains. It becomes relevant again whenever physical distance is part of the problem: loudspeaker placement and room acoustics, microphone arrays, antenna dimensions, ultrasound, and any measurement taken by two sensors in different positions.

> **Figure 5 · calculator — One frequency, four media**
>
> Frequency is set by the source. Wavelength follows from the medium.
>
> sound in air (20 °C). Sound travels slowly enough in air that a low note is several metres long, which is one reason low frequencies are difficult to control in small rooms: the room is not much larger than the wave.
>
> *(interactive figure — see the web page)*

The same calculation as a program:

```go
package main

import "fmt"

// Propagation speeds in metres per second. Wavelength depends on the medium,
// so the same frequency is a different length in each of these.
const (
	speedInAir   = 343.0       // dry air at 20 degrees Celsius
	speedInWater = 1481.0      // fresh water at 20 degrees Celsius
	speedInSteel = 5960.0      // longitudinal waves in steel
	speedOfLight = 299792458.0 // electromagnetic waves in vacuum
)

func main() {
	const frequency = 440.0 // hertz: concert A

	media := []struct {
		name  string
		speed float64
	}{
		{"air", speedInAir},
		{"water", speedInWater},
		{"steel", speedInSteel},
		{"vacuum (as radio)", speedOfLight},
	}

	for _, m := range media {
		wavelength := m.speed / frequency // metres per cycle
		fmt.Printf("%-18s %12.1f m/s   wavelength %10.3f m\n",
			m.name, m.speed, wavelength)
	}
}
```

---

## Spectrogram

A spectrum describes the ingredients of a signal across its whole duration. That is sufficient for a steady tone, but not for music or speech, where the content changes. The spectrum of a complete recording reports which notes occur without reporting when.

The usual solution is to compute a spectrum for each part of the signal and place the results side by side. This is a **spectrogram**, and it is what audio analysis tools display. One detail in that description needs care.

**A spectrum at a single instant does not exist.** Frequency describes how a quantity changes across an interval, so an instant has no frequency, in the same way that a single photograph has no speed. A spectrogram therefore selects a **window**, a span of time, moves it along the signal, and computes one spectrum per position. The length of that window has two consequences that pull in opposite directions:

Both cannot be improved at once. The product Δt⋅Δf has a lower bound, a result known as the Gabor limit. It is the same uncertainty relation that appears in quantum mechanics, and it applies to any method of analysis rather than to particular algorithms.

> **Figure 6 · live — The same signal at different window lengths**
>
> The default signal contains two steady tones 20 Hz apart and two short clicks. Separating the tones requires a long window. Locating the clicks requires a short one.
>
> Hover the spectrogram to read time, frequency and magnitude at any point. The Δt·Δf column stays at 1.00 for every window length, because with these definitions Δf=1/Δt exactly. The tradeoff is arithmetic rather than an approximation. The window is drawn to scale on the waveform above, showing how much of the signal each individual spectrum covers.
>
> *(interactive figure — see the web page)*

The relationship between the two resolutions, without the graphics:

```go
package main

import "fmt"

const sampleRate = 1000.0 // measurements per second

func main() {
	// A spectrogram computes one spectrum per window of the signal.
	// The window length fixes both resolutions at once, in opposite directions.
	fmt.Printf("%8s %12s %14s %10s\n", "samples", "window span", "bin spacing", "product")

	for _, n := range []int{16, 64, 256, 1024} {
		span := float64(n) / sampleRate     // seconds of signal inside one window
		binWidth := sampleRate / float64(n) // hertz between adjacent bins

		// The product is 1 for every window length. Improving one resolution
		// always costs exactly as much of the other.
		fmt.Printf("%8d %9.3f s %11.2f Hz %10.2f\n",
			n, span, binWidth, span*binWidth)
	}
}
```

Each of the four signals in Figure 6 responds differently. *Tones only* does not change over time, so its spectrogram is a set of horizontal lines and a long window is simply better. *Clicks only* is the opposite case, where the timing is the content and a long window removes it. *Chirp* has a frequency that rises continuously, so every window length is a compromise between the frequency at the start of the window and the frequency at its end.

This is the reason audio tools ask for an FFT size, and why speech analysis commonly uses windows of 20 to 30 ms: long enough to resolve the harmonics of a voice, short enough that a consonant is not merged with the vowel following it. The correct value depends on what is being measured.

Taking a section out of a signal is itself an operation with an effect on the result. A plain rectangular section starts and stops abruptly, and those two artificial edges are sharp transitions. The analysis reports them as energy spread across the whole frequency axis, even though they are not present in the signal. This is called spectral leakage.

The standard remedy is to fade the section in and out instead of cutting it, by multiplying it by a smooth curve that reaches zero at both ends. Figure 6 uses a **Hann window**, w[n]=0.5−0.5cos(2πn/(N−1)), which is a common default. Alternatives such as Hamming, Blackman and Kaiser trade the width of the main peak against how far the surrounding leakage is suppressed.

---

## The terms in one place

| Term | Symbol | What it describes | Note |
|---|---|---|---|
| **Amplitude** | A | How far the quantity swings from rest | The unit depends on the medium. In an audio buffer it is a convention rather than a physical quantity |
| **Frequency** | f | Cycles per second, in hertz | Angular frequency ω=2πf is the same quantity scaled. Check which one a formula expects |
| **Period** | T | Seconds per cycle, equal to 1/f | In sampling code, T usually refers to the spacing between samples, which is a different period |
| **Phase** | φ | Position within the cycle | Defined only as a difference between two waves. Repeats after one full turn |
| **Wavelength** | λ | Metres per cycle in space, equal to v/f | Requires a medium. Undefined for a signal held in an array |
| **Superposition** | — | Waves in the same place add at each instant | The sum can be smaller than either input, including zero |
| **Spectrum** | X(f) | How much of each frequency is present | A magnitude spectrum ∣X∣ does not record phase |
| **Bandwidth** | B | The width of the occupied frequency region | The quantity the sampling theorem depends on |
| **Spectrogram** | — | How the spectrum changes over time | Depends on a chosen window length. No setting is precise in both time and frequency |

### Next

With these terms defined, [the sampling theorem](../sampling/) can be stated as a property of shapes on a frequency axis rather than as a rule about twice the highest frequency. After that, [the media primer](../primer/) covers what happens when those samples are stored as finite-precision numbers.

One of a set of interactive explainers on digital media. Every figure is computed in the browser from the definitions given in the text: the harmonics are summed as described, and the spectrogram is a short-time Fourier transform running on a synthesised signal. Every code example is a complete program in Go and JavaScript; the Go version compiles and runs with `go run`, the JavaScript version with `node`.

[← back to the index](../)
