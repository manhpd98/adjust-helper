# adjust-helper

Android helper bọc **Adjust SDK** (`com.adjust.sdk:adjust-android`) cho Flutter qua
MethodChannel `com.adjust.sdk/api`. Self-hosted.

## Cài qua JitPack

```gradle
// android/build.gradle (project) — repositories:
maven { url 'https://jitpack.io' }

// android/app/build.gradle — dependencies:
implementation 'com.github.manhpd98:adjust-helper:1.0.0'
```

## Nội dung

- Module thư viện: **`adjust-kit`** (package `com.adjust.helper`)
- `AdjustChannel` xử lý channel `com.adjust.sdk/api`: `initSdk`, `trackEvent`,
  `trackAdRevenue`, `trackImpressionEvent`, `trackSubscriptionRevenue`,
  `trackTotalIapRevenue`, `isInitialized`
- `AdjustBridge` gọi Adjust SDK thật + logic "full ads gating" theo network.
- Module `:app` chỉ là app demo (không được publish).
