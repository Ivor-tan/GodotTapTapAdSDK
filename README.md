# GodotTapTapAdSDK

Godot 4 Android plugin for the official TapADN / Dirichlet advertising SDK. It provides a small GDScript API for rewarded-video and interstitial ads while keeping privacy-sensitive data collection disabled by default.

## Features

- Godot 4 Android export plugin and `GodotTapTapAd` autoload
- Rewarded-video loading, display, lifecycle callbacks, and reward verification
- Interstitial loading, display, and lifecycle callbacks
- Runtime personalized-ad preference
- Explicit privacy switches; every optional device-data permission defaults to `false`
- Prebuilt debug and release bridge AARs

## Requirements

- Godot 4 with Android export templates
- Android min SDK 21 or newer
- Java 17 when rebuilding the bridge
- TapADN / Dirichlet media credentials and ad-space IDs
- Dirichlet Android SDK 5.3.0.1

## Installation

1. Copy this repository to `res://addons/GodotTapTapAdSDK` in your Godot project.
2. Open **Project > Project Settings > Plugins** and enable `GodotTapTapAdSDK`.
3. Enable **Gradle Build** in the Android export preset.
4. Make sure the Android export can resolve Maven dependencies. The plugin requests `com.squareup.okhttp3:okhttp:3.12.1` during export.
5. Obtain a media ID, media name, media key, Tap client ID, and ad-space IDs from the TapADN / Dirichlet console.

The plugin registers the `GodotTapTapAd` autoload automatically. Its native singleton is available only in Android exports; calls from the editor or another platform fail safely.

## Privacy-first initialization

Show your privacy policy and wait for explicit user consent before initializing any advertising SDK. Do not store production credentials directly in source control.

```gdscript
func initialize_ads_after_consent() -> void:
    var accepted := GodotTapTapAd.initialize(
        123456,
        "YOUR_MEDIA_NAME",
        "YOUR_MEDIA_KEY",
        "YOUR_TAP_CLIENT_ID",
        false, # personalized ads
        false, # SDK debug logs
        {
            # Enable an option only when your privacy policy discloses it
            # and the user has granted the required consent/permission.
            "location": false,
            "phone_state": false,
            "wifi_state": false,
            "mac_address": false,
            "bssid": false,
            "ssid": false,
            "sensor": false,
            "write_external": false,
            "app_list": false,
            "android_id": false,
            "shake": false,
        }
    )

    if not accepted:
        push_error("Advertising is unavailable on this platform.")
```

`initialize()` returning `true` means the native initialization request was dispatched. Listen for `initialized` or `initialization_failed(message)` for the final result.

## Rewarded video

```gdscript
func _ready() -> void:
    GodotTapTapAd.reward_loaded.connect(_on_reward_loaded)
    GodotTapTapAd.reward_verified.connect(_on_reward_verified)
    GodotTapTapAd.reward_load_failed.connect(_on_reward_load_failed)


func request_rewarded_ad() -> void:
    GodotTapTapAd.load_reward_video(
        1000001,       # ad-space ID
        "coins",      # reward name
        100,           # reward amount
        "PLAYER_ID",  # optional user ID
        ""             # optional custom data
    )


func _on_reward_loaded() -> void:
    GodotTapTapAd.show_reward_video()


func _on_reward_verified(verified: bool, amount: int, reward_name: String, code: int, message: String) -> void:
    if verified:
        grant_reward(reward_name, amount)


func _on_reward_load_failed(code: int, message: String) -> void:
    push_warning("Rewarded ad failed: %s (%d)" % [message, code])
```

Grant client-side rewards only when `reward_verified` reports `verified == true`. For valuable rewards, use Dirichlet server-to-server verification rather than trusting the client alone.

## Interstitial

```gdscript
GodotTapTapAd.interstitial_loaded.connect(
    func(): GodotTapTapAd.show_interstitial()
)
GodotTapTapAd.interstitial_load_failed.connect(
    func(code: int, message: String):
        push_warning("Interstitial failed: %s (%d)" % [message, code])
)

GodotTapTapAd.load_interstitial(1000002)
```

Call `dispose_ads()` when the ads are no longer needed. Banner, splash, and feed formats are not included because their Android `View` lifecycle requires a dedicated Godot UI container.

## API

### Methods

| Method | Description |
| --- | --- |
| `initialize(...) -> bool` | Initializes the native SDK after privacy consent. |
| `set_personalized_ads_enabled(enabled)` | Updates the personalized-ad preference. |
| `load_reward_video(...)` | Loads a rewarded-video placement. |
| `is_reward_video_ready() -> bool` | Reports whether a rewarded video can be shown. |
| `show_reward_video()` | Displays the loaded rewarded video. |
| `load_interstitial(space_id)` | Loads an interstitial placement. |
| `is_interstitial_ready() -> bool` | Reports whether an interstitial can be shown. |
| `show_interstitial()` | Displays the loaded interstitial. |
| `dispose_ads()` | Releases currently held ad objects. |

### Signals

- Initialization: `initialized`, `initialization_failed(message)`
- Reward loading: `reward_loaded`, `reward_load_failed(code, message)`
- Reward lifecycle: `reward_shown`, `reward_closed`, `reward_video_completed`, `reward_video_skipped`, `reward_video_error`, `reward_clicked`
- Reward result: `reward_verified(verified, amount, reward_name, code, message)`
- Interstitial loading: `interstitial_loaded`, `interstitial_load_failed(code, message)`
- Interstitial lifecycle: `interstitial_shown`, `interstitial_closed`, `interstitial_error`, `interstitial_clicked`

## Rebuilding the Android bridge

The Gradle project expects Godot's debug template library at:

```text
android/build/libs/debug/godot-lib.template_debug.aar
```

From the plugin directory, run:

```powershell
gradle copyPluginAars
```

The generated bridge libraries are copied to:

```text
bin/debug/GodotTapTapAdSDK-debug.aar
bin/release/GodotTapTapAdSDK-release.aar
```

## Upstream documentation

- [Dirichlet Android SDK guide](https://ssp.dirichlet.cn/docs/dirichlet-sdk/sdk-guide/)
- [Bundled SDK provenance](UPSTREAM.md)

## License

The original plugin source is released under the [MIT License](LICENSE).

The bundled `libs/dirichlet_ad_5.3.0.1.aar` is third-party software from TapADN / Dirichlet. It is not covered by the MIT License; its use and redistribution remain subject to the upstream provider's terms.
