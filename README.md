# AppPulse  — Production-Grade Android Analytics SDK

[![Kotlin](https://img.shields.io/badge/Kotlin-2.0.21-purple.svg)](https://kotlinlang.org)
[![Android Min SDK](https://img.shields.io/badge/Min%20SDK-24-blue.svg)](https://developer.android.com)
[![Target SDK](https://img.shields.io/badge/Target%20SDK-35-green.svg)](https://developer.android.com)
[![WorkManager](https://img.shields.io/badge/WorkManager-2.9.1-orange.svg)](https://developer.android.com/topic/libraries/architecture/workmanager)
[![Room](https://img.shields.io/badge/Room-2.6.1-red.svg)](https://developer.android.com/training/data-storage/room)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

**AppPulse** is a lightweight, offline-first, production-ready Android Analytics SDK written in Kotlin. It provides robust client-side event tracking, SQLite persistence via Room, intelligent batching, guaranteed delivery via Android Jetpack WorkManager, and retry mechanics with exponential backoff.

Accompanied by a modern Material Design demo application and a Python mock server for end-to-end testing and demonstration.

---

##  Architecture Overview

The SDK follows clean architecture with strict internal encapsulation:

```
┌────────────────────────────────────────────────────────┐
│                   Demo / Host Application              │
│    (Calls public API: initialize, track, flush, etc.)   │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│                 AppPulse Public API                    │
│   • AppPulse.kt                                        │
│   • AppPulseConfig.kt                                  │
│   • AppPulseError.kt                                   │
│   • EventCallback.kt                                   │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│             Internal Orchestrator (Manager)            │
│   • AppPulseManager.kt                                 │
│   • EventValidator.kt                                  │
└───────────────────────────┬────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
┌───────────────────────────┐ ┌──────────────────────────┐
│     Local Data Source     │ │    Remote Data Source    │
│  • Room Database          │ │  • Retrofit 2            │
│  • AnalyticsEventEntity   │ │  • OkHttp 3              │
│  • AnalyticsEventDao      │ │  • Moshi JSON Converter  │
└─────────────┬─────────────┘ └───────────┬──────────────┘
              │                           │
              └─────────────┬─────────────┘
                            ▼
┌────────────────────────────────────────────────────────┐
│         Background Sync Layer (WorkManager)            │
│   • EventSyncWorker (CoroutineWorker)                  │
│   • SyncScheduler (Periodic & One-shot constraints)    │
│   • Exponential backoff & battery/network awareness    │
└────────────────────────────────────────────────────────┘
```

### Key Design Highlights
- **Strict Public API Encapsulation**: Host apps can only access `AppPulse`, `AppPulseConfig`, `EventCallback`, and `AppPulseError`. All database entities, DAOs, Retrofit interfaces, and WorkManager helpers are marked `internal`.
- **Zero Host App Pollution**: Proguard rules ensure internal classes are stripped or obfuscated, keeping the SDK bundle tiny (~124 KB AAR).
- **Offline-First & Resilient**: Events are saved to Room instantly on the IO thread. Network failures queue events for automatic background delivery when network constraints are satisfied.
- **Fail-Safe Operation**: SDK crashes or exceptions never crash the host application.

---

##  Project Structure

```
aiandroid/
├── sdk/                         # The Reusable Android Analytics SDK
│   ├── src/main/kotlin/com/apppulse/sdk/
│   │   ├── AppPulse.kt          # Public singleton entry point
│   │   ├── AppPulseConfig.kt    # Immutable configuration
│   │   ├── AppPulseError.kt     # Sealed class error hierarchy
│   │   ├── EventCallback.kt     # Delivery status callbacks
│   │   └── internal/            # Encapsulated internal implementation
│   │       ├── db/              # Room database, entity, and DAO
│   │       ├── model/           # Internal models & batch payloads
│   │       ├── network/         # Retrofit API, Moshi adapters, OkHttp
│   │       ├── repository/      # Event repository implementation
│   │       ├── sync/            # WorkManager CoroutineWorker & Scheduler
│   │       └── validation/      # Event name and property validation
│   └── src/test/                # Comprehensive unit tests (Truth, MockWebServer)
│
├── demo-app/                    # Feature-Rich Showcase Application
│   └── src/main/kotlin/com/apppulse/demo/
│       ├── AppPulseDemoApp.kt   # Application class initializing AppPulse
│       ├── MainActivity.kt      # Interactive tracking dashboard & event tester
│       └── EventHistoryAdapter  # RecyclerView for local event monitoring
│
├── mock-server/                 # Local Test Server
│   └── server.py                # Python Flask mock server accepting analytics batches
│
├── plan.md                      # Detailed architectural design & technical roadmap
└── build.gradle.kts             # Gradle multi-module root build script
```

---

##  Quick Start

### 1. Initialize the SDK
Initialize `AppPulse` once in your `Application` class:

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val config = AppPulseConfig.Builder()
            .apiKey("your_api_key_here")
            .apiEndpoint("https://analytics.yourdomain.com/")
            .batchSize(30)
            .flushIntervalSeconds(30)
            .maxRetries(5)
            .debugMode(BuildConfig.DEBUG)
            .build()

        AppPulse.initialize(this, config)
    }
}
```

### 2. Track Custom Events
Track any user action with optional typed properties:

```kotlin
// Simple event
AppPulse.track("user_signup")

// Event with typed properties
AppPulse.track(
    eventName = "purchase_completed",
    properties = mapOf(
        "order_id" to "ORD-9482",
        "total_amount" to 49.99,
        "items_count" to 3,
        "is_subscriber" to true
    )
)
```

### 3. Track with Callbacks
```kotlin
AppPulse.track("checkout_click", mapOf("cart_size" to 2), object : EventCallback {
    override fun onSuccess(eventId: String) {
        Log.d("AppPulse", "Event recorded successfully: $eventId")
    }

    override fun onError(error: AppPulseError) {
        Log.e("AppPulse", "Event recording failed: ${error.message}")
    }
})
```

### 4. Manual Sync & Purge
```kotlin
// Force an immediate sync cycle via WorkManager
AppPulse.forceSync()

// Purge cached events
AppPulse.deleteAllEvents()
```

---

##  Running the Mock Server & Demo App

### Start Mock Server
```bash
cd mock-server
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Linux/macOS:
source .venv/bin/activate

pip install flask
python server.py
```
The server will start at `http://127.0.0.1:5000/`.

### Run the Demo App
Connect an Android device or launch an emulator, then:
```bash
./gradlew :demo-app:installDebug
```

---

##  Testing & Verification

The SDK contains comprehensive unit tests covering:
- **EventValidatorTest**: Name length, regex, property type limits, null safety.
- **AppPulseConfigTest**: Default values, copy ergonomics, boundary constraints.
- **RemoteDataSourceTest**: HTTP 200, 400, 401, 429, 500 status code mapping using OkHttp `MockWebServer`.
- **AppPulseErrorTest**: Exhaustive sealed class hierarchy verification.

To run all unit tests:
```bash
./gradlew :sdk:testDebugUnitTest
```

---

## 📄 License

```
Copyright 2026 Saurabh Tiwari

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
```
