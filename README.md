# NotifyPriority

An Android app that filters the "idle but important" notifications — payment
reminders, renewals, ticket/booking updates, important emails — out of the
noise of social media and messaging alerts. It deliberately **skips** OTPs,
password resets, and payment confirmations, since you're already watching
your phone in real time for those.

## How to build and run

This is a real, complete Android Studio project — it wasn't compiled in the
environment that generated it (no Android SDK available there), so you'll
need to open and build it yourself:

1. Install **Android Studio** (Koala or newer) if you don't have it.
2. `File > Open` and select this `NotifyPriority` folder.
3. Let Gradle sync — it will download dependencies automatically (Room,
   Compose, Navigation, WorkManager, DataStore).
4. Run on a device or emulator with **API 26+** (Android 8.0+, required for
   `NotificationListenerService` behavior used here).
5. On first launch, the app will explain why it needs Notification Access
   and send you to system settings to grant it — this permission cannot be
   auto-granted, Android requires the user to enable it manually.

## Project structure

```
app/src/main/java/com/notifypriority/app/
├── MainActivity.kt              # Navigation shell: onboarding → feed / VIP screens
├── classifier/
│   ├── PackageCategoryMap.kt    # Maps app packages → category (banking, email, etc.)
│   ├── KeywordSets.kt           # Suppress vs. promote keyword lists
│   └── ClassificationEngine.kt  # Scores + buckets each notification
├── data/
│   ├── NotificationEntity.kt    # Room table for captured notifications
│   ├── VipSenderEntity.kt       # Room table for user-marked VIP senders
│   ├── NotificationDao.kt / VipSenderDao.kt
│   ├── AppDatabase.kt
│   └── NotificationRepository.kt
├── service/
│   ├── NotificationCaptureService.kt   # The core OS hook — reads every notification
│   ├── NotificationAccessHelper.kt     # Checks/requests the system permission
│   ├── PrioritySummaryNotifier.kt      # Pinned "N things need attention" notification
│   └── BootReceiver.kt                 # Re-registers the listener after reboot
└── ui/
    ├── screens/OnboardingScreen.kt     # Explains the permission before requesting it
    ├── screens/PriorityFeedScreen.kt   # Main feed of filtered important items
    ├── screens/VipSendersScreen.kt     # Manage always-important senders
    └── theme/Theme.kt
```

## How classification works (v1, all on-device, no network calls)

1. **Package lookup** — which app did this come from, and what category
   (banking, email, social, govt, travel, subscriptions)?
2. **VIP override** — if the sender matches something you've manually
   flagged, it's always HIGH priority, full stop.
3. **Suppress veto** — OTP, password reset, and payment-confirmation
   language kills the notification from ever reaching your priority feed.
4. **Keyword scoring** — pending-action language ("due", "expires", "renew",
   "last date", "reschedule") adds to a score; the app's base category also
   contributes. Total score buckets into HIGH / AMBIGUOUS / LOW.
5. Only HIGH shows up in the Priority feed and pinned summary notification.

See `classifier/KeywordSets.kt` to tune the keyword lists — this is the
easiest lever to pull as you see real notifications come through and decide
what's mis-classified.

## What's intentionally left out of this v1 (next steps)

- **AI layer for the AMBIGUOUS bucket** — right now ambiguous notifications
  are just left alone (shown normally, not touched). The natural next step
  is routing only that bucket through a cheap LLM call (e.g. Claude Haiku)
  for a second opinion, rather than classifying every notification with AI.
- **Feedback-driven tuning** — the "Not important" button already writes
  `userFeedback` to the database; nothing reads it back yet. A good v1.5:
  periodically review feedback and auto-suggest keyword/package weight
  adjustments.
- **Home screen widget** — currently the only surface is the pinned
  notification + in-app feed. A widget showing top 3-5 items would reduce
  friction further.
- **iOS** — not feasible with this architecture. Apple doesn't allow apps to
  read other apps' notification content; an iOS version would need to
  connect directly to email/calendar accounts instead (Gmail API, IMAP)
  rather than passively reading notifications.

## Privacy note for your Play Store listing

Everything in this v1 runs entirely on-device — no notification content is
ever sent to a server. Worth stating explicitly and prominently in your
store listing and onboarding, since apps requesting Notification Access get
extra scrutiny from both Google Play review and users.
