# Tài liệu tích hợp AdMob + Adjust vào Flutter

Hướng dẫn nhúng **quảng cáo AdMob** (qua `flutter_ads`) và **đo lường Adjust** (qua `adjust-helper`)
vào một app Flutter. Dùng cho người mới tiếp nhận, làm tuần tự từ trên xuống là chạy.

> **Phạm vi:** Adjust hiện **chỉ hoạt động Android** (lib này là Android-only).
> iOS chạy được AdMob nhưng Adjust iOS **chưa wire** (xem [§7](#7-ios--lưu-ý-adjust-chưa-wire)).

---

## 0. Kiến trúc tổng quan

```
        Flutter (Dart)                         Native
┌──────────────────────────┐      ┌─────────────────────────────────┐
│ MyAds (flutter_ads)       │◄────►│ google_mobile_ads + mediation    │  ← hiển thị ad
│  • initialize / show ad   │      │ (Facebook, Unity, IronSource…)   │
│  • events (onPaid…)        │      └─────────────────────────────────┘
│            │ revenue                                                    
│            ▼                       MethodChannel "com.adjust.sdk/api"   
│ AdjustUtil (Dart wrapper) │◄────►│ AdjustChannel (adjust-helper)    │  ← đo lường
│  • initialize             │      │   → Adjust SDK (com.adjust.sdk)  │
│  • trackAdRevenue…        │      └─────────────────────────────────┘
└──────────────────────────┘
```

**2 thành phần độc lập:**

| Thành phần | Repo | Cách lấy | Nền tảng |
|---|---|---|---|
| `flutter_ads` (AdMob) | `github.com/manhpd98/flutter_ads` (private) | git dependency + token | Android + iOS |
| `adjust-helper` (Adjust) | `github.com/manhpd98/adjust-helper` (public) | JitPack | Android |
| `AdjustUtil` (Dart wrapper) | nằm trong app (copy ở [§5](#5-copy-lớp-dart-adjust)) | copy file | — |

Adjust được nối vào AdMob ở chỗ: **mỗi lần ad sinh doanh thu (onPaid) → đẩy revenue sang Adjust**.

---

## 1. SETUP SKILL — Tài khoản & khóa AdMob

1. Vào **https://apps.admob.com** → tạo app (hoặc link app có sẵn).
2. Lấy **App ID** dạng `ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY` (có dấu `~`).
3. Tạo **Ad Units** cần dùng → mỗi cái cho 1 **Ad Unit ID** dạng `ca-app-pub-...../ZZZZZZZZZZ` (dấu `/`):
   - Banner, Interstitial, Native, App Open, Rewarded (tùy nhu cầu).
4. **ID test của Google** (dùng khi dev, KHÔNG bị khóa tài khoản):
   - App ID test: `ca-app-pub-3940256099942544~3347511713`
   - Interstitial: `ca-app-pub-3940256099942544/1033173712`
   - Native: `ca-app-pub-3940256099942544/2247696110`
   - App Open: `ca-app-pub-3940256099942544/9257395921`
   - Banner: `ca-app-pub-3940256099942544/6300978111`
   - Rewarded: `ca-app-pub-3940256099942544/5224354917`
5. (Tùy chọn) **Mediation**: bật network phụ (Facebook/Meta, Unity, IronSource, Mintegral, Pangle) trong AdMob → cần thêm SDK adapter ở [§3](#3-cấu-hình-native--android) & [§7](#7-ios--lưu-ý-adjust-chưa-wire).

> ⚠️ Khi build release **bắt buộc** thay hết ID test bằng ID thật, nếu không sẽ không ra tiền.

---

## 2. SETUP SKILL — Tài khoản & khóa Adjust

1. Vào **https://dash.adjust.com** → tạo app → lấy **App Token** (10–12 ký tự, ví dụ `j82uqfqs2igw`).
2. Tạo các **Event** trong dashboard → mỗi event có **Event Token**. Khuyến nghị tạo:
   - **Ad revenue** (doanh thu quảng cáo) — _bắt buộc nếu muốn đo ROAS_.
   - **Ad impression** — token cho `impressionToken`.
   - **IAP revenue** (mua hàng) — 1 token / sản phẩm, + 1 token tổng.
   - **event80** (tùy chiến lược tracking giá trị 80%).
3. **API Token** (cho tính năng "full ads gating"): Adjust → **Menu → All Settings → (hoặc Profile) → API Token / S2S token**. Token này cho lib hỏi API `inspect_device` để biết user đến từ network nào.
4. **Environment**:
   - `sandbox` khi dev/test (event hiện ở **Testing Console** của Adjust).
   - `production` khi release.
5. **(iOS) Signature SDK**: Adjust cấp `AdjustSigSdk.xcframework` để chống gian lận — đặt vào `ios/libs/` (xem [§7](#7-ios--lưu-ý-adjust-chưa-wire)).

> Ghi nhớ tối thiểu để chạy Android: **App Token** + **Ad revenue event token**. Các token khác thêm dần.

---

## 3. Thêm dependency

### 3.1 `pubspec.yaml` — AdMob (`flutter_ads`, private → cần token)

```yaml
dependencies:
  flutter_ads:
    git:
      url: https://github.com/manhpd98/flutter_ads.git
      ref: 1.0.0
  shared_preferences: ^2.2.2   # AdjustUtil cần
```

> `flutter_ads` là repo **private** → cần **quyền đọc**. Cách dùng:
> - **Khuyên dùng:** được chủ repo mời làm collaborator → `gh auth login` (hoặc git credential keychain) → giữ URL sạch như trên, không nhúng token.
> - **Hoặc** nhúng token đọc vào URL: `https://x-access-token:<PAT>@github.com/manhpd98/flutter_ads.git`
>   (token fine-grained, Contents: Read-only, tạo tại https://github.com/settings/personal-access-tokens/new).
>   ⚠️ Token nhúng sẽ bị commit — **đừng** đẩy lên repo/doc public (GitHub secret-scanning sẽ thu hồi token).

### 3.2 Android — Adjust (`adjust-helper`, public qua JitPack)

`android/build.gradle` (project-level), thêm repo JitPack:

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }   // ← cho adjust-helper
    }
}
```

`android/app/build.gradle`:

```gradle
dependencies {
    implementation 'com.github.manhpd98:adjust-helper:1.0.0'   // kéo kèm com.adjust.sdk:adjust-android:5.4.0
    // mediation tùy chọn:
    // implementation 'com.google.ads.mediation:facebook:6.21.0.2'
}
```

### 3.3 Lớp Dart Adjust

Copy thư mục `lib/adjust/` (mã ở [§5](#5-copy-lớp-dart-adjust)) vào project của bạn.

---

## 4. Cấu hình Native — Android

### 4.1 `AndroidManifest.xml`

```xml
<manifest ...>
  <uses-permission android:name="android.permission.INTERNET"/>
  <application ...>
    <!-- AdMob App ID (THAY bằng của bạn; đây là ID test của Google) -->
    <meta-data
        android:name="com.google.android.gms.ads.APPLICATION_ID"
        android:value="ca-app-pub-3940256099942544~3347511713"/>
  </application>
</manifest>
```

### 4.2 `MainActivity.kt` — gắn AdjustChannel + đăng ký Native Ad Factory

```kotlin
import com.manhpd98.adjust_helper.AdjustChannel       // ← package mới của adjust-helper
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugins.googlemobileads.GoogleMobileAdsPlugin

class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        // 1) BẮT BUỘC cho Adjust: mở channel "com.adjust.sdk/api"
        AdjustChannel(this, flutterEngine.dartExecutor.binaryMessenger)

        // 2) Đăng ký các native ad factory (nếu dùng native ad) — tên phải khớp factoryId bên Dart
        GoogleMobileAdsPlugin.registerNativeAdFactory(flutterEngine, "homeNativeAd", HomeNativeAd(context))
        // … các factory khác …
    }
}
```

> Class `AdjustChannel` nằm trong `adjust-helper` (package `com.manhpd98.adjust_helper`). Chỉ cần khởi tạo 1 dòng — toàn bộ logic Adjust nằm trong lib.

---

## 5. Copy lớp Dart Adjust

Tạo các file sau trong app (vd `lib/adjust/`). Đây là wrapper mỏng gọi qua MethodChannel.

### 5.1 `adjust/model/adjust_token.dart`
```dart
import 'dart:io';

class AdjustToken {
  AdjustToken({this.androidToken, this.iosToken});
  final String? androidToken;
  final String? iosToken;
  String? get platformToken => Platform.isAndroid ? androidToken : iosToken;
}
```

### 5.2 `adjust/model/fullads_option.dart`
```dart
/// Quy tắc "full ads gating": quyết định bật quảng cáo dày khi user đến từ network nào.
class FullAdsOption {
  const FullAdsOption({
    this.useNull = false,          // network = null
    this.useEmpty = false,         // network = ""
    this.useUnAttributed = false,  // user organic / chưa attribute
    this.maxFull = false,          // luôn bật full
  });
  factory FullAdsOption.fromJson(Map<String, dynamic> m) => FullAdsOption(
        useNull: m['useNull'] as bool? ?? false,
        useEmpty: m['useEmpty'] as bool? ?? false,
        useUnAttributed: m['useUnAttributed'] as bool? ?? false,
        maxFull: m['maxFull'] as bool? ?? false,
      );
  final bool useNull, useEmpty, useUnAttributed, maxFull;
  Map<String, dynamic> toJson() => {
        'useNull': useNull, 'useEmpty': useEmpty,
        'useUnAttributed': useUnAttributed, 'maxFull': maxFull,
      };
}
```

### 5.3 `adjust/model/ad_option.dart`
```dart
import 'dart:io';

typedef FullAdCallback = void Function(
    bool isFullAd, String? network, bool fromCache, bool fromLib, bool fromApi);

abstract class PlatformAdOption {
  PlatformAdOption({this.impressionToken, this.fullAdCallback});
  final String? impressionToken;
  final FullAdCallback? fullAdCallback;
}
class AndroidAdOptions extends PlatformAdOption {
  AndroidAdOptions({super.impressionToken, super.fullAdCallback});
}
class IOSAdOptions extends PlatformAdOption {
  IOSAdOptions({super.impressionToken, super.fullAdCallback});
}
class AdOptions {
  AdOptions({this.androidAdOptions, this.iosAdOptions});
  final AndroidAdOptions? androidAdOptions;
  final IOSAdOptions? iosAdOptions;
  String? get impressionToken =>
      (Platform.isAndroid ? androidAdOptions : iosAdOptions)?.impressionToken;
  FullAdCallback? get fullAdCallback =>
      (Platform.isAndroid ? androidAdOptions : iosAdOptions)?.fullAdCallback;
}
```

### 5.4 `adjust/model/iap_option.dart`
```dart
import 'dart:io';

abstract class PlatformIapOption {
  PlatformIapOption({this.productRevenueTokens, this.totalRevenueToken});
  /// key = productId, value = event token. Ví dụ {'com.app.weekly': 'abc123'}
  final Map<String, String>? productRevenueTokens;
  final String? totalRevenueToken;
}
class AndroidIapOptions extends PlatformIapOption {
  AndroidIapOptions({super.productRevenueTokens, super.totalRevenueToken});
}
class IOSIapOptions extends PlatformIapOption {
  IOSIapOptions({super.productRevenueTokens, super.totalRevenueToken});
}
class IapOptions {
  IapOptions({this.androidOptions, this.iosOptions});
  final AndroidIapOptions? androidOptions;
  final IOSIapOptions? iosOptions;
  Map<String, String>? get productRevenueTokens => Platform.isAndroid
      ? androidOptions?.productRevenueTokens : iosOptions?.productRevenueTokens;
  String? get totalRevenueToken => Platform.isAndroid
      ? androidOptions?.totalRevenueToken : iosOptions?.totalRevenueToken;
  Map<String, dynamic> toJson() => {
        'productRevenueTokens': productRevenueTokens,
        'totalRevenueToken': totalRevenueToken,
      };
}
```

### 5.5 `adjust/model/adjust_event.dart`
```dart
class AdjustEvent {
  AdjustEvent(this.eventToken, {this.revenue, this.currency, this.productId});
  final String eventToken;
  final num? revenue;
  final String? currency;
  final String? productId;
  Map<String, dynamic> toMap() => {
        'eventToken': eventToken, 'revenue': revenue,
        'currency': currency, 'productId': productId,
      };
}
```

### 5.6 `adjust/adjust_util.dart`
```dart
import 'package:flutter/services.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'model/ad_option.dart';
import 'model/adjust_event.dart';
import 'model/adjust_token.dart';
import 'model/fullads_option.dart';
import 'model/iap_option.dart';

enum AdjustEnvironment { production, sandbox }

class AdjustUtil {
  AdjustUtil._();
  static final AdjustUtil instance = AdjustUtil._();
  static const MethodChannel _channel = MethodChannel('com.adjust.sdk/api');

  bool isInitialized = false;
  bool _handlerSet = false;
  FullAdCallback? _fullAdCallback;
  SharedPreferences? _prefs;

  Future<void> initialize({
    required AdjustEnvironment environment,
    required AdjustToken appToken,
    required FullAdsOption fullAdsOption,
    AdOptions? adOptions,
    IapOptions? iapOptions,
    String? apiToken,
  }) async {
    _fullAdCallback = adOptions?.fullAdCallback;
    _prefs = await SharedPreferences.getInstance();
    final cached = _prefs?.getBool('isFullAds');
    if (cached != null) {
      adOptions?.fullAdCallback?.call(cached, null, true, false, false);
    } else {
      _ensureHandler();
    }
    if (isInitialized) return;
    try {
      await _channel.invokeMethod('initSdk', {
        'environment': environment.name,
        'appToken': appToken.platformToken,
        'apiToken': apiToken,
        'fullAdsOption': fullAdsOption.toJson(),
        'iapOptions': iapOptions?.toJson(),
        'impressionToken': adOptions?.impressionToken,
      });
    } on Exception catch (e) {
      // ignore: avoid_print
      print('Adjust init error: $e');
    }
    isInitialized = true;
  }

  Future<void> trackAdRevenue({required double value, required String currencyCode}) =>
      _invoke('trackAdRevenue', {'value': value, 'currencyCode': currencyCode});

  Future<void> trackImpressionEvent({required double value, required String currencyCode}) =>
      _invoke('trackImpressionEvent', {'value': value, 'currencyCode': currencyCode});

  Future<void> trackEvent(AdjustEvent event) => _invoke('trackEvent', event.toMap());

  Future<void> trackSubscriptionRevenue(
          {required String productId, required String price, required String currencyCode}) =>
      _invoke('trackSubscriptionRevenue',
          {'productId': productId, 'price': price, 'currencyCode': currencyCode});

  Future<void> trackTotalIapRevenue(
          {required String productId, required String price, required String currencyCode}) =>
      _invoke('trackTotalIapRevenue',
          {'productId': productId, 'price': price, 'currencyCode': currencyCode});

  Future<void> _invoke(String m, Map<String, dynamic> args) async {
    try {
      await _channel.invokeMethod(m, args);
    } on Exception catch (e) {
      // ignore: avoid_print
      print('Adjust $m error: $e');
    }
  }

  void _ensureHandler() {
    if (_handlerSet) return;
    _handlerSet = true;
    _channel.setMethodCallHandler((call) async {
      if (call.method == 'onFullAdCallback') {
        final p = Map<String, dynamic>.from(call.arguments);
        final isFullAd = p['isFullAds'] as bool?;
        if (isFullAd != null) {
          _prefs?.setBool('isFullAds', isFullAd);
          _fullAdCallback?.call(isFullAd, p['network'] as String?,
              p['fromCache'] as bool? ?? false,
              p['fromLib'] as bool? ?? false, p['fromApi'] as bool? ?? false);
        }
      }
    });
  }
}
```

---

## 6. Wiring — khởi tạo & nối doanh thu

### 6.1 Khởi tạo Adjust (lúc app khởi động)
```dart
Future<void> initAdjust() {
  final env = kReleaseMode ? AdjustEnvironment.production : AdjustEnvironment.sandbox;
  return AdjustUtil.instance.initialize(
    environment: env,
    appToken: AdjustToken(androidToken: '<ADJUST_APP_TOKEN>', iosToken: ''),
    fullAdsOption: const FullAdsOption(useUnAttributed: true),
    apiToken: '<ADJUST_API_TOKEN>',          // cho fullAds gating (có thể null)
    adOptions: AdOptions(
      androidAdOptions: AndroidAdOptions(
        impressionToken: '<ADJUST_IMPRESSION_EVENT_TOKEN>',
        fullAdCallback: (isFullAd, network, fromCache, fromLib, fromApi) {
          // bật/tắt quảng cáo dày theo isFullAd
        },
      ),
    ),
  );
}
```

### 6.2 Khởi tạo AdMob (`flutter_ads`)
```dart
import 'package:flutter_ads/ads_flutter.dart';

await MyAds.instance.initialize(
  navigatorKey: appNavigatorKey,
  interIntervalInSeconds: 30,        // giãn cách giữa 2 inter
  reloadNativeAdWhenClicked: true,
  admobConfiguration: RequestConfiguration(testDeviceIds: ['<TEST_DEVICE_ID>']),
);
```

### 6.3 **NỐI doanh thu ad → Adjust** (mấu chốt)
```dart
MyAds.instance.events.listen((event) {
  if (event.status.isPaid && event.valueMicros != null && event.currencyCode != null) {
    final value = event.valueMicros! / 1000000;        // micros → đơn vị tiền
    AdjustUtil.instance.trackAdRevenue(value: value, currencyCode: event.currencyCode!);
    AdjustUtil.instance.trackImpressionEvent(value: value, currencyCode: event.currencyCode!);
    // (tùy chọn) gửi event giá trị 80%
    // AdjustUtil.instance.trackEvent(AdjustEvent('<EVENT80_TOKEN>', revenue: value * 0.8, currency: event.currencyCode));
  }
});
```

### 6.4 IAP → Adjust (nếu có mua hàng)
```dart
AdjustUtil.instance.trackSubscriptionRevenue(
  productId: 'com.app.weekly', price: '1.99', currencyCode: 'USD');
AdjustUtil.instance.trackTotalIapRevenue(
  productId: 'com.app.weekly', price: '1.99', currencyCode: 'USD');
```

---

## 7. iOS — lưu ý Adjust CHƯA wire

- **AdMob iOS chạy bình thường** qua `flutter_ads`: cấu hình `GADApplicationIdentifier` trong `Info.plist`,
  thêm mediation pod vào `Podfile`, đăng ký native ad factory trong `AppDelegate.swift`.
- **Adjust iOS hiện CHƯA hoạt động**: `adjust-helper` là lib Android. Bên iOS mới chỉ có `AdjustSigSdk.xcframework`,
  **chưa có handler Swift** cho channel `com.adjust.sdk/api`, và `iosToken` để rỗng.
  → Mọi lệnh `AdjustUtil.*` trên iOS chạy vào `notImplemented` (được nuốt lỗi, app không crash).
- Muốn đo Adjust iOS: cần thêm Adjust iOS SDK + 1 class Swift xử lý channel `com.adjust.sdk/api`
  (port từ `AdjustChannel`/`AdjustBridge` của Android). Đây là việc riêng, chưa nằm trong gói này.

---

## 8. ✅ TỰ KIỂM TRA (self-check)

### 8.1 Checklist build
- [ ] `flutter pub get` không lỗi (token `flutter_ads` đúng quyền read).
- [ ] `android/build.gradle` có `maven { url 'https://jitpack.io' }`.
- [ ] `android/app/build.gradle` có `com.github.manhpd98:adjust-helper:1.0.0`.
- [ ] `MainActivity.kt` import `com.manhpd98.adjust_helper.AdjustChannel` và gọi `AdjustChannel(this, …)`.
- [ ] `AndroidManifest.xml` có meta-data `com.google.android.gms.ads.APPLICATION_ID` + quyền `INTERNET`.
- [ ] `flutter analyze` không còn `error` (chỉ warning/info là OK).
- [ ] `flutter build apk --debug` thành công (lần đầu sẽ kéo Adjust SDK về).

### 8.2 Kiểm tra Adjust nhận event (Android)
1. Đặt `environment: AdjustEnvironment.sandbox`.
2. Mở **Adjust Dashboard → Testing Console**, nhập **GPS ADID** của máy test
   (lấy bằng: cài "Device ID" app, hoặc xem log Adjust).
3. Chạy app, xem **logcat** lọc theo tag Adjust:
   ```bash
   adb logcat -s Adjust
   ```
   Phải thấy: `Adjust SDK initialised`, rồi khi có doanh thu ad là dòng gửi event/revenue.
4. Trong Testing Console phải thấy **Session** + **Ad revenue event** hiện lên (trễ ~ vài giây).

### 8.3 Kiểm tra channel & revenue đã nối
```bash
# Channel adjust phải được mở (không có log "MissingPluginException(com.adjust.sdk/api)")
adb logcat | grep -i "adjust\|MissingPlugin"
```
- Bật **test ads** → chờ AdMob bắn `onPaid` → log `trackAdRevenue` chạy.
- Nếu thấy `MissingPluginException: com.adjust.sdk/api` → **chưa gọi `AdjustChannel(...)`** trong `MainActivity` hoặc sai package import.

### 8.4 Kiểm tra AdMob hiển thị
- Dùng **ID test** của Google ([§1.4](#1-setup-skill--tài-khoản--khóa-admob)) → ad test luôn fill.
- Thêm thiết bị vào `testDeviceIds` để không bị tính click ảo.

### 8.5 Trước khi release
- [ ] Đổi `environment` → `production`.
- [ ] Thay **hết** ID test bằng ID thật (AdMob App ID + ad unit + Adjust token).
- [ ] Kiểm tra event lên đúng app **production** trên dashboard Adjust.

---

## 9. Cập nhật `adjust-helper` về sau

JitPack coi mỗi **version là bất biến và cache vĩnh viễn**. Khi sửa code lib:
- **Cách chuẩn:** tăng version (tag `1.0.1`, `1.0.2`…) rồi đổi số trong `build.gradle`.
- Nếu **bắt buộc** giữ nguyên số version: phải xoá build cũ trên JitPack trước
  (https://jitpack.io/#manhpd98/adjust-helper → đăng nhập GitHub → xoá version) rồi mới re-tag.

---

## 10. Lỗi thường gặp

| Triệu chứng | Nguyên nhân / cách xử lý |
|---|---|
| `MissingPluginException(com.adjust.sdk/api)` | Chưa gọi `AdjustChannel(this, …)` trong `MainActivity`, hoặc sai import package. |
| `Could not resolve com.github.manhpd98:adjust-helper` | Thiếu `maven { url 'https://jitpack.io' }`; hoặc version chưa build xong trên JitPack. |
| `pub get` 404 / authentication failed | Token `flutter_ads` sai/hết hạn/không có quyền Contents:Read repo đó. |
| Adjust không thấy event | Sai `appToken`; chưa ở `sandbox`; máy chưa add vào Testing Console; thiếu quyền `INTERNET`. |
| Build kéo sai code lib | JitPack cache version cũ — xem [§9](#9-cập-nhật-adjust-helper-về-sau). |
| iOS không đo Adjust | Đúng như thiết kế hiện tại — Adjust iOS chưa wire ([§7](#7-ios--lưu-ý-adjust-chưa-wire)). |
