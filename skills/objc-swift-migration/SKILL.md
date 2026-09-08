---
name: objc-swift-migration
description: Migrate one Objective-C class to Swift without runtime breakage. Use when converting a class to Swift, planning a migration order, or assessing how hard a given class would be to migrate.
---

# Migrating one Objective-C class to Swift

Migrate **one class at a time**, and build between each. A batch migration that
fails leaves you unable to tell which change broke it.

## Pick the right class first

Use `migration_candidates` if `objc-atlas` is available. It ranks classes by
difficulty and shows the factors behind each score, so you can disagree with the
ranking rather than trust it.

If you are choosing by hand, difficulty is dominated by these, roughly in order:

1. **Dependents.** A leaf class is a contained change. A class fifty files import
   means fifty files gain a bridging-header dependency.
2. **Runtime references.** See below. This is the one that bites later.
3. **Categories on the class.** Swift extensions cannot add stored properties, so
   category state must move into the class or stay behind in a shim.
4. **Objective-C++ (`.mm`).** Close to a stopper. Swift cannot import C++ types
   through a bridging header, so the class needs a C or Obj-C wrapper first.
   Deprioritize these unless the wrapper already exists.
5. **Macro density.** Macros do not survive migration and must be rewritten by
   hand as functions, constants or generics.

Good first candidates are leaves: a model object, a formatter, a small utility
with no categories and no dynamic references.

## Before writing any Swift

Run `find_dynamic_dispatch` for the class name and for every method you are about
to move. Check XIBs and storyboards for `customClass`, and `Info.plist` for the
class name.

Everything you find is a contract you must preserve. In Swift that means explicit
`@objc` annotations, and `@objcMembers` or `dynamic` where the Objective-C runtime
needs to see the member:

- A method invoked via `@selector` or target/action needs `@objc`.
- A property read or written through KVC needs `@objc`.
- A property **observed** through KVO needs `@objc dynamic` — `@objc` alone is
  not enough, because KVO requires dynamic dispatch and Swift will devirtualize
  otherwise. This is the most common silent breakage in a migration.
- A class instantiated by name needs `@objc(OriginalName)` to keep its
  Objective-C runtime name, since Swift otherwise mangles it with the module
  prefix. A XIB referencing `FooView` will not find `MyApp.FooView`.

## Preserve the interface before improving it

Migrate the class with the same public surface, build, and confirm the app still
works. Only then improve it.

Doing both at once means that when something breaks you cannot tell whether it
was the language change or the redesign. Two commits, in that order.

Specifically, resist on the first pass:
- turning `NSError **` into `throws`
- turning nullable returns into optionals beyond what the header already declares
- collapsing a class into a struct — value semantics change behavior in ways
  callers may depend on
- renaming anything

## Nullability

If the Objective-C header lacks `NS_ASSUME_NONNULL_BEGIN` / nullability
annotations, everything imports into Swift as implicitly-unwrapped optional. That
compiles and then crashes.

Annotate the Objective-C header first, as its own commit, before migrating.
Deciding what is genuinely nullable is the actual work of the migration, and it
is easier to do while the Objective-C is still in front of you.

## Verify

After each class:

1. `xcode_build` — read the structured errors, fix, repeat.
2. `xcode_test` — run the tests.
3. **Exercise the runtime paths the compiler cannot check.** If the class is
   reached from a XIB, open that screen. If it has KVO observers, trigger them.
   The build passing is not evidence here; that is the entire hazard.

## When to stop

If a class needs more than a couple of `@objc` escape hatches to keep its
callers working, it is not ready. Migrate its callers first, or leave it in
Objective-C. A permanently mixed codebase is a normal, fine outcome — migrating
everything is not the goal, and a half-migrated class with a compatibility shim
around it is worse than one that was left alone.
