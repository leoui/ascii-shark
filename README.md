# ascii-shark

A full-screen, never-ending ASCII ocean. A shark cruises through open water, a school of
small fish keeps its distance, light falls from the surface, and every frame is drawn live
in coloured monospace characters. Nothing is pre-animated and nothing loops: where the shark
goes, what it does next and when it does it are decided as the page runs.

**Live:** [verdatama.com](https://verdatama.com)

```
                         /\
                        /||\
          ___----------/||||\---------___
     ,-'' 0o0Oo0oO0o0Oo0oO0o0Oo0o0Oo0o0Oo``--.__        \\
    <=@ o0O0o0OoO0o0o0O0oO0o0O0o0oO0o0O0o0O0o0O0o0>-----===<<
     `-.__ :;:;:;:;:;:;:;:;:;:;:;:;:;:;:;:;:;:__,--''        //
          ``---..____    \\\        ____..--''
                     ``---\\\--''           >  >   >
                                          >   >  >   >
```

It is a single self-contained `index.html`: no build step, no dependencies, no network
requests, no images or fonts to load. Drop it on any static host and it fills whatever
screen it is opened on.

---

## Contents

- [What you will see](#what-you-will-see)
- [Running it](#running-it)
- [Deploying](#deploying)
- [How it works](#how-it-works)
  - [The rendering pipeline](#the-rendering-pipeline)
  - [The shark's body](#the-sharks-body)
  - [Swimming](#swimming)
  - [Behaviours](#behaviours)
  - [The school of fish](#the-school-of-fish)
  - [The rare feed](#the-rare-feed)
  - [Why it cannot be predicted](#why-it-cannot-be-predicted)
  - [Water, light and glow](#water-light-and-glow)
- [Performance](#performance)
- [Tuning](#tuning)
- [Debugging from the console](#debugging-from-the-console)
- [Browser support](#browser-support)
- [Inspiration](#inspiration)

---

## What you will see

- **A shark** with countershading (blue-grey back, white belly and a wavy line between them),
  five gill slits, eyes, a mouth, two dorsal fins, pectoral, pelvic and anal fins, and a
  shark's uneven tail whose upper lobe is longer than the lower one.
- **Smaller sharks in the distance**, fainter and slower, each with its own colouring and
  its own rhythm.
- **A school of small fish** (`<` `>` `^` `v`) that sticks together, turns as one and
  scatters when a shark comes near.
- **Marine snow**, beams of sunlight that sway in a fan from the surface, and a rippling
  band of light at the very top of the water.
- **Behaviours** that come and go on their own: cruising, turning back, swimming off the
  screen and returning, coming right up to the lens, charging the school, and very rarely,
  catching and eating a fish.

The site's host name is shown faintly in the bottom-left corner and used as the page title.
The cursor hides itself after a few seconds of stillness; double-click toggles full screen.

---

## Running it

Open `index.html` in a browser. That is all.

To serve it locally (recommended, so that the browser's storage works; see
[the rare feed](#the-rare-feed)):

```bash
python3 -m http.server 8000
```

then visit <http://localhost:8000>.

---

## Deploying

Any static host works, because the whole site is one HTML file.

### Cloudflare Pages

```bash
npx wrangler pages project create ascii-shark --production-branch main
npx wrangler pages deploy . --project-name ascii-shark --branch main
```

To serve it on your own domain, add the domain under the project's **Custom domains** tab.
If the domain's DNS is on Cloudflare, a proxied `CNAME` from the apex to
`<project>.pages.dev` is all it needs. To send `www` to the bare domain, add a
**Redirect Rule** (it is included in the free plan):

| Field              | Value                                                   |
| ------------------ | ------------------------------------------------------- |
| When               | `http.host eq "www.example.com"`                        |
| Type               | Dynamic, status `301`                                   |
| URL                | `concat("https://example.com", http.request.uri.path)` |
| Preserve query     | on                                                      |

`www` still needs a proxied DNS record of its own so that the request reaches Cloudflare
and the rule can answer it.

### GitHub Pages, Netlify, Vercel, S3, …

Publish the repository root as it is. There is nothing to build.

---

## How it works

Everything lives in one `<script>` inside `index.html`, in sections marked with
`// [name]` comments: `[chance]`, `[body]`, `[swimmer]`, `[wave]`, `[away]`, `[close]`,
`[feed]` and `[school]`.

### The rendering pipeline

Every frame goes through the same five steps:

1. **Gather points.** Each object in the scene (sharks, fish, specks of snow, bits left
   after a bite) is turned into thousands of points in 3D. Each point carries a position,
   a colour, a brightness, what kind of thing it belongs to, and for line-like things
   (fins, gill slits) a second point that gives its direction.
2. **Project.** A perspective camera turns each point into a cell of a character grid.
   The grid's size follows the window: the font size is about 1/72 of the screen height,
   and the number of columns and rows is however many fit.
3. **Z-buffer.** Each cell keeps only the nearest point that landed in it, so the near side
   of the shark hides the far side and a fish behind the shark is hidden.
4. **Choose characters.**
   - Skin uses a brightness ramp, `.:;+*oO0#@`, from dim to bright.
   - Fins and gill slits are strokes: whichever of `- / | \` lies closest to the direction
     of the fin's rays *on screen*, measured in cells (which are taller than they are wide).
   - Fish point where they swim: `>` `<` `^` `v`.
   - An **outline pass** then finds the cells where the shark's body meets open water and
     redraws them as strokes that follow the silhouette, which gives the body a crisp edge.
5. **Paint and glow.** The characters are drawn onto an offscreen canvas. That canvas is
   shrunk to a third and to a ninth of its size, and both small copies are stretched back
   over the picture with additive blending. The stretched copies blur on their own, which
   gives a soft glow and a wider haze without any blur filter.

### The shark's body

The body is 1 unit long in its own frame and scaled afterwards, so the same model draws the
big shark and the small distant ones.

- **Profile.** Along the body, `s` runs from the snout (0) to the root of the tail (1).
  Two tables give the half-height and half-width at each `s`; the cross-section is an
  ellipse with a slightly flatter belly.
- **Normals and light.** The surface is sampled on a grid along `s` and around the body,
  and every normal is computed from its neighbours after the body has been bent, so the
  shading follows the swimming wave. Light comes from above (the surface), with a faint
  highlight on the back.
- **Countershading.** The line between the grey back and the white belly wanders a little
  along the body, as it does on a real shark.
- **Marks on the skin.** Gill slits, the mouth and the eyes are drawn just outside the skin
  so that they always win over it in the z-buffer.
- **Fins** are flat sheets defined by a root edge on the body and a tip. Each fin is filled
  with strokes that run from root to tip and is darker towards the tip.
- **The tail** is a stiff blade beyond its root: it does not stretch with the wave; it
  turns with the slope of the wave where it joins the body.
- **Level of detail.** The number of points depends on how big the shark looks: a distant
  shark gets a few hundred, the big shark gets more, and a shark right in front of the lens
  gets up to about twice as many again so that it does not break into dots.

### Swimming

No path is drawn. Each shark has a heading, a pitch, a speed, an effort and a tail phase,
and a small simulation runs 60 times a second:

- **Steering.** It swings its heading towards its current target no faster than a shark can
  turn. A shark that is barely moving cannot turn hard. It tips its nose up or down towards
  the target and **banks** into the turn.
- **The tail.** Its beat speeds up with effort. Each sweep pushes, and the push is strongest
  in the middle of a sweep.
- **The water** holds it back with drag that grows with the square of its speed, so effort
  becomes cruising speed on its own.
- **The body wave.** The body bends sideways as a wave that runs from head to tail. The
  head hardly moves; the amplitude grows sharply towards the tail. A turn also bends the
  whole spine into a gentle C.
- **Feeling of weight.** A burst comes on quickly and fades slowly; the shark coasts after
  it lets up.

### Behaviours

The big shark switches between these on its own; the distant sharks only cruise and come
and go.

| Behaviour | What happens |
| --- | --- |
| **Cruise** | Picks somewhere to go and swims there. Mostly across to the other side of the view, sometimes a loop on the same side, sometimes a dive or a climb. Gets bored and picks somewhere else even if it has not arrived. |
| **Leave and return** | Swims straight on (never a U-turn) until it is wholly out of sight. The camera does not chase it; it drifts over to the school and waits. After a while the shark comes back: from where it left, as if it had turned round out there, from the other side, or rising out of the blue far off. |
| **Close pass** | Swims right up to the lens, close enough to fill the picture, then veers off and goes back out. The camera holds still in depth while it does. |
| **Charge** | Rushes at the school at full effort. The fish scatter; the shark misses on purpose, because this is not the rare feed. |
| **Feed** | See [the rare feed](#the-rare-feed). |

The camera follows the big shark loosely (it keeps it roughly in the picture, a moment
late) and slowly swings and tilts on its own.

### The school of fish

A classic flocking simulation (boids) with a few extra pulls:

- **Room:** each fish keeps a little space from its nearest neighbours.
- **Heading:** it matches the average heading of the fish around it.
- **Closeness:** it drifts towards the middle of its neighbours.
- **Home:** the whole school follows a home point that wanders slowly round the view.
- **Fear:** any shark's snout within reach pushes fish away hard, and the more frightened a
  fish is, the faster it may swim. Frightened fish also glint brighter.

### The rare feed

At most **six times in any 24 hours**, the big shark hunts for real.

1. The fish that has strayed furthest from the middle of the school is singled out.
2. It runs for its life: straight away from the snout, with sudden jinks to the side in
   random directions. It cannot hide outside the picture. It tires; the shark does not.
3. The shark sprints at it and aims **where the fish is going**, not where it is (it leads
   its target by an amount that depends on distance, with a little randomness). It turns
   much harder than usual.
4. As it closes in, the **jaw opens**: the underside of the snout drops.
5. **The bite:** the fish is gone. The shark shakes its head. Scales glint and sink, a faint
   dark cloud spreads and fades, and a few bubbles rise.
6. The school panics and regroups. A while later a newcomer swims in from the edge of the
   view to take the lost fish's place.

**How the limit works.** Every feed is timestamped in the browser's `localStorage`
(`ascii-shark-feeds`). Before a feed starts, the timestamps from the last 24 hours are
counted; if there are already six, the next attempt is pushed back to when the oldest one
expires. The planned time of the next feed is stored too (`ascii-shark-next-feed`), so
reloading the page does not bring it forward. Feeds are spread out at random, on average a
few hours apart and never less than an hour, so most visits will not see one at all.

The limit is per browser, because it is kept in that browser's storage. Where storage is
unavailable (a `file://` page in some browsers, private modes that block it), the page
still runs; it simply cannot remember across reloads.

### Why it cannot be predicted

- **Cryptographic randomness.** Every random choice is drawn from
  `crypto.getRandomValues`, not `Math.random`.
- **Memoryless waits.** The time until the next behaviour (leaving, a close pass, a charge,
  changing its mind, how long it stays away, when the next feed comes) is drawn from an
  **exponential distribution**. However long you have watched, the next event is no closer,
  so there is no rhythm to learn.
- **A temperament of its own.** On every load each shark draws its own cruising effort, how
  readily it turns and how fast its tail beats.
- **Slow wanderings.** A heading that strays and an effort that swells and fades, both
  driven by smooth random noise, so even a straight run is never quite straight.
- **Mixed choices.** Where it heads next, how it returns after leaving, and where it comes
  up to the lens are each drawn from several different options.

### Water, light and glow

- **Marine snow:** specks spread through a large box around the camera that sink very
  slowly, sway, and wrap round as the camera moves. Near ones are drawn as `+`, far ones as
  `.` or `'`, and they fade with distance.
- **Sun beams:** a fan of beams from a point above the screen, each swaying and pulsing at
  its own speed. Anything inside a beam is brightened; sparse `:` dust hangs in empty water
  inside the beams.
- **The surface:** a band of `~` at the top of the screen, lit by two crossing waves.
- **Fog:** everything fades with distance into a deep-blue gradient.

---

## Performance

- The frame rate is capped at **30 fps**, which is plenty for slow, drifting motion and
  keeps the page light enough to leave open all day.
- The physics runs at a fixed 60 steps a second, independent of the frame rate, so a slow
  machine shows the same motion with fewer frames.
- On a laptop, a typical frame takes about **8–12 ms** of work, and about 20 ms when the
  shark is right in front of the lens.
- `requestAnimationFrame` stops when the tab is hidden, so a background tab costs nothing.
  When it comes back, the simulation resumes without trying to catch up on lost time.
- The device-pixel ratio is capped at 2 to keep very large screens fast.
- Nothing grows over time: point buffers are reused, particles are removed when they fade,
  and the school keeps a fixed size.

---

## Tuning

The constants near the top of the script are the main knobs:

| Constant | Meaning |
| --- | --- |
| `FPS_CAP` | Frames drawn per second (default 30). |
| `STEP` | Physics time step (default 1/60 s). |
| `CAM_D` | Distance from the camera to the middle of the scene. |
| `THRUST`, `DRAG` | How much a tail beat pushes and how hard the water holds back. |
| `PALETTES` | Back, belly and fin colours of the sharks. |
| `MAX_FEEDS` | Feeds allowed in any 24 hours (default 6). |
| `RAMP_CODES[SKIN]` | The characters used for the skin, from dim to bright. |

The waits between behaviours are the `expo(mean, min, max)` calls inside the `Shark` class,
in seconds. The gap between feeds is set in `planFeed()`, in hours.

In `layout()`, `FS` sets the character size relative to the screen and `SCALE` sets how big
the scene is drawn.

---

## Debugging from the console

The page exposes a small object for poking at it in the browser console:

```js
__sea.fps        // frames per second
__sea.ms         // work per frame, in milliseconds (smoothed)
__sea.chars      // characters drawn in the last frame
__sea.peek()     // positions, targets and states of the sharks and the school
__sea.feeds()    // how many feeds have happened in this browser in the last 24 hours
__sea.feed()     // start the rare feed now (still counts towards, and obeys, the limit)
__sea.all()      // the Shark objects themselves
```

For example, to watch a close pass right away:

```js
__sea.all()[0].nextClose = 0
```

---

## Browser support

Any current browser with Canvas 2D and `crypto.getRandomValues`: Chrome, Edge, Firefox and
Safari, on desktop and mobile. The page uses no WebGL, no workers and no web fonts. It
picks the first available monospace font from Menlo, Consolas, DejaVu Sans Mono, Liberation
Mono and Courier New.

---

## Inspiration

The idea of drawing a procedural 3D scene as coloured ASCII with a glow pass comes from
[ascii-lab-public](https://github.com/sapozhkov/ascii-lab-public) by sapozhkov, whose
`clownfish.py` renders a clownfish and an anemone to video. This project is a separate,
from-scratch implementation for the browser, with its own models, behaviours and renderer.
