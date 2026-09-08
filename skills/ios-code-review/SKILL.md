---
name: ios-code-review
description: Review Objective-C and Swift changes in an iOS codebase. Use when reviewing a diff or PR, checking a change before merge, or auditing code for memory, threading and lifecycle problems.
---

# Reviewing iOS code

Review in this order. The first three catch the bugs that reach users; the rest
catch the ones that reach the next maintainer.

## 1. Memory and lifecycle

**Retain cycles.** The recurring ones:

- A block captures `self` strongly. Look for `self.` or a bare ivar inside any
  block stored as a property, passed to a long-lived object, or used as a
  completion handler that outlives the call. The `__weak typeof(self) weakSelf`
  dance in Objective-C, `[weak self]` in Swift.
- A delegate declared `strong` rather than `weak`. Check the property attributes
  on every delegate, every time.
- `NSTimer` retains its target. So does `CADisplayLink`. A view controller that
  schedules one and does not invalidate it in `dealloc`/`deinit` never
  deallocates — and `dealloc` will never run, so a cleanup you put there is dead
  code. Invalidate on disappear, not on dealloc.
- Parent-child object graphs where the child holds a strong back-reference.

**Observer teardown.** Every `addObserver:` needs a matching `removeObserver:` on
a path that actually runs. KVO on a deallocated observer is a crash, not a leak.
`NSNotificationCenter` is auto-removed on modern runtimes; KVO is not.

**`dealloc` correctness.** No calls to methods that could resurrect `self`, no
work that assumes properties are still valid.

## 2. Threading

- **UIKit is main-thread only.** Any UI touched from a completion handler,
  `URLSession` callback, or GCD block needs an explicit hop to main. Network
  callbacks in particular are not on main, and the bug reproduces intermittently.
- **Mutable shared state.** `NSMutableArray` and `NSMutableDictionary` are not
  thread-safe. Look for one mutated from more than one queue.
- **Deadlocks.** `dispatch_sync` onto the current queue. `dispatch_sync` to main
  from anything that might already be main.
- **Core Data** managed object contexts are confined to the queue that created
  them, and managed objects cannot cross contexts. Check `perform`/`performAndWait`
  wrapping.

## 3. Correctness at the edges

- **Force unwraps and force casts** in Swift (`!`, `as!`). Each one is a crash
  the compiler has been told not to prevent. Ask what happens when it is nil.
- **Implicitly-unwrapped optionals from Objective-C.** An unannotated Objective-C
  header imports as IUO, so a nil that Swift thinks is impossible crashes at the
  use site rather than the source.
- **Array and string bounds.** `objectAtIndex:` on an empty array; range
  arithmetic on a string containing emoji or combining characters.
- **Error paths.** `NSError **` that is set but never checked, `try?` swallowing
  a failure that should surface.

## 4. Objective-C specifics

- Property attributes: `copy` for `NSString` and block properties, not `strong` —
  a mutable string assigned to a `strong` property can be mutated behind you.
- `nonatomic` almost always; `atomic` is a performance cost that does not deliver
  thread safety anyway.
- Direct ivar access (`_foo`) outside `init` and `dealloc` bypasses property
  side effects and KVO notifications.
- New categories on framework classes must use a prefix (`bw_foo`). An
  unprefixed category method silently wins or loses against another category or
  a future OS method, and the failure is undiagnosable.

## 5. Change safety

- Renaming anything: was `find_dynamic_dispatch` run? A method reached by
  `@selector`, KVC, or a XIB does not appear in compile-time references.
- Deleting anything: is it named in a XIB, storyboard or `Info.plist`?
- Changing a widely-imported header: `import_graph` `mode: "importedBy"` to see
  the blast radius.

## What not to raise

Skip formatting, brace style, and naming preferences unless the project has a
written convention being violated. They crowd out the findings above, and in a
legacy codebase there is an unbounded supply of them.

State findings with the failure they cause: "this timer is never invalidated, so
the controller never deallocates and its observers keep firing after dismissal" —
not "consider invalidating the timer".
