# Numinous Ways — Retreat Community Platform

A cross-platform mobile app for a therapeutic retreat company, connecting retreat participants before, during, and after their retreats. I designed, built, and maintain it as the sole engineer — from first commit to production, currently live on both stores.

**[App Store](https://apps.apple.com/gb/app/numinous-ways/id6742788471)** · **[Google Play](https://play.google.com/store/apps/details?id=com.patrykgronczewski.numinousway)**

> The full source code is proprietary to the client, so this repository documents the architecture and engineering decisions instead.

## Screenshots

<p float="left">
  <img src="./images/screenshot-1.png" width="30%" />
  <img src="./images/screenshot-2.png" width="30%" />
  <img src="./images/screenshot-3.png" width="30%" />
</p>

## Background

The app started as my BEng Honours project at Edinburgh Napier University (First-Class, University Class Award) and went through beta with real retreat participants. After graduating, the client kept me on — I've since spent a year evolving it in production: new features, redesigns, and ongoing maintenance, with 99.9% uptime and zero critical incidents since launch.

## What It Does

- **Real-time group chat** — retreat groups with read receipts, unread tracking, and image messages
- **AI image gallery** — users describe experiences that are hard to put into words; the app generates images via the OpenAI API and shares them to a community gallery
- **21-day structured courses** — preparation and integration programmes with daily modules, task tracking, audio meditations, and scheduled reminders
- **Social timeline** — posts, comments, and likes within the retreat community
- **Retreat management** — schedules, venues, travel details, and facilitator info, all editable by admins without app releases

## Scale

- 36+ screens, 21 backend services, 23 data models
- Single codebase shipping to iOS and Android
- 50+ active users in a private, invitation-based community

## Architecture

**Flutter + Firebase, MVVM with Provider.** UI screens bind to ViewModels/Providers, which call dedicated service classes (one per domain: auth, chat, courses, moderation, storage…). Services talk to Firebase — Firestore for real-time data, Auth for identity, Storage for media, Cloud Functions for server-side work.

**Cloud Functions (TypeScript)** handle what shouldn't live on the client — most notably push notifications: a Firestore trigger on new group messages fans out FCM notifications to group members, with platform-specific payloads for iOS and Android and a data payload driving in-app navigation.

**AI image pipeline:** prompt → OpenAI API → preview → on publish, the app downloads the generated image (OpenAI URLs expire after ~30 minutes), compresses it, uploads to Firebase Storage, and serves the gallery from there.

## Code Highlights

A couple of representative patterns from the codebase (full source is client-proprietary):

**AI image pipeline** — OpenAI image URLs expire after ~30 minutes, so on publish the app downloads the image and re-hosts it in Firebase Storage. Two sizes are uploaded per image: a 512px thumbnail for the gallery grid and a 1024px version for detail view, cutting bandwidth and storage costs. Dependencies are injected through the constructor, which keeps the service unit-testable:

```dart
class AiGalleryService {
  static const int thumbnailMaxSize = 512; // gallery grid
  static const int detailMaxSize = 1024;   // detail view
  static const int imageQuality = 85;

  AiGalleryService({
    FirebaseFirestore? firestore,
    FirebaseStorage? storage,
    http.Client? httpClient,
  })  : _firestore = firestore ?? FirebaseFirestore.instance,
        _storage = storage ?? FirebaseStorage.instance,
        _httpClient = httpClient ?? http.Client();

  /// Download the generated image (OpenAI URLs expire),
  /// compress, and upload thumbnail + detail versions.
  Future<Map<String, String>> uploadAiImage(String imageUrl, String userId) async {
    final response = await _httpClient.get(Uri.parse(imageUrl));
    if (response.statusCode != 200) {
      throw Exception("Failed to download AI image from OpenAI.");
    }

    final bytes = response.bodyBytes;
    final fileId = _uuid.v4();

    final thumbnailUrl = await _uploadCompressedImage(
        bytes, userId, fileId, thumbnailMaxSize, 'thumbnail');
    final detailUrl = await _uploadCompressedImage(
        bytes, userId, fileId, detailMaxSize, 'detail');

    return {'thumbnailUrl': thumbnailUrl, 'detailUrl': detailUrl};
  }
}
```

**Server-side push notifications (Cloud Functions, TypeScript)** — a Firestore trigger fans out FCM notifications when a group message is created, so notification logic stays off the client:

```typescript
export const onGroupMessageCreated = functions
  .runWith({ memory: "512MB", timeoutSeconds: 120 })
  .firestore
  .document("retreat_groups/{groupId}/messages/{messageId}")
  .onCreate(async (snap, context) => {
    const messageData = snap.data();

    // System messages ("X joined the group") don't notify
    if (messageData.type === "system") return null;

    // Fan out FCM to all group members except the sender,
    // with platform-specific payloads for iOS and Android
    // and a data payload driving in-app navigation.
    // ...
  });
```

## Security & Data Protection

The app handles sensitive personal data (retreat enrolment includes passport details), which shaped several decisions:

- **Firestore Security Rules** restrict reads/writes to a user's own data, with role-based access for facilitators and admins
- **Automatic data deletion** — sensitive enrolment data is deleted 90 days after a retreat ends; the schedule locks once retreat dates are set and can't be overridden, even by admins
- **Full account wipe** on user-initiated deletion, at any time
- **Content moderation** — reporting, admin review, user blocking, and pre-generation prompt filtering for the AI gallery
- **Consent gating** — Terms/Privacy acceptance required before access

## Delivery

- **CI/CD with GitHub Actions** — analyze, test, and build on every push, with secrets (API keys, Firebase config) injected from GitHub Secrets rather than committed
- **Testing** — unit tests for services and providers, integration tests for auth flows
- **Releases** — staged rollouts to Google Play and the App Store

## Companion Admin Panel

A separate **React admin panel** (react-admin + Material UI, backed by the same Firebase project) lets facilitators and admins manage users, groups, webinars, venues, content reports, and data exports with role-based access — so day-to-day content changes never require an app release.

## Tech Stack

Flutter/Dart · Firebase (Auth, Firestore, Storage, Cloud Functions, FCM) · TypeScript (Cloud Functions) · OpenAI API · Provider (MVVM) · React + react-admin (admin panel) · GitHub Actions
