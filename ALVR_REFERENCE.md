# ALVR source and Focus 3 blink observations

These excerpts come from the ALVR fork’s `feature/htc-eye-tracker` branch. Commit `5b3c5f108dba41fd5fa9e8701e29ab5d00f3d2fe` is the headset-tested probe. Commit `6d2af37559844ad0d8174272e4bd240d4efb82fd` adds the blink conversion and transport. The original notes record a live geometry read on the headset. They leave the later blendshape and UDP transport’s end-to-end hardware validation open. The ALVR excerpts are distributed under the [original license](#alvr-license).

## Complete Rust wrapper

`alvr/client_openxr/src/extra_extensions/eye_tracking_htc.rs` at `5b3c5f1`, verbatim. The two `Info` structs for geometry and pupils in this file include a trailing `time`. HTC’s Unity bindings define those inputs with only `type` and `next`. Both versions put `time` in the output struct, and the ALVR wrapper initialized both copies before calling the runtime. Compare offsets and widths against the actual port, especially `XrBool32` and the inline two-element arrays. [HTC structure definitions](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTrackerHelper.cs)

```rust
use crate::extra_extensions::get_instance_proc;
use openxr::{self as xr, sys};
use std::{ffi::c_void, ptr};

// Reconstructed ABI for the proprietary `XR_HTC_eye_tracker` extension.
//
// This extension is NOT part of the Khronos OpenXR registry (confirmed absent from
// OpenXR-SDK openxr.h and openxr.xml), so `openxr-sys` provides no types for it. The
// structures, structure-type values and function signatures below were reconstructed
// from HTC's reference implementation in ViveSoftware/VIVE-OpenXR:
//
//   Runtime/Features/EyeTracker/ViveEyeTrackerHelper.cs
//   Runtime/OpenXRUtils.cs                (structure-type numeric values)
//   Runtime/Features/EyeTracker/ViveEyeTracker.cs (proc address names)
//
// Structure-type enumeration values are extension number 326 => base 1000326000.

pub const EYE_TRACKER_HTC_EXTENSION_NAME: &str = "XR_HTC_eye_tracker";

// Structure types are sequentially assigned starting at extension base 1000326001.
fn htc_ty(offset: i32) -> sys::StructureType {
    sys::StructureType::from_raw(1000326000 + offset)
}

const EYE_TRACKER_CREATE_INFO_HTC: i32 = 1;
const EYE_GAZE_DATA_INFO_HTC: i32 = 2;
const EYE_GAZE_DATA_HTC: i32 = 3;
const SYSTEM_EYE_TRACKING_PROPERTIES_HTC: i32 = 5;
const EYE_PUPIL_DATA_INFO_HTC: i32 = 6;
const EYE_PUPIL_DATA_HTC: i32 = 7;
const EYE_GEOMETRIC_DATA_INFO_HTC: i32 = 9;
const EYE_GEOMETRIC_DATA_HTC: i32 = 10;

#[repr(transparent)]
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
pub struct EyeTrackerHandleHTC(pub u64);

impl EyeTrackerHandleHTC {
    pub const NULL: Self = Self(u64::MAX);
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyeTrackerCreateInfoHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct SystemEyeTrackingPropertiesHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
    pub supports_eye_tracking: sys::Bool32,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyeGazeDataInfoHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
    pub base_space: sys::Space,
    pub time: sys::Time,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct SingleEyeGazeDataHTC {
    pub is_valid: sys::Bool32,
    pub gaze_pose: sys::Posef,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyeGazeDataHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
    pub time: sys::Time,
    pub gaze: [SingleEyeGazeDataHTC; 2],
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyePupilDataInfoHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct SingleEyePupilDataHTC {
    pub is_diameter_valid: sys::Bool32,
    pub is_position_valid: sys::Bool32,
    pub pupil_diameter: f32,
    pub pupil_position: sys::Vector2f,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyePupilDataHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
    pub time: sys::Time,
    pub pupil_data: [SingleEyePupilDataHTC; 2],
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyeGeometricDataInfoHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct SingleEyeGeometricDataHTC {
    pub is_valid: sys::Bool32,
    pub eye_openness: f32,
    pub eye_wide: f32,
    pub eye_squeeze: f32,
}

#[repr(C)]
#[derive(Copy, Clone)]
pub struct EyeGeometricDataHTC {
    pub ty: sys::StructureType,
    pub next: *const c_void,
    pub time: sys::Time,
    pub eye_geometric_data: [SingleEyeGeometricDataHTC; 2],
}

type CreateEyeTrackerHTC = unsafe extern "system" fn(
    sys::Session,
    *const EyeTrackerCreateInfoHTC,
    *mut EyeTrackerHandleHTC,
) -> sys::Result;

type DestroyEyeTrackerHTC = unsafe extern "system" fn(EyeTrackerHandleHTC) -> sys::Result;

type GetEyeGazeDataHTC = unsafe extern "system" fn(
    EyeTrackerHandleHTC,
    *const EyeGazeDataInfoHTC,
    *mut EyeGazeDataHTC,
) -> sys::Result;

type GetEyePupilDataHTC = unsafe extern "system" fn(
    EyeTrackerHandleHTC,
    *const EyePupilDataInfoHTC,
    *mut EyePupilDataHTC,
) -> sys::Result;

type GetEyeGeometricDataHTC = unsafe extern "system" fn(
    EyeTrackerHandleHTC,
    *const EyeGeometricDataInfoHTC,
    *mut EyeGeometricDataHTC,
) -> sys::Result;

pub struct EyeTrackerHTC {
    _session: xr::Session<xr::AnyGraphics>,
    handle: EyeTrackerHandleHTC,
    destroy: DestroyEyeTrackerHTC,
    get_eye_gazes: GetEyeGazeDataHTC,
    get_eye_pupils: GetEyePupilDataHTC,
    get_eye_geometrics: GetEyeGeometricDataHTC,
}

impl EyeTrackerHTC {
    pub fn new<G>(session: xr::Session<G>, _system: xr::SystemId) -> xr::Result<Self> {
        let create: CreateEyeTrackerHTC = get_instance_proc(&session, "xrCreateEyeTrackerHTC")?;
        let destroy: DestroyEyeTrackerHTC = get_instance_proc(&session, "xrDestroyEyeTrackerHTC")?;
        let get_eye_gazes: GetEyeGazeDataHTC = get_instance_proc(&session, "xrGetEyeGazeDataHTC")?;
        let get_eye_pupils: GetEyePupilDataHTC =
            get_instance_proc(&session, "xrGetEyePupilDataHTC")?;
        let get_eye_geometrics: GetEyeGeometricDataHTC =
            get_instance_proc(&session, "xrGetEyeGeometricDataHTC")?;

        let mut handle = EyeTrackerHandleHTC::NULL;
        let info = EyeTrackerCreateInfoHTC {
            ty: htc_ty(EYE_TRACKER_CREATE_INFO_HTC),
            next: ptr::null(),
        };
        unsafe {
            super::xr_res(create(session.as_raw(), &info, &mut handle))?;
        }

        Ok(Self {
            _session: session.into_any_graphics(),
            handle,
            destroy,
            get_eye_gazes,
            get_eye_pupils,
            get_eye_geometrics,
        })
    }

    pub fn supports_eye_tracking<G>(session: &xr::Session<G>, system: xr::SystemId) -> bool {
        super::get_props(
            session,
            system,
            SystemEyeTrackingPropertiesHTC {
                ty: htc_ty(SYSTEM_EYE_TRACKING_PROPERTIES_HTC),
                next: ptr::null_mut(),
                supports_eye_tracking: sys::FALSE,
            },
        )
        .map(|props| props.supports_eye_tracking.into())
        .unwrap_or(false)
    }

    pub fn get_eye_gazes(
        &self,
        base_space: &xr::Space,
        time: xr::Time,
    ) -> xr::Result<[SingleEyeGazeDataHTC; 2]> {
        let info = EyeGazeDataInfoHTC {
            ty: htc_ty(EYE_GAZE_DATA_INFO_HTC),
            next: ptr::null(),
            base_space: base_space.as_raw(),
            time,
        };

        let mut data = EyeGazeDataHTC {
            ty: htc_ty(EYE_GAZE_DATA_HTC),
            next: ptr::null_mut(),
            time,
            gaze: unsafe { std::mem::zeroed() },
        };

        unsafe {
            super::xr_res((self.get_eye_gazes)(self.handle, &info, &mut data))?;
        }

        Ok(data.gaze)
    }

    pub fn get_eye_pupils(&self, time: xr::Time) -> xr::Result<[SingleEyePupilDataHTC; 2]> {
        let info = EyePupilDataInfoHTC {
            ty: htc_ty(EYE_PUPIL_DATA_INFO_HTC),
            next: ptr::null(),
        };

        let mut data = EyePupilDataHTC {
            ty: htc_ty(EYE_PUPIL_DATA_HTC),
            next: ptr::null_mut(),
            time,
            pupil_data: unsafe { std::mem::zeroed() },
        };

        unsafe {
            super::xr_res((self.get_eye_pupils)(self.handle, &info, &mut data))?;
        }

        Ok(data.pupil_data)
    }

    pub fn get_eye_geometrics(&self, time: xr::Time) -> xr::Result<[SingleEyeGeometricDataHTC; 2]> {
        let info = EyeGeometricDataInfoHTC {
            ty: htc_ty(EYE_GEOMETRIC_DATA_INFO_HTC),
            next: ptr::null(),
        };

        let mut data = EyeGeometricDataHTC {
            ty: htc_ty(EYE_GEOMETRIC_DATA_HTC),
            next: ptr::null_mut(),
            time,
            eye_geometric_data: unsafe { std::mem::zeroed() },
        };

        unsafe {
            super::xr_res((self.get_eye_geometrics)(self.handle, &info, &mut data))?;
        }

        Ok(data.eye_geometric_data)
    }
}

impl Drop for EyeTrackerHTC {
    fn drop(&mut self) {
        unsafe {
            (self.destroy)(self.handle);
        }
    }
}
```

The helper called by the wrapper lives in `alvr/client_openxr/src/extra_extensions/mod.rs` at `5b3c5f1`. It also provides the `xrGetSystemProperties` chain and result conversion used by the wrapper.

```rust
use openxr::{self as xr, sys};
use std::ffi::CString;
use std::mem;

fn xr_res(result: sys::Result) -> xr::Result<()> {
    if result.into_raw() >= 0 {
        Ok(())
    } else {
        Err(result)
    }
}

fn get_props<G, T>(
    session: &xr::Session<G>,
    system: xr::SystemId,
    default_struct: T,
) -> xr::Result<T> {
    let instance = session.instance();

    let mut props = default_struct;
    let mut system_properties = sys::SystemProperties::out((&raw mut props).cast());
    let result = unsafe {
        (instance.fp().get_system_properties)(
            instance.as_raw(),
            system,
            system_properties.as_mut_ptr(),
        )
    };

    xr_res(result).map(|_| props)
}

fn get_instance_proc<G, FnTy>(session: &xr::Session<G>, method_name: &str) -> xr::Result<FnTy> {
    unsafe {
        let method_name = CString::new(method_name).unwrap();
        let mut function_handle = None;

        xr_res((session.instance().fp().get_instance_proc_addr)(
            session.instance().as_raw(),
            method_name.as_ptr(),
            &mut function_handle,
        ))?;

        function_handle
            .map(|pfn| mem::transmute_copy(&pfn))
            .ok_or(sys::Result::ERROR_EXTENSION_NOT_PRESENT)
    }
}
```

## Extension enablement

From `alvr/client_openxr/src/lib.rs` at `5b3c5f1`. ALVR’s `openxr` 0.21 used bytes in `ExtensionSet::other`. The older `openxr` 0.19.0 uses `Vec<String>` and appends the NUL in `names()`, so pass the plain `"XR_HTC_eye_tracker"` string to that version. [openxr 0.19.0 source](https://github.com/Ralith/openxrs/blob/7ff42adc9a9685a52d45d4469b7569a35afa7f51/openxr/src/generated.rs#L1497-L1502)

```rust
    selected_exts.other = [
        META_BODY_TRACKING_FIDELITY_EXTENSION_NAME,
        BD_MOTION_TRACKING_EXTENSION_NAME,
        PICO_CONFIGURATION_EXTENSION_NAME,
        EYE_TRACKER_HTC_EXTENSION_NAME,
    ]
    .into_iter()
    .map(|s| {
        let mut v = s.as_bytes().to_vec();
        v.push(0);
        v
    })
    .collect();

    selected_exts = selected_exts.intersection(&available_exts);

    let available_layers = xr_entry.enumerate_layers().unwrap();
    info!("OpenXR available layers: {available_layers:#?}");

    let other_exts = selected_exts
        .other
        .iter()
        .map(|vec| {
            CStr::from_bytes_with_nul(vec)
                .unwrap()
                .to_str()
                .unwrap()
                .to_owned()
        })
        .collect::<Vec<_>>();

    let xr_instance = xr_entry
        .create_instance(
            &xr::ApplicationInfo {
                application_name: "ALVR Client",
                application_version: 0,
                engine_name: "ALVR",
                engine_version: 0,
                api_version: openxr_version,
            },
            &selected_exts,
            &[],
        )
        .unwrap();
```

## Query timing, space, and blink conversion

For the headset probe, `lib.rs` took `vsync_time` from `xrWaitFrame`’s `predicted_display_time`, passed it to the lobby, and converted those same nanoseconds back into an `XrTime`. `lobby.rs` chose `LOCAL_FLOOR_EXT` when enabled and `STAGE` otherwise. The space parameter went only to gaze. Geometry took the timestamp alone. The relevant call site in `lobby.rs` at `5b3c5f1` follows through the two queries.

```rust
        // Phase 2 diagnostic self-test: poll the XR_HTC_eye_tracker once per second in the lobby,
        // so we can validate gaze/openness/pupil data without requiring a stream connection.
        self.eye_probe_counter += 1;
        if self.eye_probe_counter % 60 == 0 {
            let face_sources = &self.interaction_ctx.read().face_sources;
            interaction::get_face_data(
                &self.xr_session,
                face_sources,
                &self.reference_space,
                vsync_time,
            );

            if let Some(tracker) = &face_sources.eye_tracker_htc {
                let xr_time = crate::to_xr_time(vsync_time);
                let hud = match (
                    tracker.get_eye_gazes(&self.reference_space, xr_time),
                    tracker.get_eye_geometrics(xr_time),
                ) {
                    (Ok(gazes), Ok(geos)) => {
                        let fmt_dir = |s: &crate::extra_extensions::SingleEyeGazeDataHTC| {
                            let q = s.gaze_pose.orientation;
                            format!(
                                "({:+.2}, {:+.2}, {:+.2})",
                                -2.0 * (q.x * q.z + q.w * q.y),
                                2.0 * (q.w * q.x - q.y * q.z),
                                -1.0 + 2.0 * (q.x * q.x + q.y * q.y)
                            )
                        };
                        format!(
                            "EYE  L {}{} o={:.2}  R {}{} o={:.2}",
                            fmt_dir(&gazes[0]),
                            if gazes[0].is_valid != xr::sys::FALSE { "" } else { "?" },
                            geos[0].eye_openness,
                            fmt_dir(&gazes[1]),
                            if gazes[1].is_valid != xr::sys::FALSE { "" } else { "?" },
                            geos[1].eye_openness,
                        )
                    }
                    _ => "EYE  unavailable".to_owned(),
                };
                self.renderer.update_hud_message(&hud);
            }
        }
```

`get_face_data` in `interaction.rs` converted the `time: Duration` parameter with `let xr_time = crate::to_xr_time(time);`. The streaming caller in `stream.rs` at `6d2af37` passed `VIEW` space and `now` from `xr_runtime_now`, which was the OpenXR runtime’s current time. The stream’s separate prediction offset was used for head tracking.

```rust
        let face = interaction::get_face_data(
            &xr_session,
            &int_ctx.face_sources,
            view_reference_space,
            now,
        );
```

The probe in `5b3c5f1` logged geometry. `interaction.rs` at `6d2af37` added the conversion below for ALVR’s HTC eye blendshape packet. Both eyes had to have valid geometry before it generated the 14-element vector. The left and right blink weights went into slots `0` and `2`. The raw `EyeDataHtc.openness` fields were kept independently for each eye, as two optional values. The later merge put the synthesized weights in the HTC eye slot and preserved any HTC lip weights.

```rust
        if let Ok(geometrics) = tracker.get_eye_geometrics(xr_time) {
            let l = &geometrics[0];
            let r = &geometrics[1];

            let (l_open, r_open) = (
                (bool::from(l.is_valid)).then_some(l.eye_openness),
                (bool::from(r.is_valid)).then_some(r.eye_openness),
            );

            if let (Some(l_open), Some(r_open)) = (l_open, r_open) {
                let mut blink = vec![0.0; 14];
                blink[0] = (1.0 - l_open).clamp(0.0, 1.0);
                blink[2] = (1.0 - r_open).clamp(0.0, 1.0);
                synthesized_blink = Some(blink);
            }

            info!(
                "L open={:.3} wide={:.3} squeeze={:.3} valid={} | R open={:.3} wide={:.3} squeeze={:.3} valid={}",
                l.eye_openness,
                l.eye_wide,
                l.eye_squeeze,
                l.is_valid,
                r.eye_openness,
                r.eye_wide,
                r.eye_squeeze,
                r.is_valid,
            );

            eye_data_htc = Some(EyeDataHtc {
                openness: [
                    (bool::from(l.is_valid)).then_some(l.eye_openness),
                    (bool::from(r.is_valid)).then_some(r.eye_openness),
                ],
                wide: [
                    (bool::from(l.is_valid)).then_some(l.eye_wide),
                    (bool::from(r.is_valid)).then_some(r.eye_wide),
                ],
                squeeze: [
                    (bool::from(l.is_valid)).then_some(l.eye_squeeze),
                    (bool::from(r.is_valid)).then_some(r.eye_squeeze),
                ],
                ..Default::default()
            });
        }
```

## Android manifest metadata

From `alvr/client_openxr/Cargo.toml` at `5b3c5f1`. This is the Android package and manifest metadata, with the build-signing tables skipped between the package stanza and the SDK stanza. The `# Vive entries` block declares the optional `wave.feature.eyetracking` feature and `minWaveSDKVersion=1`. The two Khronos permissions, runtime broker queries, and immersive activity category were already present. The probe patch changed the package ID and launcher label so it could install alongside stock ALVR. It added zero eye-tracking manifest entries. The Quest and Pico entries belong to those platform configurations.

```toml
[package.metadata.android]
package = "alvr.client.dev.eyetracker"     # Separate package: installs alongside stock ALVR (Focus 3 launcher shows both)
install_location = "auto"
build_targets = ["aarch64-linux-android"]
runtime_libs = "../../deps/android_openxr"
resources = "resources"

[package.metadata.android.sdk]
min_sdk_version = 28
target_sdk_version = 32

[[package.metadata.android.uses_feature]]
name = "android.hardware.microphone"
required = true
[[package.metadata.android.uses_feature]]
name = "android.hardware.vr.headtracking"
required = true
version = 1
[[package.metadata.android.uses_feature]]
opengles_version = [3, 2]
required = true

[[package.metadata.android.uses_permission]]
name = "android.permission.ACCESS_WIFI_STATE"
[[package.metadata.android.uses_permission]]
name = "android.permission.ACCESS_NETWORK_STATE"
[[package.metadata.android.uses_permission]]
name = "android.permission.CHANGE_WIFI_MULTICAST_STATE"
[[package.metadata.android.uses_permission]]
name = "android.permission.INTERNET"
[[package.metadata.android.uses_permission]]
name = "android.permission.RECORD_AUDIO"
# WAKE_LOCK is needed for proper wifi locking
# https://developer.android.com/reference/android/net/wifi/WifiManager.WifiLock
[[package.metadata.android.uses_permission]]
name = "android.permission.WAKE_LOCK"
[[package.metadata.android.uses_permission]]
name = "org.khronos.openxr.permission.OPENXR"
[[package.metadata.android.uses_permission]]
name = "org.khronos.openxr.permission.OPENXR_SYSTEM"

[[package.metadata.android.queries.intent]]
actions = ["org.khronos.openxr.OpenXRRuntimeService"]
[[package.metadata.android.queries.provider]]
name = "org.khronos.openxr"
authorities = "org.khronos.openxr.runtime_broker;org.khronos.openxr.system_runtime_broker"

[package.metadata.android.application]
debuggable = false
theme = "@android:style/Theme.Black.NoTitleBar.Fullscreen"
icon = "@mipmap/ic_launcher"
label = "ALVR EyeTrack"

[package.metadata.android.application.activity]
config_changes = "density|keyboard|keyboardHidden|navigation|orientation|screenLayout|screenSize|uiMode"
launch_mode = "singleTask"
orientation = "landscape"

[[package.metadata.android.application.activity.intent_filter]]
actions = ["android.intent.action.MAIN"]
categories = [
    "android.intent.category.LAUNCHER",
    "com.oculus.intent.category.VR",
    "com.yvr.intent.category.VR",
    "org.khronos.openxr.intent.category.IMMERSIVE_HMD",
]

# Quest entries
[[package.metadata.android.uses_feature]]
name = "oculus.software.eye_tracking"
required = false
[[package.metadata.android.uses_feature]]
name = "oculus.software.face_tracking"
required = false
[[package.metadata.android.uses_feature]]
name = "oculus.software.handtracking"
required = false
[[package.metadata.android.uses_feature]]
name = "com.oculus.feature.PASSTHROUGH"
required = false
[[package.metadata.android.uses_feature]]
name = "com.oculus.software.body_tracking"
required = false
[[package.metadata.android.uses_permission]]
name = "com.oculus.permission.BODY_TRACKING"
[[package.metadata.android.uses_permission]]
name = "com.oculus.permission.EYE_TRACKING"
[[package.metadata.android.uses_permission]]
name = "com.oculus.permission.FACE_TRACKING"
[[package.metadata.android.uses_permission]]
name = "com.oculus.permission.HAND_TRACKING"
[[package.metadata.android.uses_permission]]
name = "com.oculus.permission.WIFI_LOCK"
[[package.metadata.android.application.meta_data]]
name = "com.oculus.intent.category.VR"
value = "vr_only"
[[package.metadata.android.application.meta_data]]
name = "com.oculus.supportedDevices"
# Note: value is changed for the Meta store, which requires an explicit list of platforms.
# "all" is required to support Quest 1 which doesn't have newer platform names registered.
value = "all"
[[package.metadata.android.application.meta_data]]
name = "com.oculus.vr.focusaware"
value = "true"
[[package.metadata.android.application.meta_data]]
name = "com.oculus.handtracking.frequency"
value = "HIGH"
[[package.metadata.android.application.meta_data]]
name = "com.oculus.handtracking.version"
value = "V2.0"

# Vive entries
[[package.metadata.android.uses_feature]]
name = "wave.feature.eyetracking"
required = false
[[package.metadata.android.uses_feature]]
name = "wave.feature.handtracking"
required = true
[[package.metadata.android.uses_feature]]
name = "wave.feature.lipexpression"
required = false
[[package.metadata.android.application.meta_data]]
name = "minWaveSDKVersion"
value = "1"
[[package.metadata.android.application.meta_data]]
name = "com.htc.vr.content.NumController"
value = "1,2"
[[package.metadata.android.application.meta_data]]
name = "com.htc.vr.content.NumDoFController"
value = "3,6DoF"
[[package.metadata.android.application.meta_data]]
name = "com.htc.vr.content.NumDoFHmd"
value = "3,6DoF"

# Pico entries
[[package.metadata.android.uses_permission]]
name = "com.picovr.permission.EYE_TRACKING"
[[package.metadata.android.uses_permission]]
name = "com.picovr.permission.FACE_TRACKING"
[[package.metadata.android.application.meta_data]]
name = "eyetracking_calibration"
value = "true"
[[package.metadata.android.application.meta_data]]
name = "handtracking"
value = "1"
[[package.metadata.android.application.meta_data]]
name = "picovr.software.eye_tracking"
value = "1"
[[package.metadata.android.application.meta_data]]
name = "picovr.software.face_tracking"
value = "true"
[[package.metadata.android.application.meta_data]]
name = "pvr.app.type"
value = "vr"
[[package.metadata.android.application.meta_data]]
name = "pvr.display.orientation"
value = "180"
[[package.metadata.android.application.meta_data]]
name = "pvr.sdk.version"
value = "OpenXR"
[[package.metadata.android.application.meta_data]]
name = "pxr.sdk.version_code"
value = "5900"

# Yvr entries
[[package.metadata.android.application.meta_data]]
name = "com.yvr.intent.category.VR"
value = "vr_only"

# Android XR
[[package.metadata.android.uses_feature]]
name = "android.software.xr.api.spatial"
required = true
[[package.metadata.android.uses_feature]]
name = "android.software.xr.api.openxr"
required = true
[[package.metadata.android.uses_feature]]
name = "android.software.xr.input.controller"
required = false
[[package.metadata.android.uses_feature]]
name = "android.software.xr.input.hand_tracking"
required = false
[[package.metadata.android.uses_feature]]
name = "android.software.xr.input.eye_tracking"
required = false

[[package.metadata.android.application.uses_native_library]]
name = "libopenxr.google.so"
required = false

[[package.metadata.android.application.property]]
name = "android.window.PROPERTY_XR_ACTIVITY_START_MODE"
value = "XR_ACTIVITY_START_MODE_FULL_SPACE_UNMANAGED"

[[package.metadata.android.application.property]]
name = "android.window.PROPERTY_XR_BOUNDARY_TYPE_RECOMMENDED"
value = "XR_BOUNDARY_TYPE_LARGE"
```

## What the headset test showed

Excerpt from `docs/status.md` at `5b3c5f1`.

> - Verified the VIVE WAVE runtime advertises `XR_HTC_eye_tracker`.
> - Fixed an ALVR bug: `lib.rs` stored `other` extension names **without** the NUL
>   byte, so the openxr crate's byte-exact `intersection` silently dropped the
>   extension before `xrCreateInstance`. NUL-terminating the `other` entries fixed
>   it (names() requires NUL-terminated strings anyway).
> - **Result: `xrCreateEyeTrackerHTC` now succeeds on real hardware**
>   (`[ALVR NATIVE-RUST]: XR_HTC_eye_tracker created successfully`).
> - `supports_eye_tracking = true` confirmed **once the eye tracker module is
>   physically plugged in** — validates `XrSystemEyeTrackingPropertiesHTC` struct
>   chain too (it read `false` while the module was unplugged).
> - **Per-frame data validated with a lobby self-test** (no stream needed): added a
>   ~1/sec probe in `lobby.rs::render` that calls `get_face_data` and logs
>   `XR_HTC_eye_tracker` gaze/openness/pupil output at `info!` level. Live logs
>   show per-eye gaze (pose + position, `valid=true`), openness/wide/squeeze, and
>   per-eye pupil diameter. Runtime's own `XrFacialTracker` matches (e.g.
>   `LEFT_BLINK_HTC=1.0` ⇔ `open=0.0` when eyes closed). Probe is temporary.
> - Logging was also bumped from `debug!` to `info!` in `interaction.rs::get_face_data`
>   so the probe lines are visible in logcat.
> - **In-headset live HUD text added**: `Lobby::render` now pushes a per-eye line into
>   the existing lobby HUD (via `update_hud_message`) ~1/sec, e.g.
>   `EYE  L (-0.31, 0.02, 0.95) o=0.93  R (-0.29, 0.01, 0.96) o=0.91`. The direction is
>   the unit forward vector computed from the eye gaze quaternion. **Confirmed visible
>   on-headset by the user** (values in ~[-1,1] as expected for a unit direction).

- **Manifest.** The above configuration worked on the tested Focus 3. Its eye-tracking feature declaration was optional. HTC describes `android:required="true"` in its capability guide for direct Wave apps. The ALVR patch used the OpenXR extension and left its existing Vive capability settings in place. [HTC Wave app capabilities](https://hub.vive.com/storage/docs/en-us/ConfigureAppCapabilities.html)
- **Calibration.** The ALVR wrapper has no calibration call. The original probe notes do not record the headset’s calibration history or a before/after comparison. HTC’s Focus 3 setup uses **Settings → Inputs → Eye tracker → Calibrate**, after turning the tracker on and acknowledging the first-connection notice. HTC says to calibrate on first install, for each user, and after changing IPD or headset fit. [HTC Focus 3 Eye Tracker calibration](https://business.vive.com/us/support/focus3-eye-tracker/category_howto/calibrating-eye-tracker.html)
- **Hot-plug.** `supports_eye_tracking` was checked with the module unplugged (`false`) and attached (`true`). The notes give no evidence of a running session recovering after a disconnect or a reconnect. ALVR attempted tracker creation during interaction initialization and kept the resulting handle for the session. Its probe had no reconnect handler. For a baseline hardware test, attach the module before launching the app.
- **Openness range.** HTC documents `[0, 1]` for ordinary openness, with `0` normally closed and `1` normally open. The status excerpt records `0.0` at closure and shows example HUD values of `0.93` and `0.91` while open. The transport clamps `1.0 - openness` into `[0, 1]`. [HTC geometry definitions](https://raw.githubusercontent.com/ViveSoftware/VIVE-OpenXR-Unity/master/com.htc.upm.vive.openxr/Runtime/Features/EyeTracker/ViveEyeTrackerHelper.cs)
- **Validity during a blink.** The closure example does not give the corresponding geometric `is_valid` value. The `valid=true` in the excerpt refers to gaze. The HUD printed geometric openness without displaying its validity, and the notes have no per-frame closure trace. HTC also documents that some runtime versions return zero openness when the eye is undetected. A zero is usable as a closed eye only with valid geometry. For an on-device check, log the result of `xrGetEyeGeometricDataHTC` and each eye’s `is_valid` and `eye_openness` together while winking and blinking. [HTC VIVE OpenXR 2.5.1 release notes](https://developer.vive.com/resources/openxr/unity/download/latest/)

## ALVR license

```text
Copyright (c) 2018-2019 polygraphene
Copyright (c) 2020-2024 alvr-org

Permission is hereby granted, free of charge, to any person
obtaining a copy of this software and associated documentation
files (the "Software"), to deal in the Software without
restriction, including without limitation the rights to use,
copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following
conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
OTHER DEALINGS IN THE SOFTWARE.
```
