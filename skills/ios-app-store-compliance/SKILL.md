---
name: ios-app-store-compliance
description: Pre-submission review for an iOS app. Use before an App Store submission, when a build is rejected, or when auditing a legacy app for current store requirements.
---

# App Store pre-submission review

> **Verify against current documentation.** App Review requirements change on
> Apple's schedule, and any fixed list goes stale. Treat this as a checklist of
> *where to look*, and confirm specifics against Apple's current developer
> documentation and App Store Review Guidelines before relying on them. If a
> rejection cites a rule not covered here, the rules changed, not the app.

## Start with the rejection, if there is one

App Store Connect rejections cite a guideline number and usually a specific
screen or API. Read the actual message before changing anything — most rejections
are narrower than they first appear, and a broad rewrite in response to a narrow
complaint is a common waste of a review cycle.

`ITMS-` prefixed errors are different: they come from the upload pipeline, not
review, and are almost always a build configuration problem rather than a policy
one.

## Privacy

This is where legacy apps most often fail, because the requirements arrived after
the app was written.

- **Privacy manifest** (`PrivacyInfo.xcprivacy`). Required for the app and for
  bundled third-party SDKs. Check that one exists, that it lists data collection
  accurately, and that SDKs shipping their own manifests are on current versions.
- **Required-reason APIs.** Certain common APIs — file timestamps, disk space,
  system boot time, user defaults among them — must declare an approved reason
  code in the manifest. Legacy code uses these casually.
- **Usage-description strings** in `Info.plist` for every permission the binary
  can request: camera, photos, location, microphone, contacts, tracking. A
  missing string is a crash on first use, not a warning. An unused framework that
  can request a permission still needs the string or should be removed.
- **App Tracking Transparency.** If anything resembling tracking happens, the ATT
  prompt is required, and the privacy nutrition label in App Store Connect must
  match what the code actually does.

## Build configuration

- Deployment target and SDK version meet Apple's current minimums — these are
  raised periodically and are a hard upload rejection.
- Entitlements match provisioning: push, app groups, keychain sharing, associated
  domains. A mismatch fails at upload.
- No simulator slices, no development-only code paths, no debug logging of user
  data.
- Export compliance answered correctly. Most apps use HTTPS and therefore use
  encryption; the exemption question has a right answer for your case and it is
  not automatically "no".
- Icons and launch assets complete for every required size.

## Deprecated and removed APIs

Legacy iOS codebases accumulate these, and they move from deprecated to rejected
over time. Grep for APIs that Apple has announced timelines for, and check the
current status of each rather than assuming the deadline has not arrived.

Anything private is a hard rejection: no underscore-prefixed system methods, no
`performSelector:` reaching a private API by name to dodge a symbol check. That
last one is detected.

## Content and behavior

- Sign in with Apple must be offered if any third-party social login is.
- Digital goods and subscriptions go through In-App Purchase. Steering users to
  outside payment is a rejection.
- Account deletion must be available in-app if the app supports account creation.
- Demo credentials in App Review notes if any part of the app is behind a login.
  A reviewer who cannot get in rejects the build.
- Nothing in the binary or metadata that references a platform other than Apple's,
  or an unreleased OS.

## Before submitting

1. Archive a release build with the distribution profile and confirm it uploads.
2. Install that archived build on a device and launch it — a release build can
   fail where debug succeeds, particularly around entitlements and optimization.
3. Walk every permission prompt.
4. Confirm the privacy label in App Store Connect matches the manifest.

## Reporting

For each finding: what will happen (rejection, upload failure, crash), which
part of the app, and the smallest change that fixes it. Distinguish "this will be
rejected" from "this might be questioned" — spending a review cycle on a maybe is
expensive, and so is shipping a definite.
