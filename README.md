# digital-media-study

Two interactive explainers on digital audio and video, aimed at software engineers.

- **`/sampling`** — *Sampling, for programmers.* The Nyquist–Shannon theorem made interactive:
  aliasing, reconstruction, and why the stair-step picture is wrong.
- **`/primer`** — *A digital media primer for geeks.* Bit depth and dynamic range, dither,
  µ-law companding, interlacing, gamma, Y′CbCr and chroma subsampling, pixel formats, containers.

Built as an interactive companion to Monty Montgomery's
[A Digital Media Primer for Geeks](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks).

## How this is built

Each page is a single self-contained HTML file — all CSS, JavaScript, and the KaTeX stylesheet
are inlined, and every figure is drawn at runtime on a `<canvas>`. There are no external assets,
no fonts to fetch, no build step, and no dependencies at runtime.

```
index.html          landing page
sampling/index.html  ~450 KB, self-contained
primer/index.html    ~415 KB, self-contained
```

## Running locally

Any static file server works:

```bash
python3 -m http.server 8000
# → http://localhost:8000
```

Opening the HTML files directly with `file://` also works, though the root-relative links on the
landing page (`/sampling`, `/primer`) will not resolve — use a server for that.

## Deploying

The repo is a plain static site with no build step, so it deploys as-is on Vercel, Netlify,
Cloudflare Pages, or GitHub Pages.

On Vercel: import the repo, leave the framework preset as **Other**, and leave the build command
and output directory empty. The directory-based routes (`sampling/index.html` → `/sampling`) work
without any configuration.
