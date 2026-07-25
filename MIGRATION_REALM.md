MIGRATION CHECKLIST — Realm 3.x → 20.x

Purpose
-------
This checklist helps upgrade from Realm v3.x (older API surface) to Realm v20.x (current major release used by CocoaPods in this project). It focuses on practical, minimal steps to get a working build, required code changes, runtime migration handling, and testing/rollback guidance.

Important
---------
- This is a major jump. Expect breaking API changes. Review the upstream Realm release notes/changelogs for each major intermediate version if anything here is unclear.
- Backup any production Realm files before running migrations.

Prerequisites
-------------
- CocoaPods: ensure Podfile/podspec allows Realm/RealmSwift 20.x and run `pod update Realm RealmSwift --repo-update` in the Example/ directory.
- Xcode: use a recent Xcode matching the Realm binary build requirements (the project already used Xcode 26.x in this workspace)
- Swift: match or exceed the Swift version expected by Realm (the podspec lists swift_version = '5.8').

Quick steps (high level)
------------------------
1. Relax version pins in FolioReaderKit.podspec for Realm/RealmSwift (done in this workspace) or pin to the target 20.x if you prefer deterministic builds.
2. Run: cd Example && pod update Realm RealmSwift --repo-update
3. Open workspace: open Example/Example.xcworkspace
4. Clean build folder (Cmd+Shift+K), then build (Cmd+B). Fix compile errors iteratively.
5. Implement runtime migration (schemaVersion + migrationBlock) before any Realm() access.
6. Run unit/integration tests and manual QA on sample Realm files.

Common compile-time/API changes and fixes
----------------------------------------
Below are typical source changes encountered when moving from very old Realm versions to modern RealmSwift:

1) Realm initialization now throws
Previously: let realm = Realm()
Now: do {
  let realm = try Realm()
  // use realm
} catch {
  // handle initialization error
}

Search for direct instantiations and add try + do/catch or propagate errors.

2) Configuration API and migrations
Set a default configuration with schemaVersion and migrationBlock very early in app startup (before any Realm usage):

import RealmSwift

let config = Realm.Configuration(
    schemaVersion: 1, // increment when your schema changes
    migrationBlock: { migration, oldSchemaVersion in
        if oldSchemaVersion < 1 {
            // Example: rename property "oldName" -> "newName"
            migration.renameProperty(onType: "MyObject", from: "oldName", to: "newName")

            // Example: enumerate objects to set default values
            migration.enumerateObjects(ofType: "MyObject") { oldObject, newObject in
                if newObject?["newField"] == nil {
                    newObject?["newField"] = "default"
                }
            }
        }
        // Add additional conditional branches when bumping schemaVersion in future
    }
)
Realm.Configuration.defaultConfiguration = config

Notes:
- schemaVersion is a 64-bit integer. Start at 1 and increment on each schema change that requires migration logic.
- The migration block uses Migration API: enumerateObjects(ofType:), renameProperty, deleteData(forType:), etc.

3) Threading model, ThreadSafeReference, and frozen objects
Realm objects and Results are thread-confined. If code previously passed live objects between threads, update to use ThreadSafeReference or frozen() objects:

// Pass object safely between threads
let objRef = ThreadSafeReference(to: someObject)
DispatchQueue.global().async {
    autoreleasepool {
        let realm = try! Realm()
        guard let obj = realm.resolve(objRef) else { return }
        // use obj
    }
}

Or use frozen() to create an immutable snapshot usable across threads:
let frozen = someObject.freeze()

4) Changes to optional/primitive types and List API
- Lists and Results API largely similar but check any usage of older bridging APIs. Update any use of RLM... Objective-C APIs to RealmSwift equivalents.
- Where RealmOptional<T> was used, modern versions prefer standard optionals for Swift-native types. Inspect compile-time errors for guidance.

5) Notification tokens and invalidation
Ensure you keep strong references to NotificationToken objects until you explicitly invalidate them (token?.invalidate()).

6) Migration of encryption / file locations
If you used custom fileURL or encryption keys, ensure the configuration includes those fields when assigning defaultConfiguration.

Runtime migration: safe procedure
--------------------------------
1. Add the configuration + migrationBlock early (application:didFinishLaunchingWithOptions or equivalent) before any Realm usage.
2. Use a schemaVersion value higher than the previous version recorded in your app's Realm files.
3. Implement minimal migration steps:
   - rename properties (migration.renameProperty)
   - set default values (enumerateObjects)
   - map types where necessary (manually convert or delete + recreate)
4. Run the app with simulator/device containing a copy of the old Realm file and verify data integrity.

Testing checklist
-----------------
- Unit tests: run the test suite (Example/ tests) and fix compile failures.
- Integration/manual QA:
  - Backup existing .realm files from ~/Library/Developer/CoreSimulator/Devices/... or device.
  - Launch the app; confirm migration runs without crashing and data is present.
  - Test critical flows that read/write Realm (bookmarks, annotations, user settings, etc.)
- Automated: add migration unit tests that open a pre-created Realm file with old schemaVersion and assert migration performed expected changes.

Rollback and recovery
---------------------
- Always keep a copy of the original Realm files before migration.
- If migration fails, you can restore the previous Realm file to the device/simulator and re-run the previous app build.
- Consider implementing migration logging inside the migrationBlock to help diagnose issues.

Pinning versions after verification
----------------------------------
Once project builds and the runtime behavior is verified, pin the Realm and RealmSwift versions in the FolioReaderKit.podspec to the working versions (e.g., '20.0.4') to prevent future unexpected upgrades. Steps:

1. Update FolioReaderKit.podspec to set s.dependency 'RealmSwift', '20.0.4' (and 'Realm', '20.0.4' if needed).
2. Run: cd Example && pod install
3. Commit the podspec and Podfile.lock changes.

Useful commands
----------------
- Update pods: cd Example && pod update Realm RealmSwift --repo-update
- Install pods: cd Example && pod install
- Clean Xcode build: Cmd+Shift+K (or xcodebuild clean)
- Command-line build: xcodebuild -workspace Example/Example.xcworkspace -scheme Example -destination 'platform=iOS Simulator,name=iPhone 14' build

References
----------
- Realm Swift docs: https://realm.io/docs/swift/latest/
- Realm Migration Guide: check the Realm changelog and migration docs for detailed breaking changes

If you'd like, next actions I can perform:
- Add a minimal migration unit test scaffold to Example/ to validate the migration block
- Re-pin Realm/RealmSwift versions in the podspec and commit the changes
- Run a command-line build to surface compile-time errors now and produce a short fix list

---
Generated by assistant (Copilot CLI runtime in VS Code) for this repository.