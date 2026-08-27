# Digital media study

## What this?

Interactive explainers on digital audio and video, aimed at software engineers.

- **`/waves`** — _Waves, for programmers._ Amplitude, frequency, phase, superposition, spectrum
  and wavelength, each defined and demonstrated. Ends with the spectrogram window tradeoff.
- **`/sampling`** — _Sampling, for programmers._ The Nyquist–Shannon theorem made interactive:
  aliasing, reconstruction, and why the stair-step picture is wrong. _Coming soon._
- **`/primer`** — _A digital media primer for geeks._ Bit depth and dynamic range, dither,
  µ-law companding, interlacing, gamma, Y′CbCr and chroma subsampling, pixel formats,
  containers. _Coming soon._

Every code sample comes in both Go and JavaScript, behind a toggle.

Built as an interactive companion to Monty Montgomery's
[A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks),
with material from Wikipedia.

> [!NOTE]
> The pages, samples and write-up are ~90% generated with Claude AI.
> I review and curate these pages to my liking, but my edits are minimal.
> The content is based on Wikipedia and [A Digital Media Primer For Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) by Monty Montgomery.
> The value added is interactive examples and some code samples. I use this for my own learning.
>
> Both sources are Creative Commons Attribution-ShareAlike: Wikipedia under
> [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), and the primer's text and
> wiki edition under CC BY-SA 3.0 (© 2010 Xiph.Org and Red Hat Inc.). ShareAlike means a
> derivative work must carry the same license, so this repo is also
> [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Copyright in the source
> material stays with its authors; I claim nothing beyond my own additions, and those are
> offered under the same license.

See [LICENSE](LICENSE) for the full text and attribution details.

## How this is built

Each page is a single self-contained HTML file — all CSS, JavaScript, and the KaTeX stylesheet
are inlined, and every figure is drawn at runtime on a `<canvas>`.
