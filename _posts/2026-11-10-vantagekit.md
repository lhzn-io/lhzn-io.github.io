---
title: "Standardizing 3D Viewer Navigation with vantagekit"
---

# Standardizing 3D Viewer Navigation with vantagekit

We maintain several small viewers for terrain and ocean data: a 3D view of a seafloor elevation model, a preview of the wind and current fields that force an ocean model, a map that tracks our research boat, and others. Each viewer started with its own camera code, and no two behaved alike. Three defects were typical.

- The mouse wheel zoomed by a fixed fraction of the remaining distance, so zooming stalled as the camera approached the orbit point.
- Keyboard movement used scene units. A scene drawn at 100 m per unit therefore moved a hundred times faster than a 1 m per unit scene.
- The keyboard handler tracked characters, not physical keys. Releasing Shift before W left W held and the camera drifting.

We wrote [`vantagekit`](https://github.com/lhzn-io/vantagekit) to replace that code, and we are releasing it as open source under the MIT license, since any group building viewers for Earth-system data meets the same problems. Three renderers are supported today (three.js, MapLibre GL JS and Omniverse Kit), and contributions are welcome: the last section of this post describes where help would go furthest. The **[live demo](https://lhzn-io.github.io/vantagekit/demo/)** opens on Montauk Point, Long Island: a 9.1 by 6.0 km topobathymetric grid at 3.8 m, fused with [topobathykit](https://github.com/lhzn-io/topobathykit) from NOAA topobathy lidar, BlueTopo, hydrographic survey grids and CUDEM. [data.lhzn.io](https://data.lhzn.io) serves the 2.7 MB grid. The demo also offers an invented coastline that needs no data, and it accepts mouse, keyboard, gamepad and finger input.

![The vantagekit demo on a desktop in orbit mode: Montauk Point at 5x vertical exaggeration, seen from offshore in the south-east, with the bluffs, Oyster Pond and Shagwong Point; the settings panel with the data sources on the left, and the Orbit and Fly switch with a one-line hint at top right](/assets/images/vantagekit/demo-desktop.png)

*Figure 1. Desktop browser, orbit mode, Montauk Point. The switch at top right highlights the active mode; the panel lists the data sources, the bindings, and the current range, bearing and pitch.*

## Two camera modes

Terrain and ocean viewers need two camera models.

**Orbit.** The camera sits on a sphere centered on a target, the orbit point. Dragging rotates the camera; the wheel or a pinch changes the sphere's radius; a right-button or two-finger drag translates the orbit point. Orbit suits surveying an area, inspecting a feature from several sides, and framing a view to share.

**Fly.** The camera has a position and a look direction, and no target. The mouse sets the look direction, WASD translates the camera, and the wheel sets the speed. Climbing and diving work by pitching the view and moving forward. Fly suits following a coastline, crossing a channel at low altitude, and tracing a feature end to end.

Each page chooses its modes: both (F toggles, Esc exits fly), orbit only, or fly only. A mode change preserves the view. Entering fly starts from the orbit camera's eye; leaving fly sets the new orbit point straight ahead and keeps the eye distance.

## Units

Viewers use different scene units: one unit may be 1 m or 100 m. All vantagekit motion uses meters and seconds.

- **Orbit keys** translate the orbit point at half the view distance per second, so zoom level does not change the response. At a 2 km range, that is 1 km per second.
- **Fly speed** is half the camera's height above the surface per second, when the page supplies a surface height function. Low passes are slow and high passes fast; the wheel scales the result.
- **Vertical exaggeration** does not affect motion. Most of our scenes exaggerate the vertical five times or more so that seabed relief is visible; speeds and camera links still use true meters.
- **Frame rate** has no effect: motion integrates elapsed time. Held keys count only while the page has focus, which fixed the drifting camera.

## Zoom past the orbit point

A zoom step that moves a fixed fraction of the remaining distance shrinks as that distance shrinks. vantagekit builds on [camera-controls](https://github.com/yomotsu/camera-controls), an orbit library for three.js, and enables two of its options. Zoom targets the cursor position, and at the minimum distance the orbit point advances ahead of the camera, so the camera never stops. Continued scrolling therefore keeps moving the camera forward.

## No ground collision

vantagekit has no ground collision. An early version had a floor, and we removed it. Viewing a seabed from beneath shows its shape without the water column or sky obscuring it. The demo lights the underside of the terrain so that this view stays readable. A page that needs a floor can add one; vantagekit does not impose it.

## View links

vantagekit writes the camera state into the page URL once the view settles. An example:

```
?cv=1&cam=-70.512345,42.498765,-35.0,1850.0,62.00,48.00
```

The six values are the orbit point's longitude, latitude and true height in meters; the camera distance in meters; the bearing (degrees, north = 0, clockwise positive); and the pitch (degrees, straight down = 0). Because the format is geographic, a different viewer of the same area can open the link, even one that uses another renderer.

## Phones

On a Pixel phone, fly mode worked as intended, but orbit pinch zoom responded weakly. camera-controls zooms by a fixed step per pixel of finger movement, measured from the fingers' midpoint, so a phone-sized pinch undershoots. Spreading two fingers from 100 to 300 pixels apart gave a zoom factor of 1.87 instead of 3. vantagekit now sets the zoom rate from the current finger separation, and the same spread gives 2.86.

Sound designer Zach proposed the optional on-screen joystick that fly mode now offers. One thumb moves the camera; any other finger sets the look direction.

![The demo on a phone held in landscape, in fly mode, low over the shoals east of Montauk Point and looking west at the point: the Orbit and Fly switch with Fly lit, a hint reading "Fly: the stick moves you, drag anywhere else to look, pinch for speed", and the joystick at bottom left with its knob pushed forward](/assets/images/vantagekit/demo-phone-landscape.png)

*Figure 2. Phone in landscape orientation, fly mode, joystick deflected forward.*

## Three renderers

The vantagekit core contains no renderer code. Geometry, the link format, input bindings and speed rules are plain functions, and a thin adapter per renderer connects them.

- **three.js.** The original adapter. The elevation viewer in [topobathykit](https://github.com/lhzn-io/topobathykit) and the 3D preview in [forcingkit](https://github.com/lhzn-io/forcingkit) both use it. Pages pass three.js and camera-controls to the adapter instead of the adapter importing them, so a page keeps a single copy of three.js.
- **MapLibre GL JS.** Used by map-first pages, including the map that tracks our boat. In orbit mode, MapLibre keeps its native mouse and finger handling; vantagekit adds keyboard, gamepad and fly mode. MapLibre limits camera tilt to 85 degrees, so fly mode cannot show a level view: the view stays at least 5 degrees nose-down. Movement still follows the aim direction; only the rendered view is clamped.
- **Omniverse Kit.** A Python extension for viewers built on NVIDIA Omniverse Kit and Isaac Sim, where the subject is usually a moving vessel, not terrain. It provides chase, point-of-view and orbit cameras around a moving target, supports Y-up and Z-up stages in any unit, and returns control to Kit's native navigation when released. Its Python core has no dependencies and is on PyPI (`pip install vantagekit`).

The Kit adapter exposed a frame-rate-dependent smoothing lag. The filter blended the camera toward the latest target position each frame, which treats a moving target as a series of steps. The lag behind the target therefore varied with frame time, and a chase camera trailing a boat at 15 m/s jumped about 8 cm each time the frame time changed. The filter now solves exactly for a target moving at constant velocity between frames, so the lag is always the target speed times the filter time constant.

## Open source, and contributions welcome

vantagekit came out of our own viewers, so its first features are the ones those viewers needed. Other teams have other renderers, controllers and scenes, and their changes would make the library broadly useful. [github.com/lhzn-io/vantagekit](https://github.com/lhzn-io/vantagekit) takes issues and pull requests; the README lists the tests and conventions. Useful places to start:

- adapters for other renderers, such as CesiumJS or deck.gl;
- fly-mode finger gestures for the MapLibre adapter, and climb and dive buttons beside the joystick (on a phone, vertical movement currently requires pitching the view and moving forward);
- mappings for other game controllers and input devices;
- more demo scenes: other coastlines and seabeds.

[github.com/lhzn-io](https://github.com/lhzn-io) hosts the rest of our open-source work, including [topobathykit](https://github.com/lhzn-io/topobathykit) (topobathymetric fusion), [forcingkit](https://github.com/lhzn-io/forcingkit) (model forcing) and [insitukit](https://github.com/lhzn-io/insitukit) (in situ observations).

- **Demos:** the [three.js demo](https://lhzn-io.github.io/vantagekit/demo/) and the [MapLibre demo](https://lhzn-io.github.io/vantagekit/demo/maplibre.html).
- **Source:** [github.com/lhzn-io/vantagekit](https://github.com/lhzn-io/vantagekit), MIT license. The JavaScript modules load from a CDN by version tag and need no build step, and pip installs the Python package.
- **Demo data:** [data.lhzn.io](https://data.lhzn.io) serves the Montauk grid and documents its sources and encoding. It hosts demo material only.
