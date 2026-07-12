# AirDrawAI

Draw glowing trails in the air using nothing but your hand and your phone's camera.
AirDrawAI uses **CameraX** + **Google MediaPipe Hand Landmarker** to track 21 hand
landmarks in real time, recognizes a small set of gestures, and lets you paint,
erase, pause, and resume — all without touching the screen.

---

## ✨ Features

- Live camera preview (CameraX) with front/back camera support
- Real-time hand tracking via MediaPipe Tasks Vision **Hand Landmarker** (21 landmarks)
- Jitter-free fingertip tracking using a **One Euro Filter**
- Gesture-controlled drawing:
  - ☝️ **Index finger up** → Draw
  - ✌️ **Two fingers up** → Eraser
  - ✊ **Closed fist** → Pause
  - 🖐️ **Open palm** → Resume
- Color picker (Red, Blue, Green, Yellow, White, Black)
- Adjustable brush size
- Undo / Redo
- Clear canvas (with confirmation)
- Save drawing as PNG to the gallery
- Flash/torch toggle (back camera)
- Material Design 3 UI, MVVM architecture, Kotlin

---

## 🏗 Architecture

```
MainActivity (View)
   │
   ├── DrawingViewModel (MVVM ViewModel)
   │      - selected color, brush size, camera facing, flash, gesture/status, undo/redo state
   │
   ├── CameraController        → CameraX Preview + ImageAnalysis lifecycle
   ├── HandLandmarkerHelper     → MediaPipe HandLandmarker (LIVE_STREAM mode)
   ├── GestureRecognizer        → classifies DRAW / ERASE / PAUSE / RESUME from landmarks
   ├── PointOneEuroFilter       → smooths the tracked fingertip position
   └── DrawingView (custom View) → owns stroke data, undo/redo stacks, rendering, PNG export
```

**Data flow:** CameraX delivers frames → converted to a `Bitmap` → fed into MediaPipe
`HandLandmarker` (async, LIVE_STREAM mode) → results delivered on a background thread →
posted to the main thread → `GestureRecognizer` classifies the pose → the index
fingertip is filtered and mapped from normalized image space into view space →
`DrawingView` draws/erases accordingly.

---

## 📁 Project structure

```
AirDrawAI/
├── build.gradle                          (project)
├── settings.gradle
├── gradle.properties
├── gradle/wrapper/gradle-wrapper.properties
└── app/
    ├── build.gradle                      (app module)
    ├── proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml
        ├── assets/
        │   └── hand_landmarker.task      (you add this — see Setup below)
        ├── java/com/toushif/airdrawai/
        │   ├── AirDrawApplication.kt
        │   ├── MainActivity.kt
        │   ├── camera/
        │   │   └── CameraController.kt
        │   ├── hand/
        │   │   └── HandLandmarkerHelper.kt
        │   ├── gesture/
        │   │   └── GestureRecognizer.kt
        │   ├── drawing/
        │   │   ├── DrawingView.kt
        │   │   └── DrawingViewModel.kt
        │   ├── model/
        │   │   ├── ColorOption.kt
        │   │   └── HandGesture.kt
        │   └── util/
        │       ├── OneEuroFilter.kt
        │       ├── CoordinateMapper.kt
        │       ├── ImageUtils.kt
        │       ├── ImageSaver.kt
        │       └── PermissionUtils.kt
        └── res/
            ├── layout/activity_main.xml
            ├── values/{colors,strings,themes,dimens}.xml
            ├── drawable/ (icons, backgrounds)
            ├── mipmap-*/ (launcher icons)
            └── xml/ (backup rules, file paths)
```

---

## ⚙️ Setup

### 1. Prerequisites

- Android Studio (Koala/2024.1 or newer recommended)
- JDK 17 (bundled with recent Android Studio)
- An Android device or emulator running **API 26+** with a working camera
  (physical device strongly recommended — emulator camera feeds are poor for hand tracking)

### 2. Download the MediaPipe Hand Landmarker model

The `.task` model file is a binary asset (~7-15 MB) and is **not** included in
this source bundle. Download it and place it in `app/src/main/assets/`:

```bash
curl -L -o app/src/main/assets/hand_landmarker.task \
  https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task
```

Or download manually from:
`https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task`

Then delete `app/src/main/assets/PLACE_MODEL_HERE.txt`.

