# VIVE Focus 3 eye tracking through the Wave OpenXR runtime

The ALVR fork read live per-eye gaze, openness, and pupil diameter from a VIVE Focus 3 with the eye tracker module attached. The Android app used its existing OpenXR loader and session to call HTC’s `XR_HTC_eye_tracker` extension through a small Rust FFI wrapper. HTC’s Unity plugin supplied the native ABI definitions. The direct Wave Native SDK calls, such as `WVR_StartEyeTracking` and `WVR_GetEyeTracking`, belong to a separate API. [HTC OpenXR API reference](https://developer.vive.com/resources/openxr/unity/common-interface/) · [Wave Native SDK eye tracking reference](https://hub.vive.com/storage/docs/en-us/WVR_GetEyeTracking.html)

## Port the headset-side code

1. **Initialize the target app’s Android OpenXR loader** and enumerate extensions on the headset with `xrEnumerateInstanceExtensionProperties`. Look for `XR_HTC_eye_tracker`. The working ALVR client used `libopenxr_loader.so` and requested OpenXR `1.0.34` for Vive devices. Those were ALVR’s existing Vive settings.
2. **Enable the extension before `xrCreateInstance`.** Add it to the enabled instance extension list only when advertised. In ALVR’s `openxr` 0.21 Rust binding, unknown extensions go into `ExtensionSet::other`. Its intersection with the runtime’s extension list compares raw bytes, including the terminating NUL. ALVR initially put `"XR_HTC_eye_tracker".as_bytes().to_vec()` into `other`. The runtime advertised the extension, but the intersection silently dropped it. Putting `b"XR_HTC_eye_tracker\0".to_vec()` into `other` fixed tracker creation. Check the final list passed to instance creation as well as the advertised list. A native caller passing a regular C string already supplies the terminator.
3. **Check the system property.** Chain `XrSystemEyeTrackingPropertiesHTC` into `XrSystemProperties.next` and call `xrGetSystemProperties`. Log the `XrResult` and `supportsEyeTracking` independently. On the tested Focus 3, support read `false` with the module unplugged and `true` with it attached. The property, function lookup, and per-sample validity checks each give different information.
4. **Resolve the five HTC functions** using `xrGetInstanceProcAddr` on the enabled instance. Check for a successful result and a non-null pointer before calling each function. Create a tracker with `xrCreateEyeTrackerHTC` on the app’s `XrSession`. Keep the tracker tied to that session and call `xrDestroyEyeTrackerHTC` before destroying the session. These five calls were sufficient for the working probe. [HTC’s function lookup and session lifecycle](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTracker.cs)
5. **Poll inside a running OpenXR session.** For an initial render-loop probe, use the `XrTime` from `xrWaitFrame` (`predictedDisplayTime`). Use a `VIEW` reference space for head-relative gaze. Retain the pose position if the consumer needs the ray origin. ALVR’s diagnostic HUD computed gaze direction by rotating `(0, 0, -1)` by the returned gaze quaternion. Verify the direction convention in the target app by looking left, right, up, and down. The ALVR lobby used its own floor/stage space for the HUD. The streaming path used `VIEW` and sampled at its OpenXR runtime clock’s current time. [Khronos OpenXR spec on reference spaces](https://registry.khronos.org/OpenXR/specs/1.1-khr/html/xrspec.html#reference-spaces)

The working APK already declared `wave.feature.eyetracking` with `android:required="false"` and `minWaveSDKVersion=1`, plus the usual Android OpenXR permissions and runtime broker queries. The eye-tracker patch added zero HTC-specific permissions. Those are observed settings for this APK. HTC’s guide for direct Wave apps documents its own manifest configuration. [HTC Wave app capabilities](https://hub.vive.com/storage/docs/en-us/ConfigureAppCapabilities.html)

## Native ABI used by ALVR

Use the platform’s OpenXR calling convention (`extern "system"` in Rust), C struct layout (`#[repr(C)]` in Rust), OpenXR scalar types, and a 64-bit opaque HTC tracker handle. Use `XrBool32` for validity and support fields. Each entry in the table begins with `XrStructureType type` and a pointer `next`. Initialize `type` to the listed value and `next` to null unless chaining an additional structure. `XrEyeTrackerHTC` itself is a handle. [HTC structure layouts](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTrackerHelper.cs) · [HTC structure-type values](https://github.com/ViveSoftware/VIVE-OpenXR-Unity/blob/master/com.htc.upm.vive.openxr/Runtime/OpenXRUtils.cs)

| Struct | `type` value | Fields following `type` and `next`, in order |
| --- | ---: | --- |
| `XrEyeTrackerCreateInfoHTC` | `1000326001` | — |
| `XrEyeGazeDataInfoHTC` | `1000326002` | `XrSpace baseSpace`, `XrTime time` |
| `XrEyeGazeDataHTC` | `1000326003` | `XrTime time`, gaze entries `[2]` |
| `XrSystemEyeTrackingPropertiesHTC` | `1000326005` | `XrBool32 supportsEyeTracking` |
| `XrEyePupilDataInfoHTC` | `1000326006` | — in HTC’s Unity definitions. ALVR appended `XrTime time`. |
| `XrEyePupilDataHTC` | `1000326007` | `XrTime time`, pupil entries `[2]` |
| `XrEyeGeometricDataInfoHTC` | `1000326009` | — in HTC’s Unity definitions. ALVR appended `XrTime time`. |
| `XrEyeGeometricDataHTC` | `1000326010` | `XrTime time`, geometric entries `[2]` |

For a fresh binding, use HTC’s published input layouts for pupil and geometry. Both define just `type` and `next`, and HTC’s Unity wrapper seeds the **output** `time` with `predictedDisplayTime` before calling. ALVR seeded output `time` as well and used the larger input structs in its successful probe. The observed result establishes that the tested runtime accepted the ALVR calls. Recheck the fields against the target runtime and keep the returned timestamps if the consumer needs sample timing. The ALVR wrapper returned only the arrays. [HTC’s pupil and geometry calls](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTracker.cs)

Each function returns `XrResult`. The arguments below are in call order. `input*` is a pointer to the corresponding `Info` struct and `output*` is a writable pointer to the corresponding `Data` struct.

| Function | Arguments |
| --- | --- |
| `xrCreateEyeTrackerHTC` | `XrSession`, `XrEyeTrackerCreateInfoHTC*`, `XrEyeTrackerHTC*` (out) |
| `xrDestroyEyeTrackerHTC` | `XrEyeTrackerHTC` |
| `xrGetEyeGazeDataHTC` | `XrEyeTrackerHTC`, gaze `input*`, gaze `output*` |
| `xrGetEyePupilDataHTC` | `XrEyeTrackerHTC`, pupil `input*`, pupil `output*` |
| `xrGetEyeGeometricDataHTC` | `XrEyeTrackerHTC`, geometric `input*`, geometric `output*` |

Each output holds an inline array with left at index `0` and right at index `1`. The following entries begin directly with their validity fields.

| Entry | Fields in order | How to use it |
| --- | --- | --- |
| Gaze | `XrBool32 isValid`, `XrPosef gazePose` | Use the valid position and orientation in the requested `baseSpace`. OpenXR stores the quaternion as `x, y, z, w`. |
| Pupil | `XrBool32 isDiameterValid`, `XrBool32 isPositionValid`, `float pupilDiameter`, `XrVector2f pupilPosition` | Gate diameter and position independently. HTC describes diameter in millimeters and sensor position in `[0,1]`, with +X right and +Y up. |
| Geometry | `XrBool32 isValid`, `float eyeOpenness`, `float eyeWide`, `float eyeSqueeze` | Use openness when valid. HTC describes `0` as normally closed and `1` as normally open. ALVR derived blink as `1 - openness`. |

Treat a successful query with an invalid eye or field as missing data. A zero openness value can represent a closed eye when the sample is valid. [HTC per-eye field definitions](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTrackerHelper.cs)

## Verify on the headset

Use the standalone Android app’s lobby or render loop as the first test. The ALVR lobby logged and displayed samples every 60 rendered frames, so it could verify acquisition before connecting to a server.

1. Attach the Focus 3 eye tracker module and put on the headset. The module occupied the headset’s USB port in our setup, so we used Wi-Fi ADB for logs. After reboot we had to re-enable ADB over TCP during a brief USB swap. The display’s proximity sensor can pause rendering and stall the probe while the headset is off your face.
2. Log extension advertisement, the enabled extension list, `supportsEyeTracking`, tracker creation, and the result of each query. In ALVR, `adb logcat | grep -F 'ALVR NATIVE-RUST'` showed the probe.
3. Hold your head still and look around. Confirm both valid gaze vectors follow your eyes. Blink each eye separately and check the corresponding openness. Log pupil diameter and position with their separate validity flags. Keep the probe logs at a low rate.

The original headset notes record successful tracker creation, updating gaze and openness in the in-headset HUD, and log output for gaze poses, openness/wide/squeeze, and pupil diameter. Pupil position was queried, but the notes establish live pupil diameter more clearly than usable pupil position. Confirm pupil position and timing on the target runtime if the port needs them.

## Where this lived in ALVR

The original source is in the nested ALVR checkout at `/home/rei/_/ALVR/ALVR`, on `feature/htc-eye-tracker`. From that checkout, use `git show <commit>:<path>` to read the files below without switching branches.

| Commit | Files to inspect |
| --- | --- |
| `5b3c5f108dba41fd5fa9e8701e29ab5d00f3d2fe` | `alvr/client_openxr/src/extra_extensions/eye_tracking_htc.rs` has the Rust wrapper. `lib.rs` has the extension negotiation fix. `interaction.rs` and `lobby.rs` have the session integration and local probe. `alvr/client_openxr/Cargo.toml` has the Android metadata. `docs/status.md` records the live headset results. |
| `6d2af37559844ad0d8174272e4bd240d4efb82fd` | `alvr/client_openxr/src/interaction.rs` forwards readings through `FaceData`. `alvr/server_core/src/tracking/face.rs` implements the optional Mittens output. `docs/review/htc_eyetracking.md` documents that transport and its test status. |

ALVR later carried the readings in its tracking stream and exposed them as binary UDP on `127.0.0.1:9002` for Mittens. The packet change required rebuilding both ALVR halves. The fork’s notes confirm the headset probe and leave live end-to-end validation of that sink open. The headset-side acquisition works independently of that transport.

## Handoff prompt

> Use this README to add HTC VIVE per-eye tracking to the target repository. Find its Android OpenXR loader, instance creation, session lifecycle, and frame or input loop. Enable `XR_HTC_eye_tracker` when the runtime advertises it, check what actually reaches instance creation, and hook the tracker into the existing session. Start with an on-headset probe for left/right gaze and openness. Add pupil data if the target needs it. Carry the space, `XrTime`, and per-field validity through the target’s own tracking types. Report device-tested behavior separately from build-only results.
