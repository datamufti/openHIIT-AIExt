# Pose tracking on the phone for workouts

Issue: [#5](https://github.com/datamufti/openHIIT-AIExt/issues/5). Researched 2026-10-11.

**Question.** Can a Flutter app use on-device pose tracking to count reps, detect when a work Interval ends, and maybe suggest changes to form or pace during a HIIT Timer?

**Answer.** Yes, for upright moves facing the camera (squats, jumping jacks, lunges, high knees). Two caveats. First, every Flutter binding is a community wrapper around Google ML Kit or MediaPipe. Second, floor moves (push-ups, mountain climbers, the floor half of a burpee) fall outside where the models are documented to work well. Rep counting is a solved pattern: classify each frame against an "up" or "down" pose, then count threshold crossings with hysteresis. Google ships a working example. The pose model should only *advise* the running Timer (rep counts, an "N reps reached" event, form hints). It should not be the sole thing that ends an Interval. The camera also forces the app to stay in the foreground with the screen on.

## 1. Engines and Flutter packages

| Engine | Landmarks | Platforms | Flutter route |
|---|---|---|---|
| Google ML Kit Pose Detection (BlazePose) | 33, plus experimental z and an in-frame likelihood for each point | Android + iOS | `google_mlkit_pose_detection` |
| MediaPipe Pose Landmarker (BlazePose GHUM lite/full/heavy) | 33, in image and world coordinates, optional segmentation mask | Android, iOS, Web, Python | Several small community packages |
| MoveNet Lightning/Thunder (TF Hub / LiteRT) | 17 | Anything that runs TFLite | Run it yourself through a TFLite interpreter plugin |
| Apple Vision `VNDetectHumanBodyPoseRequest` | 19 | iOS only | No mainstream wrapper (would need a platform channel) |

- **ML Kit.** It returns 33 landmarks. The base SDK runs at about 30 fps on a Pixel 4 and about 45 fps on an iPhone X. The "accurate" SDK is slower. z is experimental and not true 3D. It tracks one person, and a face must be present. [ML Kit pose overview](https://developers.google.com/ml-kit/vision/pose-detection)
- **`google_mlkit_pose_detection` 0.16.1.** Android minSdk 21; iOS 15.5+ and 64-bit only. All inference runs natively and is reached over platform channels. The package is "not sponsored or maintained by Google" (publisher flutter-ml.dev). [pub.dev](https://pub.dev/packages/google_mlkit_pose_detection). This is the most established option, and the app's `min_sdk_android: 34` (`pubspec.yaml`) is well above its minimum.
- **MediaPipe Pose Landmarker.** Three models (lite, full, heavy). Input is 256×256, output is 33 3D landmarks, and there is a `LIVE_STREAM` mode with an async callback. [Guide](https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker). Model card numbers: Lite is 3 MB and runs at ~44 fps on CPU or ~49 fps on GPU on a **Pixel 3**. Full is 6 MB, ~18 fps on CPU, ~40 fps on GPU. Heavy is 26 MB, ~4 fps on CPU, ~19 fps on GPU. [BlazePose GHUM 3D model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf)
- **Flutter MediaPipe wrappers.** None comes from Google:
  - `flutter_pose_detection` 0.4.1 (MIT, unverified uploader). Android API 31+ uses MediaPipe Tasks with a GPU-then-CPU fallback. iOS 14+ uses TFLite with CoreML/Metal. It claims about 3 ms per frame on GPU on a Galaxy S25 Ultra and has a built-in camera stream. [pub.dev](https://pub.dev/packages/flutter_pose_detection)
  - `flutter_mp_pose_landmarker` (Android only, CameraX). [pub.dev](https://pub.dev/packages/flutter_mp_pose_landmarker/score)
  - `vision_ai_pose` (bundles a `.task` model). [pub.dev docs](https://pub.dev/documentation/vision_ai_pose/latest/)
- **MoveNet.** Lightning (192 px input) targets latency and Thunder (256 px) targets accuracy. Google describes them as "faster than real time (30+ FPS) on most modern ... phones" and pitches them at "live fitness" apps. They output 17 keypoints and track a single pose. [TF Hub tutorial](https://www.tensorflow.org/hub/tutorials/movenet). There are no hand or foot points, which loses some form detail.
- **Apple Vision.** Apple names the weak spots itself: "If the people on the scene are bent over or upside down, the body pose algorithm will not perform as well". Flowing clothing, occlusion and "close to the edges of the screen" are also listed. [WWDC20 session 10653](https://developer.apple.com/videos/play/wwdc2020/10653/). It is only worth using if iOS later needs a native path. ML Kit or MediaPipe already covers iOS.

**Recommendation:** start with `google_mlkit_pose_detection` (base model, `STREAM_MODE`) on top of the `camera` plugin's image stream. If the plugin's quality or upkeep slips, fall back to calling MediaPipe Tasks through our own thin platform channel.

## 2. Accuracy for HIIT moves

- **Documented scope.** The BlazePose GHUM model card lists "Fitness and repetition counting" as an intended use. Its out-of-scope list includes multiple people, "too far away from the camera (e.g. further than 14 feet/4 meters)" and "Head is not visible". It also says the model is "sensitive to face position, scale and orientation". Motion and low light raise jitter. Most of its training images (85K) "capture a wide range of fitness poses." [Model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf)
- **Benchmark.** On Google's geographic evaluation set, keypoint accuracy (PDJ, i.e. PCK@0.2 of torso diameter) is about 83–89% for Lite, about 90–94% for Full and about 93–95% for Heavy. [Model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf). These are still images, not fast floor movement.
- **Camera angle.** Google's own classification guide says: "You'll get the best results with a level, straight forward, head-on image." Joint angles taken from 2D x/y coordinates shift as the user rotates. [Classifying poses](https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses)
- **A 2026 JMIR mHealth study on push-ups and squats.** 44 participants were filmed from 12 phone setups. Counting error and detection depended heavily on placement. The best setups were diagonal and frontal views at about 180–200 cm. Side views at 90 cm were worst for push-ups. [JMIR mHealth 2026;e82412](https://mhealth.jmir.org/2026/1/e82412/). I only read the abstract as indexed by search; the full text did not load. One secondary summary reports a mean detection rate of about 61% and a mean absolute error of about 1.1 reps ([letsdatascience](https://letsdatascience.com/news/smartphone-positioning-affects-pose-estimation-accuracy-044aecb9)). Treat those figures as unverified.

Assessment for each move. This is my inference from the limits above, not something a benchmark measured:

| Move | Expected | Why |
|---|---|---|
| Squats, lunges, jumping jacks, high knees | Good | Upright and frontal, with large angle changes. Google's sample counts squats out of the box. |
| Push-ups | Fair | The body is horizontal and the head is low. Accuracy depends strongly on angle (JMIR study). Google's sample does count them. |
| Burpees | Fair to poor | They cross upright→floor→jump. Fast motion causes blur and jitter, and the floor phase is "bent over". Counting the jump peak only (hip y-position) is more robust than full classification. |
| Mountain climbers | Poor | Horizontal, fast, legs occlude each other, head near the floor. Treat any count as approximate, or use cadence instead. |

## 3. Rep counting approaches and open-source examples

1. **Joint-angle state machine.** Compute a knee or elbow angle from three landmarks. Flip "down" and "up" at thresholds and count each full cycle. This is the method in Google's yoga and angle example. [Classifying poses](https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses)
2. **k-NN pose classifier with hysteresis.** Google's reference implementation lives in the ML Kit Android quickstart, under [`posedetector/classification/`](https://github.com/googlesamples/mlkit/tree/master/android/vision-quickstart/app/src/main/java/com/google/mlkit/vision/demo/java/posedetector/classification):
   - It embeds normalised pairwise joint distances.
   - It runs k-NN against [`fitness_pose_samples.csv`](https://github.com/googlesamples/mlkit/blob/master/android/vision-quickstart/app/src/main/assets/pose/fitness_pose_samples.csv) with top-K 30 then 10 ([`PoseClassifier.java`](https://github.com/googlesamples/mlkit/blob/master/android/vision-quickstart/app/src/main/java/com/google/mlkit/vision/demo/java/posedetector/classification/PoseClassifier.java)).
   - It smooths results with an EMA ([`EMASmoothing.java`](https://github.com/googlesamples/mlkit/blob/master/android/vision-quickstart/app/src/main/java/com/google/mlkit/vision/demo/java/posedetector/classification/EMASmoothing.java)).
   - [`RepetitionCounter.java`](https://github.com/googlesamples/mlkit/blob/master/android/vision-quickstart/app/src/main/java/com/google/mlkit/vision/demo/java/posedetector/classification/RepetitionCounter.java) enters a pose at confidence 6 and exits at 4 (out of 10 neighbours).
   - The classes shipped are `pushups_down` and `squats_down`.

   Google suggests about 100 samples per pose class, taken from varied angles. [Classifying poses](https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses). This logic is small enough to port to Dart.
3. **Learned repetition counters.** PoseRAC uses just two "salient poses" for each action. It reports OBO 0.56 against 0.29 for TransRAC and infers about 10× faster. [arXiv 2303.08450](https://arxiv.org/abs/2303.08450), [code](https://github.com/MiracleDance/PoseRAC). This is research grade. It is useful if we ever want one counter that works across moves, but it is not needed for v1.

I found no maintained open-source Flutter HIIT rep counter (searches on GitHub for "flutter pose rep counter" came back empty). The Google Java sample is the reference to port.

**Ending a round.** There are two options:
- The plugin raises a "target reps reached" event, and the Timer decides whether to skip to the next Interval.
- The plugin detects inactivity (no reps or very little landmark motion for N seconds) and suggests ending the Interval.

Both should be opt-in, with time as the fallback. Expect miscounts on floor moves.

**Form and pace suggestions.** Pace falls straight out of the rep timestamps (reps per minute, and slowdown relative to the Interval's first reps). Simple form cues can be rules on angles, e.g. squat depth (hip below knee) or a push-up hip sag (shoulder–hip–ankle angle). The 2D-angle caveat in [Classifying poses](https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses) applies: such cues are only reliable from a fixed, known camera angle.

## 4. Frame rate, battery, thermal

- **Frame rate.** The base model runs at real-time rates (§1). 10–15 fps is enough for counting. ML Kit's guidance: use `STREAM_MODE`, and when the detector is busy, drop frames rather than queue them. CameraX's default backpressure keeps one image at a time. Input should be at least 480×360, with the subject at least 256×256 px. [ML Kit Android guide](https://developers.google.com/ml-kit/vision/pose-detection/android)
- **Battery and thermal.** I found **no primary measurement** of power use or heat over a 20–40 minute session for ML Kit or MediaPipe pose. Expect a real cost: the camera, the NPU/GPU/CPU and a lit screen all run continuously, on a phone that is often in a warm room. Mitigations:
  - Cap analysis at about 10–15 fps and use a low camera resolution.
  - Prefer the GPU/NPU delegate.
  - Watch Android's thermal API: `PowerManager.getThermalHeadroom()` (API 30+, poll at most once per 10 s) and `addThermalStatusListener`. Degrade the frame rate or turn pose off as status rises. [ADPF thermal](https://developer.android.com/games/optimize/adpf/thermal)
  - **Measure on a Pixel before committing** (Battery Historian or `dumpsys batterystats` across a 30-minute Timer).

## 5. Phone placement

- The whole body must be in frame, head included. Stay within about 4 m. Keep the subject away from the frame edges and at least 256 px tall. Sources: [model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf), [ML Kit overview](https://developers.google.com/ml-kit/vision/pose-detection), [ML Kit Android](https://developers.google.com/ml-kit/vision/pose-detection/android), [WWDC20](https://developer.apple.com/videos/play/wwdc2020/10653/).
- **In practice:**
  - Prop the phone at hip height, about 2 m away, front or diagonal.
  - Use landscape orientation for floor moves.
  - Use one fixed position for the whole Timer.
  - Add a setup check before the Timer starts ("step back until your whole body is outlined").

  The JMIR study supports diagonal or frontal placement at 180–200 cm ([JMIR](https://mhealth.jmir.org/2026/1/e82412/)). A head-on view is best for angle rules ([Classifying poses](https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses)).
- One person only. Every model above tracks a single subject. Others in the frame can steal tracking.

## 6. Privacy

ML Kit pose and MediaPipe run inference on-device, and the Flutter ML Kit plugin processes frames natively ([pub.dev](https://pub.dev/packages/google_mlkit_pose_detection)). The plugin should keep only landmarks and derived events, never frames, and should never write video. The model card rules out "surveillance or identity recognition". ML Kit is a closed-source Google SDK, whereas MediaPipe and LiteRT are open source. That matters if any build of the app must stay free of proprietary dependencies (not verified here).

## 7. Newer multimodal / video models

- **Gemma 3n (open weights; E2B and E4B).** It "natively supports image, audio, video, and text inputs". Its MobileNet-V5 encoder "processes up to 60 frames per second on a Google Pixel". [Gemma 3n developer guide](https://developers.googleblog.com/introducing-gemma-3n-developer-guide/), [docs](https://ai.google.dev/gemma/docs/gemma-3n). E2B still loads about 1.91B effective parameters. That is fine for an occasional "describe my form in this clip" check after an Interval. It is not a 30-minute live counter on battery.
- **ML Kit GenAI Prompt API (Gemini Nano).** Text or image+text in, text out. It is in beta and does not mention video. [Docs](https://developers.google.com/ml-kit/genai/prompt/android)
- **Verdict.** Use a pose model for the live loop. A multimodal LLM is at most an optional, occasional "coach" on a few keyframes or landmark summaries, and only on supported devices.

## 8. Feasibility verdict

**Feasible on Android (Pixel) now. iOS is feasible through the same plugin.**
- High confidence for counting upright moves.
- Medium confidence for push-ups when the phone is well placed.
- Low confidence for mountain climbers and full-burpee form.

Ship it as an opt-in plugin that **advises** the Timer, with time-based Intervals as the default.

**What the plugin system must support:**

1. **Camera access.** The plugin declares a `CAMERA` permission and requests it at runtime. The host can grant or deny it per plugin.
2. **Foreground only.** On Android, `CAMERA` is a while-in-use permission. A camera foreground service cannot be started from the background, and continuing needs `FOREGROUND_SERVICE_CAMERA` ([FGS types](https://developer.android.com/develop/background-work/services/fgs/service-types)). iOS interrupts capture in the background ([AVCaptureSession interruption reason](https://developer.apple.com/documentation/avfoundation/avcapturesession/interruptionreason/videodevicenotavailableinbackground)). The plugin must stop gracefully when the app is backgrounded. The timer keeps running through `background_hiit_timer` (already a dependency). The screen stays on through `wakelock_plus` (already a dependency).
3. **UI surface.** The plugin needs a slot on the run screen for a camera preview or skeleton overlay, plus a pre-run placement check.
4. **Live event stream (plugin → host).** Events: `repCounted{n, ts}`, `targetReached`, `inactivity`, `formHint{text}`, `trackingLost`. Hosts and other plugins (audio, UI) can subscribe.
5. **Read Timer state (host → plugin).** The current Interval (type, name, remaining time), plus Interval-start and Interval-end events, so counting resets per Interval. Per-Interval config such as target reps and exercise type needs a place to live, either in the Timer definition or in plugin settings.
6. **Limited control over the running Timer.** Bounded commands, gated by user opt-in: `skipToNextInterval`, `pause`, `resume`. A plugin should never be able to silently rewrite the Timer.
7. **Resource lifecycle.** Start and stop hooks, so the camera and model run only during work Intervals of Timers that enable the plugin. Optional thermal and battery signals, so the plugin can throttle.
8. **Per-plugin settings and data.** Store pose samples and thresholds per exercise. Store per-session rep logs for history.
