# WebGLRenderer + WebGLNodesHandler: NodeMaterials render with the same camera for both eyes in WebXR

Minimal reproduction for three.js (r184 NodeMaterial compatibility layer, [PR #32851](https://github.com/mrdoob/three.js/pull/32851)).

**Live examples** (open on a headset, press ENTER VR):

- https://tomingtoming.github.io/three-nodes-xr-repro/ — three@0.187.0 (jsdelivr)
- https://tomingtoming.github.io/three-nodes-xr-repro/r186.html — three@0.186.0 (jsdelivr)
- https://tomingtoming.github.io/three-nodes-xr-repro/dev.html — three dev build (raw.githack mrdoob/three.js/dev)
- append `?patch=1` to any of them for the workaround

## Description

With `WebGLRenderer` + `renderer.setNodesHandler(new WebGLNodesHandler())`, a `MeshBasicNodeMaterial` sphere (blue) is drawn
from the **same camera for both eyes** during an immersive-vr session, so it has no stereo disparity. A classic
`MeshBasicMaterial` sphere (red) at the same depth is drawn with the per-eye cameras.

## Reproduction steps

1. Open a live example on a WebXR headset and enter VR.
2. Compare the two spheres, both 2 m ahead: red uses `MeshBasicMaterial`, blue uses `MeshBasicNodeMaterial`. Blue looks mismatched between the eyes.
3. Reload with `?patch=1`: blue looks stereoscopic.

## Root cause

1. `WebGLRenderer.render()` calls `_nodesHandler.renderStart( scene, camera )`
   ([r187 L1655](https://github.com/mrdoob/three.js/blob/r187/src/renderers/WebGLRenderer.js#L1655)) **before** it replaces `camera` with
   `xr.getCamera()` ([r187 L1677](https://github.com/mrdoob/three.js/blob/r187/src/renderers/WebGLRenderer.js#L1677)), so `nodeFrame.camera`
   stays the user camera.
2. `WebGLNodesHandler.onBeforeRenderCallback` receives the per-eye camera from the renderer but does not use it
   ([r187 L315](https://github.com/mrdoob/three.js/blob/r187/examples/jsm/tsl/WebGLNodesHandler.js#L315)).

Camera-dependent node uniforms (`cameraViewMatrix`, `cameraProjectionMatrix`, `cameraPosition`, …) are therefore taken from one camera for
both eyes. Which camera that is depends on the version: in r186 the user camera received the union of both eye frusta; since r187
([PR #34275](https://github.com/mrdoob/three.js/pull/34275)) the XR camera follows the first sub camera, so the right eye shows the left eye's view.

## Workaround (app side)

```js
const handler = new WebGLNodesHandler();
renderer.setNodesHandler(handler);
const orig = handler.onBeforeRenderCallback;
handler.onBeforeRenderCallback = function (r, scene, camera, geometry, object) {
  if (camera && camera.isCamera && !camera.isArrayCamera) handler.nodeFrame.camera = camera; // per-eye camera
  return orig.call(this, r, scene, camera, geometry, object);
};
```

A fix might set `nodeFrame.camera` in `onBeforeRenderCallback`. Calling `renderStart` after the XR camera swap would not be enough, because
`WebGLNodesHandler` does not update `cameraIndex` for an `ArrayCamera`.

## Measurements (emulator)

Headless Chrome 155 + [IWER](https://github.com/meta-quest/immersive-web-emulation-runtime) 2.8.0 (Meta Quest 3 profile, stereo), IPD
exaggerated to 0.6 m so the disparity is measurable in pixels, framebuffer 1024×768 (512 px per eye), 30th frame, 2026-11-06.
Disparity = right-eye centroid x − left-eye centroid x (px).

| build | red (classic) | blue (NodeMaterial) | blue with `?patch=1` |
|---|---|---|---|
| r186 (0.186.0) | −117.9 | **0.0** | −117.9 |
| r187 (0.187.0) | −117.9 | **0.0** | −117.9 |
| dev (2026-11-06) | −117.9 | **0.0** | −117.9 |

With the patch, blue matches red to 1e-13 px. The unpatched blue centroid sits where the single camera projects it: the union camera in r186,
the left eye in r187 and dev.

An earlier run (2026-09-18, different framebuffer size) gave red −98.4 / blue 0.0 / patched −98.3 on r185, r186 and r187dev. The screenshots
below are from that run (r186).

![r186 unpatched](r186-unpatched.png)
![r186 patched](r186-patched.png)

## Environment

- Hardware: Meta Quest 3, Quest Browser 152.1.0.34.52 (Chrome 152.0.7977.83). r186 page (2026-10-30, 11-05) and dev page, r188dev
  (2026-11-09): blue looks wrong in VR, `?patch=1` fixes it. Human observation, not measured.
- Earlier: a NodeMaterial-based scene inside an XRift world lost depth on a Quest headset and was fixed by the workaround (2026-09-18).
- three.js: r185, r186, r187, dev. Renderer: WebGLRenderer + WebGLNodesHandler.
