# Small synthetic tiled-collision fixture

This fixture accompanies [splat-transform #344](https://github.com/playcanvas/splat-transform/issues/344) and [supersplat-viewer #327](https://github.com/playcanvas/supersplat-viewer/issues/327). It contains **2,220 generated Gaussians** forming a colored floor and a low wall, with no source assets from the public Dayun or Bijiashan scenes.

The collision data uses ordinary **PlayCanvas world coordinates, Y up**, and voxel format **1.1**. No legacy `rotation` field, MetaFlow coordinate adapter, or additional model rotation is required. The PLY is written through the standard splat-transform PLY writer and loads with the viewer's ordinary PLY coordinate handling.

## Contents

- `scene.ply`: the small synthetic Gaussian scene.
- `scene.voxel-tiles.json` and `scene-tiles/`: six populated collision tiles, each an ordinary `.voxel.json` / `.voxel.bin` pair.
- `single.voxel.json` / `single.voxel.bin`: the same scene generated as a single collision asset for comparison.
- `settings.json`: initial camera and display settings for the example.

Generation uses 0.1 m voxels, opacity threshold 0.1, 4 m XZ tile width and 0.4 m requested overlap. Tile boundaries are aligned to four-voxel blocks; use each entry's `dataBounds` as the actual stored bounds. This is surface collision only, without global exterior fill, floor fill, navigation carving, or collision mesh output.

## Regenerate with Node 24

Activate **Node.js 24** before running these commands. Voxel generation requires a working WebGPU device through the CLI's existing GPU backend. Start in a directory where `splat-transform-tiles` does not already exist:

```sh
node --version
# Expected: v24.x.x

git clone --single-branch --branch feat/voxel-tiles \
  https://github.com/Shuang-su/splat-transform.git splat-transform-tiles
cd splat-transform-tiles
npm ci
npm run build
mkdir -p generated

node bin/cli.mjs generators/gen-voxel-tiles.mjs generated/scene.ply

node bin/cli.mjs generators/gen-voxel-tiles.mjs generated/scene.voxel-tiles.json \
  --voxel-size 0.1 --voxel-opacity 0.1 \
  --voxel-tile-size 4 --voxel-tile-overlap 0.4

node bin/cli.mjs generators/gen-voxel-tiles.mjs generated/single.voxel.json \
  --voxel-size 0.1 --voxel-opacity 0.1
```

The generator is `generators/gen-voxel-tiles.mjs` on the feature branch. The included `settings.json` is a presentation input, not generated collision data; reuse it for the same camera view. Use a fresh output directory for each export, because replacing an existing multi-file output is not transactional. Regeneration is intended to reproduce the scene and collision behavior, not guarantee identical GPU bytes on every adapter.

For the dedicated tests, from that checkout:

```sh
node --import tsx --test --test-force-exit test/voxel-tiles.test.mjs
TEST_WEBGPU=1 node --import tsx --test --test-force-exit test/voxel-tiles.test.mjs
```

The first command runs CPU validation and budget tests; the second also runs the actual GPU generation/occupancy comparison tests.

## Preview using the viewer feature branch

The pre-generated assets can be previewed without running the generator. In a new working directory, use Node 24 and run:

```sh
git clone --single-branch --branch docs/voxel-tiles-evidence \
  https://github.com/Shuang-su/supersplat-viewer.git voxel-tiles-evidence

git clone --single-branch --branch feat/tiled-voxel-collision \
  https://github.com/Shuang-su/supersplat-viewer.git supersplat-viewer-tiles

cp -R voxel-tiles-evidence/repro supersplat-viewer-tiles/public/repro
cd supersplat-viewer-tiles
npm ci
npm run build
npm run serve -- --listen 8080
```

Then open one of these local URLs:

- [Tiled collision](http://localhost:8080/?content=./repro/scene.ply&settings=./repro/settings.json&collision=./repro/scene.voxel-tiles.json)
- [Single collision comparison](http://localhost:8080/?content=./repro/scene.ply&settings=./repro/settings.json&collision=./repro/single.voxel.json)
- [Tiled collision with heatmap](http://localhost:8080/?content=./repro/scene.ply&settings=./repro/settings.json&collision=./repro/scene.voxel-tiles.json&heatmap)
- [Tiled collision with WebGL](http://localhost:8080/?content=./repro/scene.ply&settings=./repro/settings.json&collision=./repro/scene.voxel-tiles.json&webgl)

The viewer initially animates the camera. Pause with **Space**, switch to **Fly**, then select **Walk** when collision coverage is ready. Use **W/A/S/D** to move and **Esc** to exit Walk. On WebGPU, enable **Show Collision** in the settings panel to display the overlay (or the heatmap when its query flag is present). WebGL supports collision queries but does not offer the compute-based voxel overlay.

This small example exercises six tiles, so it is a convenient seam/alignment and format-compatibility fixture. It is not a large-scene memory benchmark or an exhaustive streaming stress test. The feature tests cover delayed requests, failed-load retry, stale results, missing coverage and resource lifetime separately.
