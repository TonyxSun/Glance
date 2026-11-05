# Reading Log Glasses App — Project Spec

Personal iOS app that turns Ray-Ban Meta glasses into a hands-free reading companion. Capture a page photo, speak a thought, and automatically create a Notion reading log entry + quote.

Inspired by [ThinkWithMark](https://www.thinkwithmark.com/), but built on glasses the user already owns.

## Goal

While reading, the user takes a photo of a page/paragraph with the glasses. The app immediately starts listening. The user says something like “on the paragraph about X, I feel like [thoughts].” The app:

1. Runs OCR on the photo.
2. Transcribes the speech.
3. Links them.
4. Writes a structured entry to a Notion database (reading log + quotes collection).

No WhatsApp “Hey Meta, send to…” workaround. One continuous flow after the photo.

## User Flow

1. User opens the app (or starts a reading session once).
2. User looks at the page and takes a photo with the glasses (button or “Hey Meta” if supported).
3. App receives the photo and immediately starts the glasses mic / ASR.
4. User speaks the thought (optional short phrase identifying the paragraph).
5. App stops recording after silence or a short timeout / stop gesture.
6. App OCRs the page image on-device.
7. App transcribes the audio (xAI Grok Voice Transcribe).
8. App creates (or appends to) a Notion page:
   - Book / session title
   - Timestamp
   - OCR text (quote / passage)
   - User’s thought
   - Optional: compressed photo attachment
9. App shows a brief confirmation (haptic, sound, or on-phone toast). Session stays open for the next capture.

## Technical Architecture

```
Ray-Ban Meta glasses
  → photo + mic audio (Meta Wearables DAT)
      ↓
iOS app (Swift / SwiftUI)
  → on-device OCR (Apple Vision)
  → speech-to-text (xAI Grok Voice Transcribe)
  → Notion API (create/update database page)
```

### Stack

| Layer | Choice | Why |
|---|---|---|
| Platform | iOS first (Swift / SwiftUI) | Apple Vision OCR is free and excellent; free Personal Team sideload works for personal use; Meta DAT iOS SDK exists |
| Glasses access | Meta Wearables Device Access Toolkit (DAT) 1.0+ | Official camera photo + mic access. Register via Meta AI app. Developer Mode for personal testing. |
| OCR | Apple Vision (`VNRecognizeTextRequest`) | Free, on-device, private, good on book text. Prefer over cloud. |
| STT | xAI Grok Voice Transcribe (`grok-voice-transcribe-2.0`) | User has API key. Batch $0.10/hr, streaming $0.20/hr. |
| Output | Notion API | Free for solo personal workspace. Create database pages. |
| Optional AI structuring | xAI Grok (multimodal or text) | Clean up OCR, extract quote, structure the thought. Token cost is low for personal volume. |
| Persistence | Local (SwiftData or files) + Notion | Keep a local queue so captures are not lost if Notion is offline. |

### Key Meta DAT pieces

- Register the app with the Meta AI app (once).
- Create a session.
- Capture photo (`capturePhoto` / photo data publisher).
- Access mic audio (Bluetooth HFP or DAT speech/ASR where available; experimental PCM noted in docs).
- Start listening immediately after photo arrives.
- Developer Mode + free Apple ID is sufficient for personal testing (7-day signature expiry).

Official docs: https://wearables.developer.meta.com/  
iOS SDK: https://github.com/facebook/meta-wearables-dat-ios  
Android SDK: https://github.com/facebook/meta-wearables-dat-android  

Useful samples / starting points:
- Official BirdSpotter sample (photo + speech + context)
- [RayBan_Meta_NoteBuddy](https://github.com/Alphonso84/RayBan_Meta_NoteBuddy) — glasses camera + Apple Vision OCR + on-device AI (closest existing app)
- [OpenVision](https://github.com/rayl15/OpenVision)
- [GlassFlow](https://github.com/Intent-Lab/GlassFlow) (speech path)

## Notion Data Model

Create one database (or two linked ones). Suggested properties:

**Reading Log** (one row per capture)

| Property | Type | Notes |
|---|---|---|
| Title | Title | Auto: book name or “Capture YYYY-MM-DD HH:mm” |
| Date | Date | Capture timestamp |
| Book / Source | Text or Relation | Optional manual or voice-identified |
| Passage / Quote | Text | OCR result (cleaned) |
| Thought | Text | Transcribed speech |
| Full OCR | Text | Raw OCR for debugging |
| Photo | Files | Optional compressed JPEG (respect 5 MB Free plan limit) |
| Session | Text or Relation | Group captures from one reading session |
| Tags | Multi-select | Optional |

**Quotes Collection** (optional view or second database)
- Same data, filtered or linked so quotes are easy to browse separately from the full log.

The app should create pages via the Notion API (`POST /v1/pages`) with the database parent. Store the integration token securely (Keychain).

## Implementation Plan

### Phase 1 — Minimal viable capture (personal use)
- [ ] Set up Xcode project with Meta DAT iOS package.
- [ ] Register app in Meta AI (Developer Mode).
- [ ] Photo capture from glasses → save image locally.
- [ ] Immediately start mic / record short audio after photo.
- [ ] On-device OCR (Apple Vision) on the photo.
- [ ] Send audio to xAI STT; get transcript.
- [ ] Hard-coded Notion database ID + token; create one page with OCR + transcript + timestamp.
- [ ] Basic UI: start session, status, last capture preview.
- [ ] Local queue so failed Notion writes retry.

### Phase 2 — Better coupling and UX
- [ ] Silence detection or explicit stop to end recording.
- [ ] Optional short spoken identifier (“paragraph about X”) parsed into the entry.
- [ ] Clean OCR text (simple post-processing or light Grok call).
- [ ] Session concept: multiple captures under one reading session.
- [ ] Confirmation feedback (sound / haptic).
- [ ] Settings screen for Notion token, database ID, xAI key.

### Phase 3 — Polish
- [ ] Photo attachment (compressed under 5 MB).
- [ ] Better quote extraction (highlight the spoken paragraph from OCR if possible).
- [ ] Offline queue + background retry.
- [ ] Optional phone-camera fallback when glasses are not connected.
- [ ] Simple history list of recent captures.

### Out of scope for v1
- Android port
- App Store / TestFlight distribution (requires $99 Apple Developer Program)
- Multi-user or sharing
- Full book management / progress tracking
- Real-time streaming STT (batch is fine and cheaper)

## Costs (personal use)

- Meta DAT: free
- Apple Vision OCR: free
- xAI STT: ~$0.10–$0.20 per hour of speech
- Notion API: free (solo Free plan; 5 MB file upload limit)
- Apple Developer Program: not required for personal Xcode sideload (7-day re-sign)

Expected monthly API cost for typical reading: well under $1–$5.

## Monetization and Distribution

Personal use does not require any Meta store listing or App Store publishing. Developer Mode in the Meta AI app is sufficient to register and run the unpublished app on the builder’s own glasses.

There is not yet an open self-serve public “Meta AI glasses app store.” Current discovery surfaces are curated:

- A showcase list inside the Meta AI app.
- Explore app on Meta Ray-Ban Display (mostly web apps).

Meta’s Wearables FAQ notes that submissions will open “soon,” starting with a curated showcase. Creating a Wearables Developer Center account is the recommended way to be ready. No revenue-share or in-app payments model has been announced.

DAT apps are phone companion apps. They do not run on the glasses themselves. To share beyond personal use today:

- Private invite-only release channels (one Beta channel per integration).
- Email invites to Meta accounts.
- Tester limit is shown on the channel Users tab.

Public publishing so any glasses owner can register without an invite remains partner-gated or not generally open. Meta’s iOS integration docs (updated ~Oct 29, 2026) state that publishing to the App Store is not currently supported. The SDK uses the ExternalAccessory framework, which leads to App Store rejection under Apple’s MFi program and privacy manifest requirements. Community reports and prior Meta staff comments confirm developers are limited to internal testing (Developer Mode + TestFlight for invited users) until Meta removes the MFi requirement and opens publishing.

Android / Google Play is more straightforward in principle, but non-developer users still need Meta registration or a release-channel invite.

Display Web Apps (Ray-Ban Display only) are a separate path: HTTPS URL + developer mode, no store. Unofficial community catalogs exist. This path is not suitable for displayless Ray-Ban Meta reading capture.

Monetization is plausible later (paid companion app, subscription, or bring-your-own API keys; Notion stays user-owned) once Meta and Apple open proper distribution. Today the practical audience is the builder plus invited testers. Distribution, not the product idea, is the current constraint.

## Open Decisions / Notes for the Agent

- Prefer forking or heavily referencing RayBan_Meta_NoteBuddy or the official BirdSpotter sample rather than greenfield if it accelerates the DAT integration.
- Confirm current DAT speech/ASR vs raw mic path on the user’s glasses firmware. Docs note speech (beta) and experimental PCM.
- Capture LED on the glasses lights during photo; cannot be disabled.
- Free Apple ID signing expires every 7 days — document the reinstall step.
- Notion Free plan 5 MB upload: compress photos or skip attachment in v1.
- Store API keys in Keychain, never in the repo.
- Test with Mock Device Kit first if hardware is not immediately available, then real glasses.

## Success Criteria (v1)

User can:
1. Start a session in the app.
2. Take a glasses photo of a book page.
3. Speak a short thought immediately after.
4. See a new Notion page appear with the OCR text and the transcribed thought linked together.

No manual “send to” command required after the photo.
