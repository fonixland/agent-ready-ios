<!--
  Project memory skeleton for a legacy iOS codebase.

  Fill in every section, delete the ones that do not apply, and delete these
  comments. Keep the finished file short — under roughly 150 lines. A long file
  goes stale, and a stale file is worse than no file because it gets believed.

  The test for including something: could an agent work it out by reading the
  code? If yes, leave it out. This file is for what the code cannot say.
-->

# <App name>

<One paragraph: what the app does, who uses it, and roughly how old the codebase
is. Age is genuinely useful context — it tells an agent which era's patterns to
expect.>

## Languages and frameworks

- Objective-C: <rough share, and where it dominates>
- Swift: <rough share, and whether new code is expected to be Swift>
- UI: <UIKit / SwiftUI / mixed, and which is used for new screens>
- Minimum deployment target: <version>
- Dependencies: <CocoaPods / SPM / Carthage / vendored — and which file is
  authoritative>

## Building

```
<the exact command, including -workspace vs -project and the scheme name>
```

- Open the **workspace**, not the project, if CocoaPods is in use.
- Schemes: <which one is the app, which are tests, which are internal-only>
- <Anything non-obvious: code generation steps, required env vars, a script that
  must run first, credentials needed for a full build>

## Architecture

<Where things live and who owns what. Directory names are unreliable in an old
codebase — say what is actually true. Three to six bullets.>

- <e.g. Networking goes through XNetworkClient; nothing else should construct
  URLSession tasks directly.>
- <e.g. Persistence is Core Data, but the older FooStore SQLite layer is still
  live for the Bar feature.>

## House rules

<Conventions that are enforced but not obvious from reading. Be specific about
what an agent should do differently from its default.>

- New Objective-C categories on framework classes use the `<xx_>` prefix.
- <e.g. New screens are UIKit, not SwiftUI, until the design system is ported.>
- <e.g. Do not add new singletons; use the existing container.>

## Frozen and hazardous areas

<The most valuable section. Where should an agent stop and ask?>

- `<path>` — <why it is frozen: pending rewrite, no test coverage, owned by
  another team, legally sensitive>
- <e.g. Anything under Payments/ requires review by <name> before merge.>

## Runtime-resolved references

<Objective-C reaches code by string in ways the compiler cannot see, so a rename
can break the app while still compiling. Note the hot spots here.>

- XIBs and storyboards reference view controllers by `customClass` — treat them
  as source.
- KVO is used in <where>.
- <Anything instantiated via NSClassFromString or configured by string keys.>

**Before renaming or deleting any class, method or property, check for
string-based references** (`find_dynamic_dispatch` if `objc-atlas` is wired up).

## Testing

- Run: `<command>`
- Coverage is <honest description — "thin outside the networking layer" is more
  useful than a number>.
- <What is expected of new code: tests required, or not.>

## Release

- <Branch model, who cuts releases, cadence.>
- <Anything an agent should never do: push to a release branch, bump a version,
  change entitlements or the privacy manifest without review.>

## Known problems

<Things that will look like bugs but are known and deliberate, so an agent does
not "fix" them.>

- <e.g. The duplicate model classes in Legacy/ are intentional during migration;
  do not consolidate them.>
