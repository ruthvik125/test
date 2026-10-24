# Cardboard Room Web VR Demo

This document explains the structure and runtime behavior of `index.html`, a browser-based Google Cardboard-style VR demo for Android.

## Overview

The demo renders an interactive 3D room with Three.js. It supports:

- Normal single-camera viewing
- Side-by-side stereo rendering for Cardboard viewers
- Phone gyroscope/device-orientation head tracking
- Gaze-based object selection in VR mode
- Pointer/touch selection in normal mode
- Fullscreen and landscape orientation handling
- Enter and exit VR viewing controls

## Page structure

The HTML contains three main visual areas:

- `#stage`: fixed container where Three.js inserts its WebGL canvas
- `#panel`: normal-mode instructions and control buttons
- `#hud`: overlays such as the reticle, status text, help text, and exit button

The viewport meta tag disables browser scaling and allows the page to use the device viewport safely on mobile screens.

The CSS makes the page fill the entire browser window and prevents scrolling. The canvas is set to occupy the stage, while the HUD is positioned above it.

## Three.js initialization

The script imports Three.js from jsDelivr and creates:

```js
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(...);
const renderer = new THREE.WebGLRenderer(...);
```

The main camera is used for normal viewing and for storing the user's current head orientation. Two additional cameras, `leftCamera` and `rightCamera`, are used for stereo VR rendering.

The renderer uses a capped pixel ratio:

```js
renderer.setPixelRatio(Math.min(devicePixelRatio || 1, 1.5));
```

The cap reduces GPU load on mobile devices while retaining sharper output than a pixel ratio of `1`.

## Scene construction

The room is assembled from box meshes using the helper function:

```js
box(parent, name, size, position, material, extra)
```

This creates floors, walls, a ceiling, a window, a desk, a monitor, a chair, a plant, and decorative floor strips.

Lighting is provided by:

- A hemisphere light for general ambient illumination
- A directional light for the main room lighting
- A point light near the lamp area for colored local illumination

The room is intentionally open at the front so the viewer can see the interior from the initial camera position.

## Interactive objects

Interactive objects are created through the `target()` helper. Each target receives:

- A Three.js geometry and emissive material
- A name and description
- A `userData.target` flag
- A base vertical position
- A glowing torus halo

The objects are stored in the `targets` array. During animation, they float slightly and rotate. When selected, their emissive intensity is temporarily increased and the status message displays their description.

## Normal viewing mode

When VR mode is disabled, the animation loop renders the scene once using the main camera:

```js
renderer.render(scene, camera);
```

On desktop and mobile, pointer dragging changes the camera's yaw and pitch. A pointer tap performs a raycast and selects an object under the tap location.

## Entering VR mode

The `enterVR()` function performs the following sequence:

1. Requests device-orientation permission when the browser requires it.
2. Starts listening for `deviceorientation` events.
3. Requests fullscreen mode.
4. Attempts to lock the screen to landscape orientation.
5. Waits briefly for mobile browser dimensions to settle.
6. Resizes the renderer.
7. Hides the normal control panel.
8. Shows the VR exit button and gaze reticle.
9. Sets `vrMode` to `true` so the animation loop uses stereo rendering.

Fullscreen and orientation APIs are best-effort because browser support and permission rules vary between Android browsers.

## Device orientation and head tracking

`startSensors()` attaches `onDeviceOrientation()` to the window.

The orientation handler:

1. Reads `alpha`, `beta`, and `gamma` from the phone.
2. Converts the values from degrees to radians.
3. Builds a Three.js Euler rotation using the `YXZ` order.
4. Applies a correction quaternion so the phone orientation maps correctly to the camera.
5. Applies the current screen orientation angle.
6. Stores the first orientation as the starting reference.
7. Calculates the phone's orientation relative to that starting reference.
8. Applies the result to the camera quaternion.

The recenter button clears the starting orientation, causing the next sensor reading to become the new forward direction.

## Gaze interaction

`updateGaze()` casts a ray from the center of the camera view:

```js
raycaster.setFromCamera(center, camera);
```

When the ray intersects an interactive target, a dwell timer starts. After `DWELL_MS` milliseconds, currently 1,400 ms, the object is selected.

In VR mode, the reticle becomes visible and its circular progress indicator shows dwell progress. The ray is based on the main camera orientation, which is also used as the source for both stereo eye cameras.

## Stereo rendering

The `renderStereo()` function renders two views:

1. Copies the main camera into the left and right cameras.
2. Sets a slightly different horizontal eye position for each camera.
3. Updates each camera's aspect ratio.
4. Enables scissor testing.
5. Renders the left eye into the left half of the canvas.
6. Renders the right eye into the right half of the canvas.
7. Disables scissor testing.

The two views are intended to be placed behind the left and right lenses of a Cardboard viewer.

### Important mobile sizing detail

The renderer has both a CSS size and a WebGL drawing-buffer size. These can differ when `devicePixelRatio` is greater than `1`.

Viewport and scissor coordinates should use the drawing-buffer dimensions. The robust implementation is:

```js
renderer.getDrawingBufferSize(canvasSize);

const w = canvasSize.x;
const h = canvasSize.y;
const half = Math.floor(w / 2);
```

Then use `w`, `h`, and `half` for `setViewport()` and `setScissor()`.

Using CSS dimensions from `renderer.getSize()` for WebGL viewport coordinates can cause the stereo image to occupy only part of the physical canvas on high-density mobile screens. This commonly appears as a half-screen or incorrectly sized landscape VR view.

## Resize handling

The `resize()` function reads the actual `#stage` dimensions, resizes the renderer, and updates the main camera aspect ratio. It is called from:

- The window `resize` event
- The `orientationchange` event
- The `fullscreenchange` event
- A `ResizeObserver` watching the stage
- Additional delayed callbacks after entering or exiting VR

The delayed resize calls are important because Android browsers may update fullscreen and orientation dimensions asynchronously.

## Animation loop

`animate()` runs continuously with `requestAnimationFrame()` and performs three jobs:

1. Animates the floating interactive objects and their halos.
2. Updates gaze targeting and reticle progress.
3. Renders either the normal camera view or stereo VR view.

The selected state is time-limited using `selectedUntil`, after which the object's emissive intensity returns to its normal value.

## Exiting VR mode

`exitVR()`:

- Disables VR mode
- Removes the orientation listener
- Clears gaze state
- Hides the exit button and reticle
- Restores the normal panel and help text
- Exits fullscreen when active
- Unlocks screen orientation
- Resizes the renderer again

The page also exits VR automatically when it becomes hidden through the `visibilitychange` handler.

## Recommended stereo rendering fix

Replace the beginning of `renderStereo()` with:

```js
function renderStereo() {
  renderer.getDrawingBufferSize(canvasSize);

  const w = canvasSize.x;
  const h = canvasSize.y;
  const half = Math.floor(w / 2);

  // Continue with the existing camera setup and viewport rendering.
}
```

The rest of the function can continue using `w`, `h`, and `half` for the viewport and scissor rectangles.

## Browser considerations

Behavior depends on the Android browser:

- Device orientation events may require a secure HTTPS origin.
- Some browsers require a user gesture before motion permission can be requested.
- Fullscreen and orientation locking may be rejected or ignored.
- Browser UI bars can change the visible viewport after rotation.
- Sensor readings may be unavailable if motion sensors are disabled at the OS or browser level.

The demo reports a warning if no orientation readings arrive shortly after entering VR. Touch-look remains available in normal mode.
