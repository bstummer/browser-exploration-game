# Cloudlight

A quiet, infinite exploration game about a lone traveller drifting between floating islands above a
sea of clouds. Walk, leap off edges, glide, ride spiralling updrafts, and light the old stone beacons
one by one, turning the places you have explored into a constellation of light.

Everything ships in one self-contained file, **`index.html`**. three.js (r169, pinned) is loaded from
jsDelivr. Every model, texture and sound is generated in code.

## Play

Open `index.html` in a desktop browser, or serve the folder (`npx serve .`) and visit it. Click to begin.

| Input | Action |
| --- | --- |
| `W` `A` `S` `D` | walk (relative to the camera) |
| Mouse | look; while gliding the glider turns toward where you look |
| `Shift` | sprint |
| `Space` | jump; in the air, open or close the glider |
| `W` / `S` while gliding | dive for speed / pull up (a brief climb, then a stall) |
| `A` / `D` while gliding | steer (camera and traveller bank into turns) |
| `E` | light a dormant beacon you are standing beside |
| Mouse wheel | camera distance |
| `Esc` | pause: seed, copy link, new world, quality, volume, mouse sensitivity |
| `F3` | performance overlay (FPS, frame time, draw calls, triangles, chunks, instances) |
| `H` | hide the HUD |

**Worlds and seeds.** A world comes from the seed in the URL (`index.html#seed=48213`). The same
seed always produces the same world, and lit beacons are remembered per seed in `localStorage`.
If storage is blocked the game still works, it just forgets. Editing the hash loads that world.

Falling into the clouds is never a punishment. The screen fades to white and you wake on the last
island you stood on.

## How it works

- **Streaming.** The world is split into 256-unit chunks, generated deterministically from the seed
  and the chunk coordinates.
  - Generation runs as generator jobs under a per-frame time budget (about 3 ms). Jobs are chosen by
    distance, biased toward the direction of flight.
  - Distant chunks use low-detail terrain and simplified props. Nearby chunks upgrade to full detail
    with shadow-casting props, flowers and grass.
  - Unloading disposes geometry, returns typed arrays to a pool and releases instance blocks.
- **Islands.** Each island top is a polar heightfield with a soft edge falloff, strata bands on the
  sides, and a jagged underside cone with hanging roots.
  - Ground collision looks up the *same triangles* the mesh is built from; there are no raycasts.
  - Island spacing is resolved against neighbouring chunks by priority, so chunk borders are seamless.
- **Biomes** follow altitude. Low islands are lush meadow, forest and blossom. The middle band holds
  autumn woods, ruins and mesa. The highest islands are snow, alpine and ice. A second noise varies
  the biome within each band.
- **Reachability.** Every island is checked against its neighbours for a *way down* (a lower island
  within a conservative 6:1 glide) and a *way in* (a higher island that can glide to it).
  - Islands missing either get an updraft beside their rim. Entrance updrafts rise from just above
    the cloud sea, so they can be caught from almost anywhere.
  - Where a column would be blocked by an island below, it becomes a wind vent on that island.
  - A graph search over 17×17 chunks (`__cloudlight.testReachability()`) confirms that every island
    is reachable from the start island, even at a pessimistic 6:1 glide ratio.
- **Glider.** Drag is parasitic plus induced (`A·s² + B/s²`), under the same gravity as falling.
  Total energy can only decrease, so without an updraft you cannot climb sustainably. The neutral
  glide is 9:1. `__cloudlight.testGlide()` tries several piloting strategies to break this.
- **Rendering.**
  - One merged, flat-shaded, vertex-coloured mesh per chunk. Every prop type is a pair of shared
    `InstancedMesh`es. There is one directional light plus a hemisphere light.
  - The shadow map is tight, follows the traveller and is skipped on Low; a blob shadow is always on.
  - Fog is computed per pixel from the same function as the sky gradient, using radial distance, so
    islands dissolve exactly into the sky behind them. The far plane sits just past the fog.
  - Lit beacon beams and distant landmarks are drawn toward the camera in the vertex shader, so they
    stay visible beyond the fog.
  - Updrafts, waterfalls, clouds and sparks are animated on the GPU.
- **Quality.** Low, Medium, High, or Auto (default). Auto adjusts render resolution and view
  distance from measured frame times. The pixel ratio is clamped to 1.5.

## Debug hooks

`window.__cloudlight` exposes `stats()`, `testReachability(R, glideRatio)`, `testSpacing()`,
`testDeterminism()`, `testGlide()`, `testUpdraft()`, `profileGen()`, `teleport(x, y, z)` and
`setTod(t)`.
