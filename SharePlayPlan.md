# BookPlayer SharePlay Implementation Plan

## Overview

Add SharePlay support to a fork of [TortugaPower/BookPlayer](https://github.com/TortugaPower/BookPlayer) (GPL-3.0, Swift/iOS) so that two users listening to the same audiobook can listen together with playback kept in sync, position, play/pause, seek, chapter, and speed, started with a single tap.

**Core architecture decision:** use Apple's purpose-built **`AVPlaybackCoordinator`** for playback synchronization rather than hand-rolling sync over `GroupSessionMessenger`. `AVPlaybackCoordinator` is the framework Apple designed for exactly this (synchronized media over SharePlay). It coordinates rate, play/pause, and seek across all participants and handles network latency, clock differences, and waiting for slow-to-buffer participants automatically. This deletes the most error-prone parts of a custom implementation: manual command messages, echo suppression, cross-device clock math, and latency compensation.

**Design constraints:**
- **Clean and minimal.** Prefer additive files. Touch existing playback code in exactly one place, the surgical hook that hands the group session to the `AVPlayer`'s coordinator.
- **Merge-friendly against upstream `develop`.** The new code is a self-contained `SharePlay/` directory plus a handful of small insertions.

**Deployment target is iOS 18.0**, so every API used here (GroupActivities iOS 15+, `AVPlaybackCoordinator` iOS 15+, unreliable messenger iOS 16+) is available unconditionally. **No `@available` guards are needed anywhere.**

---

## Why `AVPlaybackCoordinator` (the purpose-built path)

The first draft of this plan synced playback by observing `PlayerManager`'s Combine publishers, serializing each change into a `PlaybackCommand`, sending it over `GroupSessionMessenger`, and replaying it on the other device. That approach has three hard problems that `AVPlaybackCoordinator` solves for free:

| Problem with manual messenger sync | How `AVPlaybackCoordinator` handles it |
|---|---|
| **Echo loops.** A remote command applied locally re-fires the local publishers, which re-broadcasts the command. A boolean "suppress echo" flag is racy because `AVPlayer` state changes are asynchronous. | The coordinator owns the player's timeline. There is no local-publisher round trip to suppress. |
| **Clock skew.** `Date().timeIntervalSince1970` on one device is not comparable to another's, so "seek to position X as of time T" cannot be computed correctly. | The coordinator establishes a shared timebase across participants and compensates for latency internally. |
| **Buffering / slow joiners.** A participant whose audio is still loading would drift out of sync. | The coordinator suspends and resumes the group automatically so everyone stays aligned. |

What we still own (and what stays small): identifying that both devices are playing the **same book**, loading the local copy on the joiner, and the one-tap UI. That is the entire job of the `SharePlay/` layer below.

---

## Architecture Context

BookPlayer's playback stack is cleanly layered and protocol-oriented:

```
PlayerManagerProtocol          ← play/pause/seek/speed/jumpTo/load interface (SHARED with watch target)
  └─ PlayerManager             ← concrete iOS impl, wraps a private AVPlayer (`audioPlayer`)
       ├─ PlaybackService      ← data coordination (time updates, next/prev item)
       ├─ SpeedService         ← per-book speed storage
       └─ LibraryService       ← CoreData-backed library queries

PlayableItem                   ← NSObject/Identifiable; book identity + playback state
PlayableChapter (Codable)      ← chapter metadata (title, start, duration, index)

PlayerViewModel                ← SwiftUI bridge, subscribes to PlayerManager publishers
AppServices / CoreServices     ← DI container, wires everything up (iOS struct; watch has its own)
```

Key facts that shape the design:

- **`PlayerManager` owns a `private var audioPlayer = AVPlayer()`.** This is the instance whose `playbackCoordinator` we hand to the group session. Because it is `private`, the coordination hook must live **inside `PlayerManager.swift`** (a same-file extension), not in the `SharePlay/` directory.
- **`audioPlayer` is recreated** (reassigned on load; the source even comments that "the `AVPlayer` instance can be recreated"). Coordination binds to a specific `AVPlayer`, so we must store the active group session and **re-hand it to the new player after recreation**. This is the one piece of real care the integration needs.
- **`PlayerManagerProtocol` is shared with the watch target** (`#if os(watchOS) import BookPlayerWatchKit`). Adding a SharePlay method to it would force the watch `PlayerManager` to implement it. So the coordination hook goes on a **new iOS-only protocol** that only the iOS `PlayerManager` conforms to.
- **Speed is applied via `audioPlayer.rate`.** Under coordination the rate is synchronized, so a speed change on one device propagates to all participants (the intended "listen together" behavior, see the speed note in the design section).
- `PlayableItem` exposes `title`, `author`, `duration`, and `chapters` as public properties, everything the content fingerprint needs. Note it is an `NSObject`, **not** `Codable`; we never send a `PlayableItem` over the wire, we read these properties to build our own small `Codable` fingerprint.
- `GroupStateObserver` tells us whether the user is already in a FaceTime/Messages call (activate directly) or not (present the system sharing sheet).

---

## The Book-Matching Problem

This is the core design challenge that `AVPlaybackCoordinator` does **not** solve for us. BookPlayer identifies books by `relativePath` and a per-device `uuid`. Two users who imported the same audiobook have different values for both.

**Solution: content-based fingerprint.**

```swift
struct BookFingerprint: Codable, Hashable {
    let titleNormalized: String    // lowercased, whitespace-trimmed
    let authorNormalized: String   // lowercased, whitespace-trimmed
    let durationRounded: Int       // total duration in whole seconds
    let chapterCount: Int          // number of chapters

    /// Stable string used as the AVPlaybackCoordinator content identifier.
    var identifier: String {
        "\(titleNormalized)|\(authorNormalized)|\(durationRounded)|\(chapterCount)"
    }
}
```

The fingerprint plays **two** roles in this design:

1. It is the payload carried by the `GroupActivity` itself (see below), so the joiner knows which book to load without any separate messenger.
2. Its `identifier` is what each device returns from the `AVPlayerPlaybackCoordinatorDelegate`, so the coordinator confirms both sides are playing the same content before it synchronizes them.

**Matching realism.** Independently imported copies (different transcodes, M4B vs a folder of MP3s) frequently differ by a few seconds of total duration or in chapter count, so exact equality will miss more often than the happy-path framing suggests. Two mitigations:
- Match `durationRounded` with a small tolerance (for example within ±2 seconds) rather than exact equality.
- **Manual picker fallback:** if no confident match is found, present a book picker so the joiner can map the session to a local book. This keeps one-tap UX for the happy path and degrades gracefully otherwise.

Match against lightweight library metadata (title/author/duration/chapter count queried from CoreData), not by constructing a full `PlayableItem` for every library entry.

---

## Design: division of labor

```
AVPlaybackCoordinator   →  ALL timing sync: play, pause, seek, rate/speed, latency, slow-joiner waiting.
GroupActivity payload   →  WHICH book: carries the BookFingerprint + title to the joiner on session start.
SharePlay/ layer        →  Session lifecycle, local book match + load, one-tap UI, "book not found" UX.
```

Because the `GroupActivity`'s stored properties travel with the session, the joiner reads `session.activity.bookFingerprint` directly. **No `GroupSessionMessenger` is needed for the happy path.** (A thin messenger is only required if we later support changing books mid-session; that is deferred as a follow-up to keep v1 minimal.)

---

## New Files to Create

All new files live in `BookPlayer/SharePlay/`. None of them touch existing playback logic.

### 1. `BookPlayer/SharePlay/AudioBookActivity.swift`

The `GroupActivity` conformance, the system entry point. Its stored properties are the book handshake.

```swift
import GroupActivities
import Foundation

struct AudioBookActivity: GroupActivity {
    let bookFingerprint: BookFingerprint
    let bookTitle: String

    static let activityIdentifier = "app.bookplayer.shareplay.listen-together"

    var metadata: GroupActivityMetadata {
        var meta = GroupActivityMetadata()
        meta.title = NSLocalizedString("Listen to \"\(bookTitle)\" together", comment: "")
        meta.type = .listenTogether  // audio experience → proper Now Playing + audio session handling
        return meta
    }
}
```

### 2. `BookPlayer/SharePlay/BookFingerprint.swift`

Just the fingerprint type (shown above) plus a convenience initializer from a `PlayableItem`:

```swift
extension BookFingerprint {
    init(from item: PlayableItem) {
        self.titleNormalized = item.title.lowercased().trimmingCharacters(in: .whitespacesAndNewlines)
        self.authorNormalized = item.author.lowercased().trimmingCharacters(in: .whitespacesAndNewlines)
        self.durationRounded = Int(item.duration)
        self.chapterCount = item.chapters.count
    }
}
```

> The `PlaybackCommand` and `PlaybackSyncState` message types from the first draft are **gone**, the coordinator replaces them entirely.

### 3. `BookPlayer/SharePlay/PlaybackCoordinating.swift`

A tiny iOS-only protocol so the SharePlay layer can drive coordination without depending on the concrete `PlayerManager` and without touching the shared `PlayerManagerProtocol`.

```swift
import GroupActivities

protocol PlaybackCoordinating: AnyObject {
    func coordinatePlayback(with session: GroupSession<AudioBookActivity>)
    func stopCoordinatingPlayback()
}
```

`PlayerManager` conforms to this in `PlayerManager.swift` (see "Existing Files to Modify"), because it needs access to the private `audioPlayer`.

### 4. `BookPlayer/SharePlay/SharePlayService.swift`

The session layer. Much smaller than a messenger-based design because it does **no** playback forwarding.

Responsibilities:
- Observe `AudioBookActivity.sessions()` for incoming sessions.
- On a new session: read `session.activity.bookFingerprint`, match it to a local book, load it, `join()`, then call `playerManager.coordinatePlayback(with: session)` to hand all timing sync to `AVPlaybackCoordinator`.
- If no local match: surface "book not found" (with title/author) and offer the manual picker.
- Manage lifecycle: `join`, `leave`, `end`, and clearing coordination on teardown.

Core flow:

```swift
for await session in AudioBookActivity.sessions() {
    let fingerprint = session.activity.bookFingerprint

    if let localItem = await libraryMatcher.find(matching: fingerprint) {
        await playerLoader.load(localItem)
        session.join()
        playerManager.coordinatePlayback(with: session)   // ← AVPlaybackCoordinator takes over timing
    } else {
        coordinator.presentBookNotFound(fingerprint, title: session.activity.bookTitle)
    }
}
```

### 5. `BookPlayer/SharePlay/SharePlayCoordinator.swift`

Higher-level coordinator between `SharePlayService` and the UI.

Responsibilities:
- `@Published` state for UI binding (`isInSession`, `participantCount`, `sessionState`).
- The one-tap entry point `startSession(for item: PlayableItem)`: build `AudioBookActivity` with the item's fingerprint, then use `GroupStateObserver` to decide, already in a call → `activity.activate()`; not in a call → signal the UI to present `GroupActivitySharingController`.
- `leaveSession()` / `endSession()`, which also call `playerManager.stopCoordinatingPlayback()`.
- The book-not-found and manual-picker presentation.

### 6. `BookPlayer/SharePlay/SharePlayButton.swift`

The SwiftUI one-tap control.

```swift
import SwiftUI
import GroupActivities

struct SharePlayButton: View {
    @ObservedObject var coordinator: SharePlayCoordinator
    @State private var showSharingSheet = false
    let currentItem: PlayableItem

    var body: some View {
        Button {
            Task {
                // prepareForActivation() drives the activate-vs-sheet decision per the framework's own result.
                if await coordinator.startSession(for: currentItem) == .needsSharingSheet {
                    showSharingSheet = true
                }
            }
        } label: {
            Image(systemName: "shareplay")
                .foregroundColor(coordinator.isInSession ? .green : .primary)
        }
        .sheet(isPresented: $showSharingSheet) {
            ActivitySharingView(activity: coordinator.makeActivity(for: currentItem))
        }
    }
}
```

The SF Symbol `shareplay` is the standard system icon; it turns green during an active session, consistent with system conventions.

### 7. `BookPlayer/SharePlay/ActivitySharingView.swift`

`UIViewControllerRepresentable` wrapper for `GroupActivitySharingController` (used when not already in a call, lets the user pick contacts via Messages or FaceTime). Note the initializer **throws**, so handle it rather than force-trying.

```swift
import GroupActivities
import SwiftUI
import UIKit

struct ActivitySharingView: UIViewControllerRepresentable {
    let activity: AudioBookActivity

    func makeUIViewController(context: Context) -> UIViewController {
        do {
            return try GroupActivitySharingController(activity)
        } catch {
            // Fail soft: present an empty controller and log, never crash the player.
            return UIViewController()
        }
    }

    func updateUIViewController(_ controller: UIViewController, context: Context) {}
}
```

---

## Existing Files to Modify

### 1. `BookPlayer/Player/PlayerManager.swift`  ← the one deliberate playback-code change

This is the single place we touch existing playback logic, and it is the deliberate cost of using the purpose-built API. Three small, additive pieces, all `#if !os(watchOS)` so the watch target is unaffected:

```swift
#if !os(watchOS)
import GroupActivities

extension PlayerManager: PlaybackCoordinating {
    /// Hand the current AVPlayer's coordinator to the SharePlay group session.
    /// Re-called automatically whenever `audioPlayer` is recreated (see `load`).
    func coordinatePlayback(with session: GroupSession<AudioBookActivity>) {
        activeGroupSession = session
        audioPlayer.playbackCoordinator.delegate = self
        audioPlayer.playbackCoordinator.coordinateWithSession(session)
    }

    func stopCoordinatingPlayback() {
        activeGroupSession = nil
        // Coordination tears down when the GroupSession ends/leaves; no explicit "uncoordinate" call exists.
    }
}

extension PlayerManager: AVPlayerPlaybackCoordinatorDelegate {
    func playbackCoordinator(_ c: AVPlayerPlaybackCoordinator, identifierFor item: AVPlayerItem) -> String {
        // Same content on both devices → same identifier → coordinator synchronizes them.
        currentItem.map(BookFingerprint.init(from:))?.identifier ?? UUID().uuidString
    }
}
#endif
```

Plus **one stored property** in the class body and **one re-apply hook** where `audioPlayer` is recreated (~line 155):

```swift
#if !os(watchOS)
private var activeGroupSession: GroupSession<AudioBookActivity>?
#endif

// immediately after `audioPlayer = AVPlayer()`:
#if !os(watchOS)
if let activeGroupSession { coordinatePlayback(with: activeGroupSession) }
#endif
```

That re-apply hook is the only modification to existing executable lines in the entire project; everything else is additive.

### 2. `BookPlayer/Info.plist`

Add the key that enables SharePlay. Without it, `GroupActivity.sessions()` never emits.

```xml
<key>NSSupportsGroupActivities</key>
<true/>
```

### 3. `BookPlayer/BookPlayer.entitlements`  *(verify whether this is actually needed)*

The Xcode "Group Activities" capability is what the system expects, and in current toolchains it primarily sets the `NSSupportsGroupActivities` Info.plist key above. **Verify in Xcode whether a separate `com.apple.developer.group-session` entitlement is required before adding one**; it may be unnecessary. If Xcode's capability UI adds an entitlement, let it, do not hand-author a key that the toolchain does not require.

### 4. `BookPlayer/Utils/AppServices.swift`

In `createCoreServicesIfNeeded(from:)` (~line 88, after `playerManager` is created), create and inject the coordinator, passing the concrete `playerManager` as the `PlaybackCoordinating` dependency:

```swift
let sharePlayCoordinator = SharePlayCoordinator(
    playerManager: playerManager,           // also conforms to PlaybackCoordinating
    libraryService: libraryService,
    playerLoaderService: playerLoaderService
)
```

### 5. `BookPlayer/Utils/CoreServices.swift`  (the iOS `struct`, not the watch class)

Add one property:

```swift
let sharePlayCoordinator: SharePlayCoordinator
```

### 6. `BookPlayer/Player/Views/PlayerView.swift`

Insert `SharePlayButton` into the existing player controls area (for example next to the AirPlay picker). One additive view element, not a redesign.

---

## Merge Safety Analysis

| File | Change Type | Conflict Risk |
|------|-------------|---------------|
| `BookPlayer/SharePlay/*` (7 new files) | New directory, new files | **Zero**, upstream will never have these |
| `PlayerManager.swift` | `#if !os(watchOS)` extension + 1 stored property + 1 re-apply line | **Low-Medium**, the only existing-logic touch; conflicts only if upstream reworks player recreation. The extension itself never conflicts |
| `Info.plist` | One new key | **Minimal** |
| `BookPlayer.entitlements` | Possibly none (verify) | **Minimal** |
| `AppServices.swift` | ~4 lines inserted after `playerManager` creation | **Low** |
| `CoreServices.swift` (iOS) | One new property | **Low** |
| `PlayerView.swift` | One new view element | **Low-Medium**, manual rebase if the player UI is redesigned |

**Net complexity is lower than the messenger design** despite touching `PlayerManager`: choosing `AVPlaybackCoordinator` removes roughly 300 lines of fragile sync code (command serialization, echo suppression, heartbeat, latency math) in exchange for a ~30-line same-file extension and a single re-apply line.

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       DEVICE A (Initiator)                       │
│  PlayerView → SharePlayButton ──tap──▶ SharePlayCoordinator      │
│                                          │                       │
│                                          ├─ AudioBookActivity     │
│                                          │   (carries fingerprint)│
│                                          ├─ activate()  OR        │
│                                          │   GroupActivitySharing  │
│                                          ▼                        │
│  PlayerManager.audioPlayer ──.playbackCoordinator──┐             │
│       (play/pause/seek/rate owned by coordinator)  │             │
└────────────────────────────────────────────────────┼────────────┘
                                                      │
                         AVPlaybackCoordinator        │  shared timebase,
                         over E2E SharePlay channel    │  latency-compensated,
                         (FaceTime / Messages)         │  buffering-aware
                                                      │
┌────────────────────────────────────────────────────┼────────────┐
│                       DEVICE B (Joiner)            ▼             │
│  AudioBookActivity.sessions() → SharePlayService                │
│       ├─ read session.activity.bookFingerprint                  │
│       ├─ match to local library → load PlayableItem             │
│       ├─ session.join()                                         │
│       └─ playerManager.coordinatePlayback(with: session)        │
│             │                                                   │
│  PlayerManager.audioPlayer.playbackCoordinator ◀────────────────┘
│       (timing now driven by the coordinator)                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Detailed Implementation Sequence

### Phase 1: Foundation
1. Local fork; create `feature/shareplay` off `develop`.
2. Add the Group Activities capability in Xcode (sets `NSSupportsGroupActivities`; verify whether any entitlement is added).
3. Create `BookPlayer/SharePlay/`.
4. Implement `BookFingerprint` (+ `init(from:)` and `identifier`) and `AudioBookActivity`. Pure data, fully unit-testable.
5. Implement `PlaybackCoordinating`.

### Phase 2: Player coordination hook
6. In `PlayerManager.swift`, add the `#if !os(watchOS)` `PlaybackCoordinating` + `AVPlayerPlaybackCoordinatorDelegate` extension, the `activeGroupSession` property, and the re-apply line after player recreation.
7. Verify the delegate returns a stable identifier for the current item, and that speed changes propagate under coordination (see speed note below; you may need to route speed through `defaultRate` so the coordinator broadcasts it).

### Phase 3: Session + matching
8. Implement `SharePlayService`: observe `sessions()`, read the activity fingerprint, match locally, load, join, hand off to the coordinator.
9. Implement the lightweight library matcher (metadata-based, with duration tolerance).
10. Implement `SharePlayCoordinator`: published state, `startSession(for:)` with `GroupStateObserver` routing, leave/end, book-not-found + manual picker.

### Phase 4: UI
11. Implement `ActivitySharingView` (throwing init handled).
12. Implement `SharePlayButton`.
13. Wire into `AppServices` / `CoreServices`; add the button to `PlayerView`.

### Phase 5: Edge cases & polish
14. Book-not-found alert with title/author so the joiner knows what to import.
15. Participant disconnect: session continues locally, indicator turns off, `stopCoordinatingPlayback()` clears state.
16. Confirm `AVAudioSession` (`.playback` / `.spokenAudio`) coexists with SharePlay's audio session.
17. Decide and document the speed-sync behavior (shared session speed vs per-user).

---

## Speed note

BookPlayer applies per-book speed by setting `audioPlayer.rate` directly. Under `AVPlaybackCoordinator`, rate is part of what gets synchronized, so a speed change on one device will propagate to all participants. For "listen together" that shared speed is the natural behavior, but it is a product decision worth calling out (some users may expect to keep their own speed). On iOS 16+ the coordinator broadcasts playback rate via `AVPlayer.defaultRate`; verify BookPlayer's speed path drives coordinated rate correctly and adjust if a speed change does not propagate.

---

## Testing Strategy

SharePlay cannot be tested in the Simulator. You need:
- **Two physical iOS devices** with different Apple IDs signed into FaceTime.
- A **paid Apple Developer account** (Group Activities capability).
- Both devices on the same build.

| Scenario | What to verify |
|----------|---------------|
| Start during a FaceTime call | Activity activates immediately; both devices coordinate |
| Start without a call (Messages) | `GroupActivitySharingController` appears; session starts over iMessage |
| Same book, same format | Fingerprint matches; book auto-loads on joiner |
| Same book, different format (M4B vs MP3 folder) | Fingerprint (with duration tolerance) still matches |
| Book not found on joiner | Alert with title/author; manual picker available |
| Play / pause / seek | Coordinator keeps both devices aligned without echo loops |
| Speed change | Rate propagates to all participants (per the speed note) |
| Chapter skip | Underlying seek is coordinated; both follow |
| Late joiner | Coordinator aligns the joiner to the group's current time after load |
| Slow buffering on one device | Coordinator suspends/resumes so no drift |
| **Player recreation mid-session** | After `audioPlayer` is recreated, coordination is re-applied and sync survives |
| One participant leaves | Other continues locally; indicator off; `stopCoordinatingPlayback()` clears state |
| Upstream merge | Rebase `feature/shareplay` on latest `develop`; confirm no conflicts in playback logic |

---

## API Reference Summary

| Type / API | Role |
|------------|------|
| `GroupActivity` protocol | Define the shareable activity (`AudioBookActivity`); its stored props travel to the joiner |
| `GroupActivityMetadata` | Title and `type = .listenTogether` |
| `GroupSession<AudioBookActivity>` | The active session; observe `$state`, `$activeParticipants` |
| `AudioBookActivity.sessions()` | Async sequence of incoming sessions on the joiner |
| `GroupStateObserver` | Whether the user is already in a FaceTime/Messages call |
| `GroupActivitySharingController` | System sharing UI when not already in a call (throwing init) |
| `activity.prepareForActivation()` / `activate()` | Ask the system to start the session |
| **`AVPlayer.playbackCoordinator`** | The per-player coordinator (the heart of sync) |
| **`coordinateWithSession(_:)`** | Bind the player's coordinator to the group session |
| **`AVPlayerPlaybackCoordinatorDelegate.playbackCoordinator(_:identifierFor:)`** | Return the fingerprint identifier so both sides agree on "same content" |

`GroupSessionMessenger` is intentionally **not** used in v1; `AVPlaybackCoordinator` carries all playback state. A thin messenger would only be added later to support changing books mid-session.

---

## File Inventory

```
NEW FILES (fork only — never conflicts with upstream):
  BookPlayer/SharePlay/
    ├── AudioBookActivity.swift        (~30 lines)  GroupActivity + fingerprint payload
    ├── BookFingerprint.swift          (~40 lines)  Codable fingerprint + identifier + init(from:)
    ├── PlaybackCoordinating.swift     (~10 lines)  iOS-only coordination protocol
    ├── SharePlayService.swift         (~120 lines) Session observe + match + load + hand-off
    ├── SharePlayCoordinator.swift     (~150 lines) UI state, activation routing, not-found/picker
    ├── SharePlayButton.swift          (~40 lines)  One-tap SwiftUI button
    └── ActivitySharingView.swift      (~25 lines)  GroupActivitySharingController wrapper

MODIFIED FILES (minimal, additive — one real logic touch):
  PlayerManager.swift                  +~35 lines (#if !os(watchOS) coordination extension)
                                       +1 stored property, +1 re-apply line  ← only existing-logic change
  Info.plist                           +2 lines (NSSupportsGroupActivities)
  BookPlayer.entitlements              0–2 lines (verify; possibly handled by Xcode capability)
  AppServices.swift                    +4 lines (create + inject coordinator)
  CoreServices.swift (iOS)             +1 line  (one new property)
  PlayerView.swift                     +3 lines (insert SharePlayButton)

TOTAL: ~415 lines of new code; ~12 lines of insertions; exactly 1 modified existing line.
```

---

## Contribution Approach

The plan is to **build it on a local fork, use it for real, then open the conversation with the maintainers**, working build in hand.

This respects the project's process. `CONTRIBUTING.md` asks contributors to "first discuss the change... via issue, **email**, or any other method with the owners... before making a change," and warns that large unsolicited PRs may be closed to focus discussion. So:

1. **Build and use it.** Run the feature on your own devices against a real audiobook. This both proves the design and gives you concrete answers (does the fingerprint match across your imports, does coordination hold during buffering, how does speed-sync feel).
2. **Email the maintainers** (their preferred channel per CONTRIBUTING) describing the feature, the `AVPlaybackCoordinator`-based approach, and the deliberately minimal footprint (one playback-file touch). Mention that you already have a working build you use day to day. Surface the open product questions: whether they want SharePlay at all, the shared-vs-per-user speed decision, and whether they'd prefer this folded into their existing sync infrastructure.
3. **Offer the PR once they greenlight the approach.** The working build is your evidence and de-risks the review; the email is the discussion CONTRIBUTING asks for, not a cold 400-line drop.

Keep the branch rebased on `develop` throughout so the eventual PR is clean.
