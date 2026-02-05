---
title: Understanding iCloud Sync Between Mac and iOS Apps - Principles and Implementation
date: 2026-01-18 11:44
excerpt: I synchronized data between Mac and iOS without using a network.
coverImage:
categories:
  - iOS
tags:
  - iOS
  - CloudKit
author: Geunil Park
featured: false
---

## 1. Overview
The `Meku` project currently in development has successfully implemented data synchronization between Mac App and iOS App. This synchronization is achieved through the combination of Apple's latest data management framework **SwiftData** and the backend service **CloudKit**.

Without developers writing separate synchronization logic or networking code, data automatically syncs through Apple platform's internal mechanisms simply by defining the data model and configuring the appropriate settings.

---

## 2. Core Technologies

### 1) SwiftData
- Apple's latest data management framework unveiled at WWDC23.
- Built on Core Data, but provides a much more concise and intuitive API leveraging Swift language features.
- Data models can be easily defined using the `@Model` macro.

### 2) CloudKit
- A framework for interacting with Apple's iCloud servers.
- Stores app data in iCloud and enables access to that data across all user devices.
- Supports various data sharing methods including Public, Private, and Shared Databases.

### 3) NSPersistentCloudKitContainer (Internal Operation)
- SwiftData internally utilizes Core Data's `NSPersistentCloudKitContainer`.
- This container acts as a **synchronization engine** managing data mirroring between the local database (SQLite) and iCloud (CloudKit).

---

## 3. Sync Mechanism

Data synchronization operates on the following principles:

1. **Local Storage**: When data is created or modified in the app, SwiftData saves it to local storage (SQLite).
2. **Change Detection**: `NSPersistentCloudKitContainer` detects changes in local storage.
3. **iCloud Upload**: Changed data is automatically uploaded to CloudKit Private Database in the background.
4. **Remote Notification (Push Notification)**: When data changes in iCloud, CloudKit sends a Silent Push notification to other devices (e.g., Mac or iPhone) using the same account, indicating "data has been changed."
5. **Data Fetch**: The device receiving the notification downloads the changed data from CloudKit and merges it into its local storage.
6. **UI Update**: SwiftUI views bound to SwiftData (`@Query`, etc.) automatically refresh to show users the latest data.

All of these processes are handled automatically at the framework level **without developers writing separate synchronization code**.

---

## 4. Current Implementation in `Meku` Project

The current project has all the necessary configurations for integrating SwiftData with CloudKit.

### 1) Data Model Definition (`Item.swift`)
The data model is defined using SwiftData's `@Model` macro. This model has a structure that can be automatically converted to Core Data entities and CloudKit records.

```swift
// Meku/Item.swift
import SwiftData

@Model
final class Item {
    var timestamp: Date = Date()
    var platform: String = ""

    init(timestamp: Date, platform: String) {
        self.timestamp = timestamp
        self.platform = platform
    }
}
```

### 2) ModelContainer Setup (`MekuApp.swift`)
The `ModelContainer` is configured at the app's entry point. By default, when persistent storage is enabled through `ModelConfiguration`, SwiftData automatically detects the CloudKit container set in the Entitlements and attempts synchronization.

```swift
// Meku/MekuApp.swift
import SwiftData

@main
struct MekuApp: App {
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([
            Item.self,
        ])
        // Set isStoredInMemoryOnly: false to use persistent storage and enable CloudKit sync
        let modelConfiguration = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)

        do {
            return try ModelContainer(for: schema, configurations: [modelConfiguration])
        } catch {
            fatalError("Could not create ModelContainer: \(error)")
        }
    }()
    // ...
}
```

### 3) Entitlements Configuration
The `Meku.entitlements` file grants the app permission to use iCloud and CloudKit.
- **iCloud Services**: `CloudKit` usage is specified.
- **iCloud Containers**: The container identifier `iCloud.com.konit611.Meku` designates the space for storing data.
- **Background Modes**: (Checked in settings) `Remote notifications` must be enabled to receive real-time sync notifications.

---

## 5. References

Here are Apple official documents that can be referenced for this blog post.

### [Syncing model data across a person's devices](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices)
> **Key Content**: Explains how SwiftData synchronizes model data across a user's devices and the CloudKit permissions and settings required for this. Includes information about schema compatibility and how `ModelContainer` infers CloudKit containers.

### [Preserving your app's model data across devices](https://developer.apple.com/documentation/swiftdata/preserving-your-apps-model-data-across-devices)
> **Key Content**: Provides a comprehensive guide on data persistence and preserving data across devices.

### [Mirroring a Core Data Store to CloudKit](https://developer.apple.com/documentation/coredata/mirroring_a_core_data_store_to_cloudkit)
> **Key Content**: Covers the mirroring operation principles between Core Data (the foundation of SwiftData) and CloudKit in depth. Useful for understanding how synchronization is handled internally.
