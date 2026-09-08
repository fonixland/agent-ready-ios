---
name: legacy-objc-orientation
description: Orient in an unfamiliar Objective-C or mixed Obj-C/Swift codebase before changing anything. Use at the start of any task in a legacy iOS project — locating a class, tracing behavior, planning a change, or answering "where does X happen".
---

# Orienting in a legacy Objective-C codebase

Your instinct will be to grep. In this codebase that instinct is wrong, and
following it will exhaust your context before you understand anything.

Searching a large Objective-C project for a class name returns thousands of hits
across headers, implementations, categories, comments and dead code. Reading even
a tenth of them costs more context than the task itself.

## Do this instead

Use structural queries. If `objc-atlas` is available, prefer it over any text
search:

1. **`codebase_overview`** once per session. Size, language mix, the
   most-imported headers, how much of the code is reached dynamically.
2. **`find_symbol`** to locate something. Returns declaration and implementation
   sites, not every textual mention.
3. **`class_detail`** before reasoning about any class. This is the important
   one — see below.
4. **`import_graph`** with `mode: "importedBy"` before changing a header, to see
   the blast radius.

Fall back to text search only for string literals, comments, and build settings —
things with no structure to query.

## Never reason about a class from one file

An Objective-C class is not in one place:

- `@interface` is in the header, `@implementation` in the `.m`.
- A **class extension** — `@interface Foo ()` — usually sits at the top of the
  `.m` and declares the private surface.
- **Categories** add methods from files that may not mention the class in their
  name, and those methods are as real as any other.

`class_detail` assembles all of this and tags each method with the category it
arrived through. Reading `Foo.m` alone gives you a partial class and no warning
that it is partial.

## Check the runtime edges before you touch a name

This is the step that prevents the expensive mistakes. Objective-C reaches code
by string in more places than most languages:

| Mechanism | Looks like |
| --- | --- |
| Target/action | `addTarget:action:@selector(tap:)` |
| Deferred calls | `performSelector:`, `NSInvocation` |
| KVC / KVO | `valueForKey:@"count"`, `forKeyPath:` |
| Dynamic classes | `NSClassFromString(@"FooController")` |
| Interface Builder | `customClass="FooView"` in a XIB or storyboard |
| App launch | `NSPrincipalClass` and friends in `Info.plist` |

None of these are compile-time references. **Before renaming or deleting any
method, class or property**, run `find_dynamic_dispatch` against its name. A
rename that ignores them compiles cleanly and fails at runtime, often far from
the change, sometimes only on a screen nobody opens during testing.

Treat XIBs, storyboards and `Info.plist` as source. They are.

## Categories on framework classes

Run `list_categories` with `onExternalOnly: true` early. A codebase that has
extended `NSString`, `UIView` or `NSDate` has changed the behavior of types you
are assuming are stock. You will not notice this by reading call sites.

## Reading the directory tree

Don't infer architecture from folder names. A codebase this old has been
reorganized repeatedly, and the layout records the team's history rather than the
app's structure. `import_graph` with `mode: "fanIn"` tells you what is actually
central; the directory named `Core` frequently is not.

## Before you report back

State plainly what you did not verify. In this codebase the honest answer often
includes "I did not trace the KVO observers" or "there may be category methods in
files I did not open". That is more useful than a confident summary that omits it.
