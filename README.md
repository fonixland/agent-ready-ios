# Making a legacy iOS codebase agent-ready

A method, and the skills that implement it, for getting AI coding agents to work
usefully inside large Objective-C and mixed Objective-C/Swift codebases.

Most "AI in your codebase" advice assumes a greenfield TypeScript repo. A
fifteen-year-old iOS app is a different problem. The code is fine — it shipped,
it makes money — but almost everything an agent needs in order to reason about
it is either implicit, spread across files, or expressed at runtime rather than
in syntax. Dropping an agent into that repo produces confident, wrong work.

This repository is what I install when a client asks me to fix that: four skills,
a `CLAUDE.md` template, and an MCP configuration. The reasoning behind each is
below, because the reasoning transfers even if your stack does not.

## Why agents struggle here specifically

Five properties of Objective-C codebases break the assumptions agents carry in.

**Declarations and definitions are separated.** `@interface` in one file,
`@implementation` in another, and neither is the whole class. An agent that reads
`Foo.m` has not read `Foo`.

**Categories add behavior from anywhere.** A method on `Foo` can be declared in
`Foo+Analytics.h`, implemented in a file that never mentions `Foo` in its name,
and be entirely absent from `Foo.h`. Categories on framework classes —
`NSString+Whatever` — are worse: the agent has no reason to suspect `NSString`
has been extended, so it reasons about stock behavior that no longer applies.

**Half the wiring is strings.** Target/action, KVC, KVO, the responder chain,
`NSInvocation`, `NSClassFromString`, and every XIB and storyboard reach code by
name, not by reference. An agent that renames a method by following compile-time
references produces something that builds cleanly and crashes at runtime. This is
the single most expensive failure mode on this list, because the feedback comes
late and looks unrelated.

**Nothing is where the file tree suggests.** Twelve years of reorganizations
leave a directory layout that reflects the team's history rather than the app's
architecture. Directory names are not a reliable guide to responsibility.

**Build output is enormous.** A single `xcodebuild` invocation emits tens of
thousands of lines, and one compiler command line alone routinely exceeds four
thousand characters. An agent that runs a build and reads the output has spent
its context and learned nothing.

The failure that follows from all five is the same: the agent burns its budget on
orientation, gets a partial picture, and acts on it.

## The method

Four steps, in this order. The order matters — each one makes the next cheaper.

### 1. Give the agent a map instead of a filesystem

Grep is the wrong instrument. Searching a large codebase for a class name returns
thousands of hits across headers, implementations, categories and comments, and
reading even a fraction exhausts the context window.

Replace it with structural queries. [`objc-atlas`](https://github.com/fonixland/objc-atlas)
is the MCP server I built for this: it indexes the codebase once and answers
questions like "what does this class actually respond to, including methods
arriving through categories declared elsewhere" in one call. `templates/mcp.json`
wires it up.

The measurable difference is context spent before useful work begins. One
`class_detail` call replaces reading four or five files.

### 2. Write down what the code cannot say

Some facts are simply not in the source: which module owns which concern, which
areas are frozen, which patterns are deprecated but not yet removed, what the
release process is. Left unwritten, the agent infers them from whatever file it
happened to read.

`templates/CLAUDE.md` is the skeleton I fill in with a client. It is deliberately
short. A long file goes stale, and a stale file is worse than none because it is
believed.

### 3. Make the runtime edges explicit

This is the step people skip, and it is the one that prevents the expensive
failures. Before any rename or migration, the string-dispatched references have
to be surfaced — `@selector`, `NSClassFromString`, KVC keys, XIB `customClass`
entries, `Info.plist` principal classes.

`skills/legacy-objc-orientation` and `skills/objc-swift-migration` both make this
a required step rather than a suggestion, because an agent will otherwise treat
compile-time references as the complete picture.

### 4. Close the loop with a real build

An agent that cannot build cannot check itself, so it substitutes confidence for
verification. But the loop only works if build output arrives distilled: pass or
fail, and the errors as structured rows with file, line and column.

Headless matters here. Xcode 26 ships its own MCP server at
`Developer/usr/bin/mcpbridge`, and it is good, but it attaches to a *running*
Xcode instance and exits without one. That rules out CI, SSH, and any unattended
run. `objc-atlas` covers the headless case.

## What a first pass turns up

The worked example below is [AFNetworking](https://github.com/AFNetworking/AFNetworking),
because it is public and every number is reproducible with
`objc-atlas --report`. It is a well-maintained library rather than a neglected
app, which makes it a conservative example — a real legacy app is worse.

At 82 files and 15,581 lines it has:

- **30 categories**, 11 of them on framework classes the codebase does not own.
- **193 runtime-resolved references** — 147 `@selector` literals, 36 KVC keys, 10
  `NSClassFromString`. Every one is invisible to a rename that follows only
  compile-time references.
- **A hot header imported by 21 files**, which is the real reason incremental
  builds are slow.

The dead-code result is the instructive one. Naively, 32 classes look
unreferenced. After suppressing classes instantiated from XIBs, named in
`Info.plist`, discovered by the test runner as `XCTestCase` subclasses, reached
via `NSClassFromString`, kept alive as superclasses, and declared file-private
inside a `.m`, the correct answer is **zero**.

That gap between 32 and 0 is the whole argument for doing this deliberately. An
agent given the naive analysis will confidently propose deleting 32 classes, and
some of that will look reasonable in review.

## What's here

```
skills/
  legacy-objc-orientation/     How to orient in an unfamiliar Obj-C codebase
  objc-swift-migration/        Migrating one class without runtime breakage
  ios-app-store-compliance/    Pre-submission review
  ios-code-review/             Review checklist for Obj-C and Swift
templates/
  CLAUDE.md                    Project memory skeleton
  mcp.json                     MCP server wiring
```

Skills are plain directories with a `SKILL.md`. Copy the ones you want into your
project's skills directory, or point your agent at this repo.

## Honest limits

- The skills encode judgment, not guarantees. They make the right steps the
  default; they do not make a wrong migration impossible.
- `objc-atlas` uses a heuristic parser rather than a compiler frontend, so
  macro-generated declarations are invisible to it. That is a deliberate trade so
  it can index codebases that do not build cleanly — which is most of the ones
  worth doing this to.
- Nothing here removes the need for someone who knows the codebase to review the
  result. The goal is an agent that is useful under supervision, not unsupervised.

## License

MIT © Bill Weakley

I do this work for clients as [Bill Weakley LLC](mailto:mc_fonix@mac.com) —
legacy Objective-C modernization, and making existing iOS codebases agent-ready.