> If you skip this step, the app will show a toast ("Failed to load hand tracking
> model...") but will not crash.

### 3. Open in Android Studio

1. `File → Open` → select the `AirDrawAI` folder.
2. Let Gradle sync. If the wrapper JAR is missing, Android Studio will offer to
   regenerate it automatically — accept the prompt. Alternatively, with Gradle
   installed locally, run `gradle wrapper --gradle-version 8.7` from the project root.
3. Wait for indexing/sync to finish.

### 4. Run

1. Connect a physical Android device (API 26+) with USB debugging enabled, or
   start an emulator with camera support.
2. Click **Run ▶** or use:
   ```bash
   ./gradlew installDebug
   ```
3. Grant the camera permission when prompted.
4. Hold your hand up in front of the camera, extend your index finger, and start drawing!

---

## 📦 Building a release APK

```bash
./gradlew assembleRelease
```

The unsigned APK will be output to `app/build/outputs/apk/release/app-release-unsigned.apk`.

To produce an installable **signed** APK:

1. Generate a keystore (skip if you already have one):
   ```bash
   keytool -genkey -v -keystore airdrawai-release.keystore \
     -alias airdrawai -keyalg RSA -keysize 2048 -validity 10000
   ```
2. Add signing config to `app/build.gradle` inside `android { }`:
   ```groovy
   signingConfigs {
       release {
           storeFile file("../airdrawai-release.keystore")
           storePassword "YOUR_STORE_PASSWORD"
           keyAlias "airdrawai"
           keyPassword "YOUR_KEY_PASSWORD"
       }
   }
   buildTypes {
       release {
           signingConfig signingConfigs.release
           // ...existing minifyEnabled / proguardFiles lines
       }
   }
   ```
3. Build:
   ```bash
   ./gradlew assembleRelease
   ```
4. Signed APK: `app/build/outputs/apk/release/app-release.apk`

For a Play Store-ready Android App Bundle instead:
```bash
./gradlew bundleRelease
```
Output: `app/build/outputs/bundle/release/app-release.aab`

---

## 🎮 How to use

| Gesture | Action |
|---|---|
| ☝️ Index finger extended only | Draw a stroke that follows your fingertip |
| ✌️ Index + middle fingers extended | Eraser — clears pixels along your fingertip path |
| ✊ Closed fist | Pause — lifts the "pen" without ending the session |
| 🖐️ Open palm (all fingers extended) | Resume drawing/erasing |

UI controls (bottom panel):
- **Color swatches** — tap to pick a brush color
- **Brush size slider** — drag to resize the brush/eraser
- **Undo / Redo** — step backward/forward through strokes
- **Clear** — wipes the canvas (confirmation dialog)
- **Save** — exports the current canvas as a PNG to `Pictures/AirDrawAI/`

Top bar:
- **Status pill** — shows current mode (Drawing / Erasing / Paused / Show your hand)
- **FPS counter**
- **Switch camera** — toggle front/back
- **Flash** — toggle torch (back camera only)

---

## 🔧 Performance notes

- `ImageAnalysis` is configured with `STRATEGY_KEEP_ONLY_LATEST` so the analyzer
  never backs up behind slow inference — old frames are dropped automatically.
- The Hand Landmarker runs in `LIVE_STREAM` mode with a GPU delegate by default,
  falling back to CPU automatically if the device has no usable GPU delegate.
- `DrawingView` caches completed strokes into an offscreen bitmap so `onDraw()`
  never has to replay the full stroke history — only the active stroke and one
  bitmap blit per frame.
- Target frame budget is 30–60 FPS depending on device; the FPS counter in the
  top bar reflects the actual analyzed-frame throughput.

---

## 🐛 Troubleshooting

| Problem | Fix |
|---|---|
| Toast: "Failed to load hand tracking model" | You haven't added `hand_landmarker.task` to `app/src/main/assets/` — see Setup step 2 |
| Hand tracked but drawing feels offset | Make sure you're using a physical device; emulator camera + coordinate mapping can drift |
| Low FPS | Try the back camera (usually less demanding ISP pipeline), or a mid/high-end device — MediaPipe inference is CPU/GPU intensive |
| App crashes on Undo with empty canvas | Shouldn't happen — Undo/Redo buttons are disabled automatically when there's nothing to undo/redo. If you hit this, please file an issue with device/Android version. |

---

## 📄 License

This project is provided as-is for educational/personal use. MediaPipe and its
pretrained models are © Google LLC and subject to their own license terms
(Apache 2.0) — see https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker.

---

Made by **Toushif Ansari**
