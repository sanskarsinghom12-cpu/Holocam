# HoloAR Cam

A lightweight futuristic AR camera app for Android: place glowing holographic
objects that stay fixed in the real world, make a heart shape with both hands
to trigger a neon heart effect, and get a short positive compliment based on
real-time face analysis (nothing is ever saved).

## What's included

This is a real, buildable Android Studio project (Kotlin + Jetpack Compose),
not a mockup:

```
app/src/main/java/com/holoar/cam/
├── MainActivity.kt              # permissions, lifecycle, Compose HUD wiring
├── ar/
│   ├── ArManager.kt             # ARCore session, hit-testing, object list, gestures
│   ├── BackgroundRenderer.kt    # draws the live camera feed (GL)
│   ├── HoloObject.kt            # data model for a placed object (anchor + shape + scale + rotation)
│   ├── HoloObjectRenderer.kt    # draws neon wireframe/glow cubes (GL)
│   ├── HoloSceneRenderer.kt     # ties rendering + throttled ML analysis together
│   ├── HoloGLSurfaceView.kt     # tap/drag/pinch/twist touch handling
│   ├── ImageUtils.kt            # YUV → Bitmap conversion for analysis frames only
│   └── GlUtil.kt                # shader helper
├── gesture/
│   ├── HandTracker.kt           # MediaPipe HandLandmarker wrapper
│   └── HeartGestureDetector.kt  # geometric two-hand heart-shape heuristic
├── face/
│   └── ComplimentEngine.kt      # ML Kit face analysis → compliment text (no storage)
└── ui/
    ├── HudOverlay.kt            # shape picker, delete button, compliment bubble, heart overlay
    └── theme/NeonTheme.kt       # neon cyan/blue Compose theme
```

## How it fits together

ARCore owns the camera and drives the whole pipeline — there's no separate
CameraX preview competing for the sensor. Every frame:

1. `ArManager`/`HoloSceneRenderer` calls `session.update()` and draws the
   camera passthrough + any placed holographic objects.
2. Every ~6th frame (roughly 5×/second, not every frame), it grabs the camera
   image, converts it off the GL thread, and runs it through the hand tracker
   and face analyzer. This keeps the expensive ML work from ever touching the
   smooth 30fps render loop.
3. Hand landmarks feed a **geometric heuristic** (thumb tips together, index
   tips together and above the thumbs, other fingers curled, hands mirrored)
   rather than a trained classifier — it's cheap, explainable, and tunable in
   `HeartGestureDetector.kt` if it's too strict/loose for how people actually
   pose.
4. Face analysis only ever looks at smile/eye-open probabilities from ML Kit
   to pick a compliment string from a small pool — the bitmap and face object
   are local variables that go out of scope immediately after. Nothing is
   written to disk, uploaded, or cached.

Placed objects stay fixed in the world because they're attached to ARCore
`Anchor`s, which ARCore continuously re-solves against tracked visual
features as the phone moves — that's what gives the "stays put" behavior.

## Setup steps before this builds and runs

1. **Open in Android Studio** (Koala/2024.1+) and let it sync Gradle.
2. **Add the hand landmark model.** MediaPipe's HandLandmarker needs a
   `.task` model file bundled in `app/src/main/assets/hand_landmarker.task`.
   Download it from Google's MediaPipe model index (search "MediaPipe Hand
   Landmarker task file") and drop it in that assets folder — it's a few MB
   and isn't something to vendor into source control blindly, so it's left
   out here.
3. **Add a real launcher icon** if you want something nicer than the simple
   neon-hexagon vector icon included here.
4. **Run on a physical ARCore-supported device.** ARCore needs real camera
   motion to track anchors; the emulator won't give you a usable experience.
5. First run will prompt to install/update **Google Play Services for AR**
   if it isn't already on the device — that's expected, ARCore handles it
   via `ArCoreApk.requestInstall`.

## Known simplifications (call these out if you hand this to someone else)

- The "glow" on holographic objects is a cheap two-pass additive-blend trick
  (faint fill + bright wireframe + a sine pulse), not a bloom post-process
  pipeline — it looks good and costs almost nothing, but a true bloom shader
  would look even better if you have GPU budget to spare.
- The heart-shape detector is a hand-tuned geometric heuristic. It works well
  for the classic "finger heart" pose but may need threshold tweaks
  (`HeartGestureDetector.kt`) depending on hand size/distance from camera in
  your testing.
- Object "move" is implemented as re-anchoring to a new hit-test point under
  the finger each drag frame, which is simple and robust but not perfectly
  smooth on sparse-feature surfaces (e.g. blank walls) — ARCore just won't
  find many hit-test points there.
- MediaPipe/ML Kit API surfaces shift between versions; double-check the
  exact method names in `HandTracker.kt` and `ComplimentEngine.kt` against
  whatever `tasks-vision`/`face-detection` versions Gradle actually resolves,
  since this was written without the ability to compile against them live.

## Privacy

Camera frames used for gesture/face analysis are processed entirely
on-device and discarded immediately after each analysis pass. No image,
landmark, or face data is written to disk, logged, or sent anywhere.
