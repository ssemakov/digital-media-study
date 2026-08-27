# digital-media-study

Interactive explainers on digital audio and video, aimed at software engineers.

- **`/waves`** — *Waves, for programmers.* Amplitude, frequency, phase, superposition, spectrum
  and wavelength, each defined and demonstrated. Ends with the spectrogram window tradeoff.
- **`/sampling`** — *Sampling, for programmers.* The Nyquist–Shannon theorem made interactive:
  aliasing, reconstruction, and why the stair-step picture is wrong. *Coming soon.*
- **`/primer`** — *A digital media primer for geeks.* Bit depth and dynamic range, dither,
  µ-law companding, interlacing, gamma, Y′CbCr and chroma subsampling, pixel formats,
  containers. *Coming soon.*

Every code sample comes in both Go and JavaScript, behind a toggle.

Built as an interactive companion to Monty Montgomery's
[A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks),
with material from Wikipedia.

## Branches

`main` is the working branch and holds drafts that have not been reviewed yet. `live` is what
Vercel deploys — pages land there one at a time, after review. Pages still in review show as
greyed-out "Coming soon" cards on the landing page.

## How this is built

Each page is a single self-contained HTML file — all CSS, JavaScript, and the KaTeX stylesheet
are inlined, and every figure is drawn at runtime on a `<canvas>`. There are no external assets,
no fonts to fetch, no build step, and no dependencies at runtime.

```
index.html        landing page, ~9 KB
waves/index.html  ~364 KB, self-contained
```

## Running locally

Any static file server works:

```bash
python3 -m http.server 8000
# → http://localhost:8000
```

Opening the HTML files directly with `file://` also works, though the root-relative link on the
landing page (`/waves`) will not resolve — use a server for that.

## Deploying

The repo is a plain static site with no build step, so it deploys as-is on Vercel, Netlify,
Cloudflare Pages, or GitHub Pages.

On Vercel: import the repo, set the production branch to `live`, leave the framework preset as
**Other**, and leave the build command and output directory empty. The directory-based routes
(`waves/index.html` → `/waves`) work without any configuration.
