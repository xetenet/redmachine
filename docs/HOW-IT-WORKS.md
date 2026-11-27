# How RedMachine works

The reference for RedMachine 0.1.0: every option, rule, verdict, message and limit, each in one place. Start with the [README](../README.md). Output samples are pasted from runs of this build; paths are shortened to `/Users/you/`.

## Contents

1. [Options](#1-options)
2. [A run, step by step](#2-a-run-step-by-step)
3. [Projects and languages](#3-projects-and-languages)
   1. [Built-in languages](#31-built-in-languages)
   2. [How a project is found](#32-how-a-project-is-found)
   3. [Several projects in one repository](#33-several-projects-in-one-repository)
   4. [Source with no built-in support](#34-source-with-no-built-in-support)
   5. [Your own commands](#35-your-own-commands)
   6. [Swift: the copy](#36-swift-the-copy)
   7. [Xcode projects](#37-xcode-projects)
4. [Which lines are judged](#4-which-lines-are-judged)
   1. [The change](#41-the-change)
   2. [Lines counted although not edited](#42-lines-counted-although-not-edited)
   3. [Files set aside](#43-files-set-aside)
   4. [Lines left out](#44-lines-left-out)
5. [How a line is broken](#5-how-a-line-is-broken)
   1. [The breaks](#51-the-breaks)
   2. [Order, and strict mode](#52-order-and-strict-mode)
   3. [Lines no build reads](#53-lines-no-build-reads)
6. [Output, verdicts and results](#6-output-verdicts-and-results)
   1. [Output lines](#61-output-lines)
   2. [Verdicts](#62-verdicts)
   3. [RESULT lines](#63-result-lines)
   4. [Exit codes](#64-exit-codes)
7. [Time, memory and processes](#7-time-memory-and-processes)
   1. [Time limits](#71-time-limits)
   2. [Memory limit](#72-memory-limit)
   3. [Processes](#73-processes)
   4. [The check after the run, and confirm-kills](#74-the-check-after-the-run-and-confirm-kills)
8. [Your files](#8-your-files)
   1. [Restore](#81-restore)
   2. [Interrupts and recovery](#82-interrupts-and-recovery)
9. [Waivers](#9-waivers)
10. [The receipt and verify](#10-the-receipt-and-verify)
    1. [Records](#101-records)
    2. [The proof id](#102-the-proof-id)
    3. [Verify](#103-verify)
11. [What stops a run](#11-what-stops-a-run)
12. [Limits](#12-limits)
13. [Licence](#13-licence)
14. [Index](#index)

## 1. Options

```
redmachine [options]                      judge the change in this git repository
redmachine verify RECEIPT [--expect ID] [--ran TEST]   check a receipt and print its proof id
redmachine license | activate KEY         show the licence, or activate a key
redmachine --help | --version             (-h, -V)
```

Every option that takes a value also takes `--name=value`. An unknown option stops the run (exit 2). Judging needs a licence ([13](#13-licence)); `--help`, `--version`, `--list`, `verify`, `license` and `activate` do not.

| Option | What it does |
| --- | --- |
| `--base REF` | Compare against this commit or branch. `HEAD`: uncommitted work only. `ROOT`: every tracked file is judged as new. Default: [4.1](#41-the-change). |
| `--only PREFIX` | Judge only files under this path from the repository top. May repeat. Matches whole path parts: `src` matches `src/a.rs`, not `srcs/a.rs`. |
| `--exclude PREFIX` | Leave out files under this path. May repeat. |
| `--test CMD` | The command that runs the tests, run with `sh -c` in the project folder. Exit 0 is green, anything else red. For an Xcode project, a scheme name picks the scheme ([3.7](#37-xcode-projects)). |
| `--check CMD` | The command that says a broken file still builds, exit 0 meaning yes. `{file}` is the broken file's path from the project folder, quoted. `--check true` checks nothing ([3.5](#35-your-own-commands)). |
| `--timeout SECS` | Limit for one test run of a broken line, and the least a check may take. Default: [7.1](#71-time-limits). |
| `--max N` | Run the tests for at most N broken lines (default 60). Only KILLED and SURVIVED lines count. The rest are NOT_TRIED. |
| `--max-memory MB` | Stop a run whose processes hold more than this. `0` turns it off. Default: [7.2](#72-memory-limit). |
| `--list` | Show what would be judged and with which commands. Breaks nothing, runs no check or test, writes no receipt. Exit 0, also with `problem:` lines. |
| `--profile NAME` | Judge only `rust`, `python`, `node`, `swift` or `go` files. See [3.3](#33-several-projects-in-one-repository). |
| `--ext LIST` | Your own source extensions, comma-separated (`c,h`), in any case. Needs `--test` and `--check`. With `--profile`, replaces that language's extensions. See [3.5](#35-your-own-commands). |
| `--hash-comments` | With `--ext`: lines starting with `#` are comments. |
| `--mac-app-on-screen` | Let Mac app tests open the app ([3.7](#37-xcode-projects)). |
| `--receipt PATH` | Where to write the receipt. Default `redmachine-receipt.jsonl` at the repository top. A relative path is relative to where you ran RedMachine. |
| `--strict` | Every break that builds must turn a test red ([5.2](#52-order-and-strict-mode)). |
| `--confirm-kills` | After each KILLED line, run the unbroken tests again ([7.4](#74-the-check-after-the-run-and-confirm-kills)). |
| `--expect ID` | Fail (exit 1) unless the proof id is exactly ID. The receipt is still written. |

## 2. A run, step by step

1. Check the licence ([13](#13-licence)). Without one, stop (exit 2) unless this is `--list`.
2. Take the repository lock (`.git/redmachine/lock`), clean up after a cut-off earlier run ([8.2](#82-interrupts-and-recovery)), and remove an earlier receipt.
3. Find the base, the common commit and the changed lines ([4](#4-which-lines-are-judged)). On macOS, if a Swift file changed, work in a copy from here on ([3.6](#36-swift-the-copy)).
4. Match each changed file to a project ([3](#3-projects-and-languages)), set aside the files not judged and leave out the lines that are not behaviour.
5. Print the header and read `.redmachine-waive`. With `--list`, print the lines and stop.
6. Run the baseline: each project's check, then its tests, with nothing broken. If either is red, every counted line is NOT_TRIED.
7. For each counted line, in path and line order: WAIVED if a waiver matches; otherwise try the breaks in order: write the broken file, run the check, run the tests if the check passed, put the file back.
8. If any line was KILLED, run the unbroken tests once more ([7.4](#74-the-check-after-the-run-and-confirm-kills)).
9. Compare the working tree with how it started, write the receipt, print the summary.

## 3. Projects and languages

### 3.1 Built-in languages

| Language (profile) | Extensions | Test command | Check command |
| --- | --- | --- | --- |
| Rust (`rust`) | `rs` | `cargo test --workspace --quiet` | `cargo test --workspace --no-run --quiet` |
| Python (`python`) | `py`, `pyw`; a file with no extension and a Python `#!` first line | `python3 -m pytest -x -q -p no:cacheprovider` (no `.pytest_cache`) when pytest can be imported, else `python3 -m unittest -f -q` | a `python3` compile of the file |
| JavaScript, TypeScript (`node`) | `js mjs cjs jsx ts tsx mts cts` | `npm test --silent` | `node --check {file}` for `js mjs cjs`; `tsc --noEmit -p .` for the rest |
| Go (`go`) | `go` | `go test -count=1 ./...` | `go test -count=1 -run '^$' ./...` |
| Swift (`swift`), preview | `swift` | package: `swift test`; Xcode: [3.7](#37-xcode-projects) | package: `swift build --build-tests` |

- Platforms: macOS (Apple silicon and Intel), Linux (x86_64 and arm64) and Windows through WSL 2, where the Linux program runs. Xcode projects need a Mac with Xcode.
- Preview: Swift and Xcode support is new in this version. It was timed only on one iPhone app with 9 edited lines and one 119-line SwiftUI screen in a Swift package, and its SwiftUI rules read text rather than parse code.
- Commands run in the project folder with `sh -c`, empty standard input and `PYTHONDONTWRITEBYTECODE=1`. Their output goes to `.git/redmachine/run.log`, which holds the last command's output after a run. Each tool must be on the `PATH`.
- TypeScript, TSX, JSX, MTS and CTS files need a `tsconfig.json` in the project and `tsc`: the nearest `node_modules/.bin/tsc` up to the repository top, else `tsc` on the `PATH`. A file in no tsconfig program cannot be checked and stops the run; a `.jsx` file needs `allowJs`.

### 3.2 How a project is found

Each changed file belongs to the nearest project folder above it, up to the repository top.

| Language | The project folder is the nearest one with |
| --- | --- |
| Rust | `Cargo.toml`. A crate inside a Cargo workspace is judged as the workspace. |
| Python | `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements.txt`, `pytest.ini` or `tox.ini`. With none, the repository top. |
| Node | a `package.json` whose top-level `"scripts"` has a `"test"` entry. With none above the file, the nearest `package.json`. |
| Go | `go.mod`. Modules a `go.work` lists are one project at the `go.work` folder; a module it does not list is its own project, run with `GOWORK=off`. |
| Swift | an `.xcworkspace`, else an `.xcodeproj`, else `Package.swift`. |

- A source file with no project above it (Python excepted) is not judged: it is named in a note and a `problem:` line, and the run fails. When nothing in the change has a project, the run stops (exit 2).
- A project file on a test path of an enclosing project (a `Cargo.toml` under a crate's `tests/`) is not a project; files under it are test files.
- Build output is never judged: untracked files under `target/` beside a `Cargo.toml`, `.build/` beside a `Package.swift`, `Pods/` beside a `Podfile`, `Carthage/` beside a `Cartfile`, `build/` and `DerivedData/` beside an Xcode project, the `outDir` of a `tsconfig.json`, and any `node_modules/` or `__pycache__/`.

### 3.3 Several projects in one repository

- One project at the repository top prints `profile: KIND`. A project in a subfolder, or several, print `projects: N` and a `project DIR (KIND)` block each.
- Each project has its own baseline and its own time limits. The memory limit is one for the run.
- `--test` and `--check` replace the commands of every project. When the repository has more than one project, they need `--profile`, or the run stops (exit 2).
- A project whose built-in tests ran no test at the baseline is set aside: its files are named and fail the run unless excluded, and the other projects are judged; a line after the notes gives the count still judged (`changed lines after setting aside python in .: 1 counted`). With no other project, the run stops (exit 2). A built-in command that exits 5 and says no tests ran (Python 3.12 and later, pytest) counts as running none; with `--test`, exit 5 is red.
- `--profile KIND` judges only that kind's files. Other supported source in the change is set aside, counted as `not source for this profile` and named in a note: a PASS does not cover it. Judge it in a second run, with its own `--receipt`.

### 3.4 Source with no built-in support

Changed `.kt`, `.kts`, `.java` and `.cs` files, and files whose extension is a supported one in upper case (`.PY`, `.RS`), are named in a note and fail the run, with `--profile` too, unless `--ext` names their extension. The rest of the change is judged, and the RESULT names them. When they are the whole change, the run stops (exit 2).

### 3.5 Your own commands

`--ext LIST --test CMD --check CMD` judges files with those extensions as one project at the repository top.

```
redmachine --ext rb --test 'ruby test_calc.rb' --check 'ruby -c {file}' --hash-comments --exclude test_calc.rb
```

Changed source of a built-in kind is set aside and named in a note, so a PASS does not cover it; source with no built-in support still fails the run ([3.4](#34-source-with-no-built-in-support)).

- `--check true` checks nothing, so a break that stops the build reads as KILLED, and a note says so. Use it only when there is nothing to build.
- A check that reads one file cannot see a break that stops another file from building; that break reads as KILLED. A check that builds the whole program can.
- Without `--hash-comments`, `//` and `/* */` are comments and `#` lines are code (C preprocessor lines). With it, only `#` starts a comment.
- RedMachine does not know which of your files are tests: leave them out with `--exclude`.
- Declarations are not recognised.

### 3.6 Swift: the copy

On macOS, when the change has a `.swift` file, or with `--profile swift`, the whole run works in a copy of the repository, every language in it. Your files are never written.

- The copy is an APFS clone at `~/Library/Caches/redmachine/ID/tree`, or under `$REDMACHINE_CACHE`. It is deleted at the end of the run. Build folders stay, one per repository path and project. `~/Library/Caches/redmachine` can be deleted when no run is going.
- A Swift package's built-in commands add `--scratch-path {cache}/swiftpm.noindex/…`, so a later run reuses the build. If the baseline check fails in a kept build folder, the folder is deleted and the check runs once more, with a note. `--list` leaves `{cache}` unfilled, with a note.
- The header adds `copy: judged in a copy at PATH; your files, Xcode, previews and your simulators are not touched`.
- In your repository RedMachine writes only the receipt and `.git/redmachine/` (lock, run log, saved files).
- One copy run per Mac at a time; a second prints `waiting:` and starts when the first ends.
- Files you save during the run are judged as they were at the start; a note names them.
- The end line is `restore: your files were never written; the copy holds every judged file as it was read`. If the build or the tests rewrote a judged file in the copy, the run fails with exit 1, not 3.
- Projects the copy refuses are named in [11](#11-what-stops-a-run).
- A tool installed to point at the original folder by its full path (`pip install -e`, `npm link`) tests the original, so the lines it covers SURVIVE.
- On Linux, Swift lines are broken in place, as for other languages ([8.1](#81-restore)).

### 3.7 Xcode projects

- **Needs:** a Mac with Xcode, a simulator runtime for the app (iOS for an iPhone or iPad app, tvOS for an Apple TV app) (Xcode > Settings > Components), and a unit-test target with at least one passing test.
- **Project and scheme:** two workspaces or two projects in one folder stop the run, and `--only` cannot choose between them: judge such a folder with `--ext swift --check CMD --test CMD` ([3.5](#35-your-own-commands)). The scheme is the one with tests. When several have tests, one named like the project wins, and a note says so. Without one, the run stops and lists them: name one with `redmachine --test SCHEME` (a `--test` value that is exactly a scheme name picks it). With no tests in a scheme, the run stops before any build ([11](#11-what-stops-a-run)).
- **Platforms:** iPhone, iPad and Mac apps; a multiplatform app is tested on the iPhone simulator. An app is iPad-only when every app target of the scheme has its device family set to iPad alone; it is tested on RedMachine's own iPad simulator, and any other mix keeps the iPhone simulator. Apple TV apps are supported, tested on RedMachine's own Apple TV simulator. Apple Watch and Vision Pro apps are coming soon; until then a run on one stops and says so. Its own simulator copies the device model and OS version of the simulator you last used (the one running now, or else the one booted most recently); with none of the right kind, RedMachine picks the newest OS and a default device, and a line at the start says which. RedMachine never starts, stops, changes or deletes your simulator. Physical devices are never used. Mac tests that open the app run only with `--mac-app-on-screen`: the app opens on your screen with broken code in it and uses its real data on this Mac, so quit your own copy first.
- **The copy:** lines are broken, built and tested in the copy ([3.6](#36-swift-the-copy)), never in Xcode's DerivedData.
- **RedMachine's simulator:** an iPhone app is tested on `redmachine tests (do not use)`, an iPad-only app on `redmachine iPad tests (do not use)` and an Apple TV app on `redmachine Apple TV tests (do not use)`, simulators RedMachine makes for itself; one run uses one of them. Each is created from the simulator you last used (same device model and OS version) and made again when that changes. Only RedMachine's own simulators are started, stopped, made or deleted. Before every test run RedMachine starts it again if something shut it down, and removes your app from it; at the end it shuts it down. If you are using the one the run needs, the run stops and touches nothing. For the memory it needs, see [7.2](#72-memory-limit).
- **Red:** with the built-in tests, a red run counts only when a real test failed, as Xcode's result bundle records it; Swift Testing and XCTest both count. If the test runner could not start, the tests run once more after 3 seconds. If the app crashes before any test runs, the simulator is restarted and the line runs again. The line is `KILLED [before any test ran]` only if that happens again with the line broken and not with nothing broken; if it also happens with nothing broken, the run stops (exit 2). No test checked such a line: the summary, the RESULT, the receipt (`crash_only`) and `verify` count it apart.
- **Build failures:** only a compiler or linker failure means a break does not build: the next break is tried, and the line is UNVIABLE when none builds. When another step fails (a script phase such as SwiftLint, code signing), the break is set aside; if no other break builds, the line is NOT_TRIED and names the step (`the script phase "Lint" of target Demo`). After a build that exits other than 65, RedMachine builds the unbroken code; red stops the run (exit 2). If the app does not build with nothing broken, the red baseline says to build it in Xcode first (Product > Build For > Testing).
- **What runs and what is judged:** UI-test targets are built, but their tests are skipped for each line (a scheme with only UI tests runs them for each line). Test files are under a `Tests` folder or a folder named like a test target, or have a line starting `import XCTest`, `import Testing` or `@testable import`. `#Preview` blocks, `PreviewProvider` types and SwiftUI layout and display lines with literal arguments only are left out ([4.4](#44-lines-left-out)). Objective-C, C, C++ and Metal files, and tracked changes under `Pods/` or `Carthage/`, fail the run until left out with `--exclude`. Storyboards, xibs, asset catalogs, plists, entitlements and xcconfig files are not source.
- **View code:** unit tests cannot reach SwiftUI view code, so its judged lines usually survive. A note after the survivor list counts them, names their files and says what to do (a model type, a UI test run by `--test`, a waiver, `--exclude`); it changes no verdict, exit code or receipt.
- **Time:** about one build plus one test run per line. On an iPhone app with 9 edited lines: a cold build took a median of 6.5 seconds over four builds, and each of 24 judged lines took 5.0 to 12.2 seconds. The first run builds from scratch, may download packages and prints `first run here:`; after the first line, `estimate:` gives the time left.
- **Will not build in the copy:** a project that uses files outside the repository (`../SharedKit`). A custom build location (`SYMROOT`, `OBJROOT`, `CONFIGURATION_BUILD_DIR`) stops the run.
- **Crashes:** a break that crashes your app leaves a crash report in `~/Library/Logs/DiagnosticReports`.
- **Your own commands:** your own `--check` needs your own `--test`. In them, `{simulator}` becomes the id of RedMachine's simulator and `{cache}` its folder for this repository. An `xcodebuild` without `-derivedDataPath` uses Xcode's build folder and your simulators; a note says so. `--list` runs no Xcode tool and shows `<scheme>` and `<destination>`.

## 4. Which lines are judged

### 4.1 The change

- A change is everything that differs from the common commit of the base and `HEAD`: commits on the branch, staged and unstaged edits, and untracked files git does not ignore.
- Only added and edited lines are judged. Removed lines are not; a note says how many lines were only removed.
- The default base is the first that exists of `main`, `master`, `origin/main`, `origin/master`, `origin/HEAD`. A remote one that is `HEAD`'s own commit is skipped. With none, the run stops (exit 2); it never falls back to `HEAD`.
- The upstream of `main` (the branch it tracks, else `origin/main`) is the base in two cases: you are on `main` with unpushed commits (those are judged), or your branch already holds newer upstream commits while local `main` was never pulled.
- `--base ROOT` compares with the empty tree: `base: ROOT (common commit 4b825dc642cb)`.
- Your git diff settings (`diff.relative`, `diff.renames` and the like) do not change what is judged. Every git diff runs against a private copy of the index, so your index is never rewritten. A run first removes copies older than an hour (`redmachine-index-*` in the temp folder) that a cut-off run left.

### 4.2 Lines counted although not edited

| Case | What is counted | How it is broken |
| --- | --- | --- |
| became code | unchanged lines the old file left out and the new file counts: you deleted a `#[cfg(test)]` or a comment opener around code, moved code out of a test module, or removed or changed a `#[cfg(...)]` line | as usual |
| became reachable | unchanged lines after a removed `return`, `break`, `continue`, `raise`, `throw` or exit call (`panic!(`, `sys.exit(`, `process.exit(`, `fatalError(`, `os.Exit(`), in the same block; the body of an always-false block (`if false {`, `while 0 {`, `#if 0`, `if !true {`, `if false && x > 1 {`) when its opener and closing line were both removed, at any indent, or its opener was changed (`{`, `if true {`, `loop {`, another condition); the body of a dead `else if false {` or `elif False:` made a plain `else` or a live `else if`; `if false { A } else { B }` rewritten to keep A; a Python body after a removed `else:`, `elif` or `finally:` | as usual |
| moved file | a file moved from an unjudged path (a test folder) onto a judged one: every line | as usual |
| Go file switched on | a Go file whose rename or removed build line now builds it: every line | as usual |
| repointed import | an import that replaces an older import in the same hunk (`from fast import tax` to `from legacy import tax`) | `put back as it was` only |
| new enumeration member | a member inserted before an existing one in an enumeration numbered by place (Rust, Swift with an integer raw type) | `delete` only |

The `changed lines:` line says how many of each, `--list` marks each line, and the line records carry a `why` field. Paths that a removed guard changes in code that already ran are not judged. An always-false condition is `false`, `0`, `!true` (Python `False`, `not True`, `not 1`), alone, in parentheses, or as one AND conjunct; any `||` or other expression makes the condition live. Nor is a block always false when its opener or closer shares a line with other code, or a comment follows the opener.

### 4.3 Files set aside

A changed file passes these filters in order; the first that applies sets it aside. The `changed files not judged:` line counts the files set aside by filters 1 to 3; filter 4 is counted on its own line, `changed test files, run but not themselves judged:`.

1. Not under `--only`, or under `--exclude`: `outside --only/--exclude`.
2. Not source of a judged kind (a `.md`, a `.json`, `.pyi`, `.pyx`, `Package.swift`): `not source for this profile` when tracked; an untracked file that is not source is not counted.
3. No project above it: named, and it fails the run ([3.2](#32-how-a-project-is-found)).
4. On a test path: counted as a test file. It is not broken and not judged, and the test command still runs it.

| Language | A test file is |
| --- | --- |
| Rust | under a `tests` or `benches` folder; `build.rs` outside `src`; named `tests.rs`, `*_test.rs` or `*_tests.rs` |
| Python | under a `tests` or `test` folder; named `test_*`, `*_test.py`, `*_test.pyw`, `conftest.py`, `tests.py`, `tests.pyw` or `setup.py` |
| Node | under `__tests__`, `__mocks__`, `tests`, `test`, `spec` or `e2e`; a name holding `.test.`, `.spec.`, `-test.`, `_test.`, `-spec.` or `_spec.`, or starting `test.`, `test-` or `test_` |
| Swift package | under a `Tests` folder; named `*Tests.swift` or `*Test.swift` |
| Swift in Xcode | [3.7](#37-xcode-projects) |
| Go | named `*_test.go` |
| Your own commands | none; use `--exclude` |

A changed source file that is read-only, or binary (not UTF-8 and holding NUL bytes), stops the run (exit 2), as does a changed file git cannot read. One not in the working tree (outside a sparse-checkout cone, missing, or new and unreadable) is named and fails the run, or stops it (exit 2) when nothing else is judged; a file the change deletes is not one.

### 4.4 Lines left out

A left-out line is counted in `changed lines:` by reason, and is not broken, listed or proven.

| Reason | What | Languages |
| --- | --- | --- |
| `blank` | empty or whitespace only | all |
| `comment` | only a comment, or inside a block comment | all |
| `docstring` | a string that is a statement by itself, and its lines | Python |
| `implicit return` | a final `return` or `return None` of a function | Python |
| `import` | `use`, `import`, `from ... import`, `export ... from`, through the last line of a multi-line import. Counted: a side-effect import (`import "./x"`, a bare `require("./x");`, Go `_ "pkg"`; broken by `delete` only), code after a `;`, and a repointed import ([4.2](#42-lines-counted-although-not-edited)) | Rust, Python, Node, Swift, Go |
| `package` | `package ...` | Go |
| `module declaration` | `mod name;` | Rust |
| `attribute` | only attributes (`#[...]`, `@Name(...)`), through the last line of a wrapped one | Rust, Swift |
| `directive` | Swift `#else`, `#endif` and other `#` lines that are not conditions; Go pragmas (`//go:generate`, `//go:noinline`, `//line`); a `#!` first line | Swift, Go, Node |
| `structural` | no letters, digits or non-ASCII (`}`, `);`); only `else`, `try`, `finally` or `do` with brackets (`} else {`); a JSX line that only closes a tag | all |
| `declaration` | a header of a function, struct, class, enum, trait, impl, protocol, extension or module, every line of a header over several lines, and the fields inside a Rust or Go struct or enum; Swift properties with no value, protocol requirements and plain enum cases; Rust `type` aliases | Rust, Swift, Go, Python |
| `type declaration` | an `interface` or `type` declaration, every line | TypeScript |
| `view layout` | a SwiftUI layout or display line with literal arguments only: a view from SwiftUI's own set (`Text("Hello")`, `Spacer()`, `Image(systemName: "star")`, `VStack(alignment: .leading) {`, `Section("Title") {`, `List {`) or a modifier such as `.padding(.horizontal, 6)`, `.font(.title)`, `.bold()`, inside the braces of a declaration whose result is `some View`, `some Scene`, `some ToolbarContent` or `some Commands`, or of `@ViewBuilder`. Style constants such as `Color.red` and `Font.title` count as literal arguments. Counted: a line that builds your own view (`HomeView()`), a `NavigationLink`, `Toggle`, `Picker`, `TextField` or `ForEach`, names a value (`Text(title)`, `Image(Assets.logo)`, `Route.settings`), has a `value:`, `destination:`, `tag`, `id:`, `selection:` or `for:` argument, computes (an operator, a call, string interpolation), is a statement (`let`, `return`, an assignment, `if`, `switch`, `case`), or sits inside an action closure (`Button("Go") { ... }`, `.onAppear`, `.task`, `.onChange`). Being inside a type that conforms to `View` is not enough | Swift with `import SwiftUI` |
| `preview` | inside a `#Preview` block or a `PreviewProvider` type | Swift |
| `test code` | a Rust item under `#[cfg(test)]` (or a test-only `cfg(all(...))`/`cfg(any(...))`), and a file that starts with `#![cfg(test)]` | Rust |
| `symlink` | a symbolic link | all |
| `not UTF-8 text` | a file that is not UTF-8 | all |

- A header is counted when it holds something breakable: a default argument (`def f(x=5):`), a discriminant (`enum Level { Low = 1 }`), an argument in a Python class header (`class B(Base, flag=True):`). JavaScript and TypeScript function headers are always counted.
- Swift `#if` and `#elseif` lines and Go `//go:build` lines are counted, and broken by negating them.
- When the scanner is not sure, it counts the line.

## 5. How a line is broken

### 5.1 The breaks

A break changes only code: not strings, comments, character literals, regular expressions or JSX text. Each kind changes only the first place on the line where it fits.

| Break | The change | Where |
| --- | --- | --- |
| `== to !=`, `!= to ==` | the first `==` or `!=` (`===` and `!==` too) | all |
| `true to false`, `false to true` | a whole word; `True` and `False` in Python | all |
| `< to >`, `> to <` | needs a space on each side, so `Vec<u8>` is left alone | all |
| `<= to <`, `>= to >` | not inside `<<=` or `>>=` | all |
| `&& to \|\|`, `\|\| to &&` | | all |
| `and to or`, `or to and` | | Python |
| `+ to -`, `- to +` | not in `++`, `+=`, `--`, `-=` or `->`; with `--hash-comments`, not a sign that starts a word (`-x`, `--flag`) | all |
| `* to /`, `/ to *` | needs a space on each side | all |
| `+= to -=`, `-= to +=` | | all |
| `is not None to is None`, `is None to is not None` | | Python |
| `condition negated` | the condition of an `if`, `else if`, `elif` or `while`; a Rust match-arm guard; a Swift `#if` or `#elseif`; a Go `//go:build` line | all |
| `Some to None` | `Some(x)` becomes `None` | Rust |
| `delete` | the line is emptied, so later lines keep their numbers; attribute lines directly above go with it; a line inside a multi-line string, or after a backslash, is removed | all |
| `statement replaced with pass` | a statement that is not a block header | Python |
| `number bumped` | the first whole number gets 1 added (not `0xff`, `10u8`, or a Rust array length) | all |
| `string changed` | the first string that opens and closes on the line becomes `zz-mutated-zz`; in Rust, each line of a multi-line string too | all |
| `match arm emptied` | the arm's value becomes `Default::default()` | Rust |
| `loop skipped` | a `for` loop runs zero times (last resort) | Rust, Python, Node, Go, Swift |
| `value replaced with default` | a tail expression, a `return` value, a `let` value, the inside of `Ok`, `Some` or `Err`, or a struct field value becomes the type's default (last resort) | Rust |
| `return value dropped` | `return x;` becomes `return;`, or `return undefined as never;` where TypeScript needs it (last resort) | Node |
| `condition forced` | an `if let`, `while let` or `guard let` can never be true (last resort) | Rust, Swift |
| `put back as it was` | a repointed import gets its old line back; its only break | Rust, Python, Node, Swift, Go |

### 5.2 Order, and strict mode

- The breaks are tried in the order of the table. The first one that passes the check decides the line. A break that fails the check is put back and its tests are not run. If none passes the check, the line is UNVIABLE.
- The last-resort breaks are tried only when nothing above them built.
- With `--strict`, every break that builds runs the tests. One that leaves them green makes the line SURVIVED and is named. When all turn red, the line is KILLED, shown as `[every break]` when more than one built. Each break that builds costs a test run.

### 5.3 Lines no build reads

A line that a platform or feature condition leaves out of this machine's build (`#[cfg(target_os = "linux")]` on a Mac, `#if os(iOS)` in a Mac build, a `_windows.go` file) passes every test whatever is done to it. In Rust, Swift and Go, with the built-in check, a line that would be SURVIVED is first replaced by text that cannot compile, and only the check runs. If the check still passes, the line is SURVIVED with a reason line:

```
SURVIVED  src/lib.rs:7  [- to +]  every test passed with this line broken
          line: 8080 - 1
          broken as: 8080 + 1
          this line is not built by this machine's test build (a platform, feature or build-mode condition leaves it out), so nothing here can prove it; if another platform's job proves it, waive it with that reason
```

- A `#[cfg(not(test))]` line in a crate with only a binary says `this line is built by cargo build but not by the test build (cfg(not(test)) or similar), so no test can reach it`. In a library crate it is SURVIVED with no reason line: cargo builds the plain library for doc tests, so the test build reads it.
- In an Xcode project, a file of a Swift package beside the project that the scheme does not build is SURVIVED with a reason that names the command to judge the package alone: `redmachine --only Sources --check 'swift build --build-tests' --test 'swift test'`.
- Run RedMachine where the line is built, or [waive](#9-waivers) it with that reason.
- This costs one more check for each line that would survive. It is not done with your own `--check`.

## 6. Output, verdicts and results

### 6.1 Output lines

| Line | Meaning |
| --- | --- |
| `base:` | the base, and the first 12 characters of the common commit |
| `profile:` or `projects:` | the one project's kind, `custom`, or `none needed`; or the number of projects, each with a `project DIR (KIND)` block |
| `check:`, `test:` | the commands as they will run, cut to 100 characters with `…` |
| `copy:`, `waiting:`, `first run here:`, `estimate:` | Swift and Xcode runs ([3.6](#36-swift-the-copy), [3.7](#37-xcode-projects)) |
| `changed lines:` | lines counted, lines left out by reason ([4.4](#44-lines-left-out)) |
| `changed test files, run but not themselves judged:` | the number of changed test files; they are not broken, but the test command runs them; shown only when there are some |
| `changed files not judged:` | other files set aside, by filter ([4.3](#43-files-set-aside)); shown only when there are some |
| `note:` | something that did not stop the run |
| `problem:` | with `--list`: a file the run would refuse or not judge |
| `baseline:` | one run with nothing broken, its time, and the time limit for a broken line; when it fails, `RED`, its `error:` lines and last lines, `the full output is in PATH`, and a hint when it found no tests |
| verdict lines | the verdict, `FILE:LINE`, the break in brackets; under SURVIVED and UNVIABLE, `line:` with the original text of the line (trimmed, cut to 160 characters with `...`, control characters shown as `?`); under SURVIVED, `broken as:` with the text the tests passed with, or `(the line emptied)` |
| summary | the counts, the survivors with their `waive key:`, the unviable lines and the lines not tried, at most 25 per list |
| `restore:` | whether your files are as they started ([8.1](#81-restore)) |
| `receipt:`, `proof id:` | the receipt written, and its proof id ([10](#10-the-receipt-and-verify)) |
| `RESULT:` | PASS or FAIL for the whole change ([6.3](#63-result-lines)) |

### 6.2 Verdicts

| Verdict | Meaning | Fails the run |
| --- | --- | --- |
| `KILLED` | a test went red with the line broken: proven. `[timeout]`: the tests ran past their limit and the unbroken tests then passed. `[memory limit]`: two runs with the line broken were stopped for memory. `[before any test ran]`: an Xcode app failed before any test, twice, with the line broken; no test checked it ([3.7](#37-xcode-projects)). | no |
| `SURVIVED` | every test passed with the line broken, or no build here reads the line ([5.3](#53-lines-no-build-reads)), or under `--strict` only a crash before any test ran stopped it | yes |
| `UNVIABLE` | no break of the line still builds; listed, not proven, a person decides | no |
| `NOT_TRIED` | no verdict: left over by `--max`, a check past its time or memory limit, tests red before any break, a kill `--confirm-kills` could not confirm, or, in an Xcode run, a build or test run that failed for another reason, or low memory ([3.7](#37-xcode-projects)) | yes |
| `WAIVED` | a person decided the line needs no proof and wrote down why ([9](#9-waivers)) | no |

"Red" is any non-zero exit or a signal.

### 6.3 RESULT lines

- `PASS. Every counted line is proven by a forced fail.`
- `PASS. N lines proven by a forced fail. M could not be broken and still build, so are not proven (listed above).` Waived lines add `K waived by a person`.
- `PASS. N lines proven by a failing test, C by a crash before any test ran (listed above).`
- `PASS, with nothing proven.` Every counted line was UNVIABLE or WAIVED.
- `PASS. No changed line needs proving against BASE.` After `nothing to prove:`. If you expected lines, read the `base:` line.
- `FAIL. N changed lines are not proven by any test.` Lines survived. Lines not tried and changed files not judged add their own parts, joined by `; `.
- `FAIL. The baseline was red, so nothing could be proven.`
- `FAIL. The tests fail with nothing broken after the run, so the KILLED verdicts cannot be trusted: ...` See [7.4](#74-the-check-after-the-run-and-confirm-kills).
- `FAIL. The working tree could not be verified as restored.` Exit 3 ([8.1](#81-restore)).
- `FAIL. A judged file in the copy could not be verified as put back: ...` Exit 1 ([3.6](#36-swift-the-copy)).

With `--expect`, a different id adds `EXPECTED proof id ID, got a different one: ...` and exit 1.

### 6.4 Exit codes

| Code | Meaning |
| --- | --- |
| 0 | PASS. Also `--list`, and a change with nothing to prove. |
| 1 | FAIL: a line SURVIVED or NOT_TRIED, the baseline was red, the tests were red after the run, a changed source file could not be judged, the copy was rewritten, or `--expect` did not match. |
| 2 | RedMachine could not run, or stopped ([11](#11-what-stops-a-run)). Any broken line is put back. No receipt. |
| 3 | Your working tree needs attention: run `git status` and `git diff` first ([8](#8-your-files)). It replaces 0 or 1, and 2 when git took in a broken line. |
| 130 | Interrupted, or in an Xcode run the Mac ran short of memory during a build or test run. The broken line was put back. No receipt. |

`redmachine verify` exits 0 for a receipt that records a pass, 1 for one that records a fail or does not match `--expect`, 2 for one that is not valid. With `--ran TEST` it exits 0 when the test ran on the unbroken code, 1 when it did not, and 4 when it cannot tell: the receipt records no test names (your own `--test` command), or the name given is the end of several test names.

## 7. Time, memory and processes

### 7.1 Time limits

- The baseline has no time limit; it has the memory limit. With the built-in Xcode commands, each baseline command may take up to an hour.
- The test limit for a broken line is `--timeout`, or 3 x the whole seconds of the baseline test run + 3, at least 60. The `baseline:` line prints it.
- The check limit is 3 x the longest baseline check time + 3, and never less than the test limit. The first time a check runs past it in a project, RedMachine measures a rebuild with nothing broken and tries that check once more, then raises the limit if needed; notes say so.
- A line whose check runs out of time, with no other break that builds, is NOT_TRIED.
- Tests past the limit are stopped, the file is put back, and the unbroken check and tests of every project run again. If they pass, the line is `KILLED [timeout]`. If not, something was left behind (a process, a lock, a port) and the run stops (exit 2).
- In an Xcode run, the line is then broken, built and tested once more. It is `KILLED [timeout]` only if that run also goes past the limit; otherwise that run decides the line.
- If that re-run also runs out of time, a last run gets a longer limit; if it passes, the run stops: the limit is too low.
- Before each break and each put-back, RedMachine waits for the next whole second, so tools that compare file times in whole seconds (`make`, Python's bytecode cache) always rebuild.

### 7.2 Memory limit

- Every command RedMachine runs, the baseline included, is stopped once its processes together hold more than the limit.
- The limit is `--max-memory MB`, by default a quarter of the machine's memory, at least 1024 MB and at most 8192 MB. `--max-memory 0` turns it off.
- The limit must be at least twice the peak of the unbroken check and tests, or the run stops before anything is broken (exit 2) and names the figure to give; so does an unbroken run over the limit.
- Tests of a broken line stopped for memory: the unbroken commands run again, then the tests with the line broken once more. A second stop makes the line `KILLED [memory limit]`. Otherwise that second run decides the line.
- A check of a broken line stopped for memory skips that break. With no other break that builds, the line is NOT_TRIED.
- Memory is read with `ps` (`/bin/ps`, `/usr/bin/ps`, or on the `PATH`). Every judging run needs a working `ps`, also with `--max-memory 0`: without one the run stops before any line is broken (exit 2) and names the `procps` package on Debian and Ubuntu. `--list` does not need it.
- In an Xcode run, the app and its test runner inside RedMachine's simulator, and programs that run from its cache folder (a Mac app it built), count as the command's processes. A test bundle with no host app, inside the simulator, is not weighed.
- In an Xcode run, RedMachine also watches the Mac's free memory. Starting the simulator needs the memory limit plus 1 GB plus about 4 GB free; each line needs the limit plus 1 GB. Swap use that has grown since the run started counts as short of memory. Short of memory before the first line: the run stops (exit 2). Short of memory later: the lines left are NOT_TRIED (`low memory`). Under 1 GB free (or 1% of memory, if more) during a build or test run: the run stops with no verdict and no receipt (exit 130). With `--max-memory 0`, a quarter of the Mac's memory stands in for the limit.

### 7.3 Processes

- Each command runs in its own process group. Every descendant RedMachine sees is remembered, also one that moves to a group or session of its own. When a command ends, for any reason, they are all stopped. RedMachine never stops itself, its parents or process 1.
- What can escape: a child that left the group and whose parent exited between two looks, a process started after the last look, and a process the baseline started that left its group.
- After a `kill -9` of RedMachine, its command keeps running until the next run in that repository, which stops the recorded processes and prints `recovered: N processes of a cut-off run were still running and have been stopped`.

### 7.4 The check after the run, and confirm-kills

- When any line was KILLED, the unbroken tests of each project with a KILLED line run once more at the end. Red fails the run: a broken run left something behind (a file, a folder, a lock), or a test fails at random. A stop for time or memory runs once more with three times the limit; a second stop fails the run too.
- `--confirm-kills` runs the unbroken tests after every KILLED line. If they are red, the line is NOT_TRIED (`kill not confirmed`) and the run fails.
- Both catch only a failure that repeats: a flaky test that fails once, while a line is broken, still gives a false KILLED.

## 8. Your files

### 8.1 Restore

Without a copy ([3.6](#36-swift-the-copy)), RedMachine breaks your files in place and puts them back.

- Before each break, the file's bytes go to a journal in `.git/redmachine/journal/`. After the check and the tests, the bytes are written back and the journal is cleared. Only regular files are written.
- If a file changes while RedMachine works on it, the other content is saved under `.git/redmachine/saved/`, the file is put back as RedMachine read it, and the run stops (exit 3). Do not edit files during a run.
- At the end, the working tree is compared with how it started. The comparison covers the index, every tracked file that differs from `HEAD` (content and mode), every judged file, and every untracked source file that was there at the start. It does not cover ignored files, files created during the run (build output), the inside of a submodule, modification times, or commits, branches and stashes. An untracked file that is not source and that the tests changed gets a note only.
- A difference prints `restore: MISMATCH`, names what differs, fails the run and exits 3.
- If a test command runs `git add` or `git commit` while a line is broken, git holds the broken line. RedMachine looks in the index and the last 100 new commits, names what it finds (`the index holds FILE with line N as redmachine broke it`, `commit SHA holds ...`) and exits 3.

### 8.2 Interrupts and recovery

- `SIGINT`, `SIGTERM`, `SIGHUP`, `SIGQUIT`, `SIGUSR1`, `SIGUSR2`, `SIGALRM` and a few more stop the command, put the file back and exit 130 with no receipt: `interrupted. Every file is as it was; no receipt was written.` A conflict, or a broken line git took in, makes it exit 3.
- After `kill -9`, the file stays broken and the journal stays. The next run in the repository, `--list` included, puts it back first: `recovered: an earlier run was cut off with a line broken; FILE has been put back as it was`.
- If the file was edited since, it is left alone, its earlier content is saved, and every run stops (exit 3) until the file holds exactly that saved content again, or you delete `.git/redmachine/journal` once the file is right.
- After a cut-off copy run, the next run deletes the leftover copy and shuts down a simulator it left running, and says so in one `recovered:` line.

## 9. Waivers

A waiver says a person decided a SURVIVED line needs no test.

- Put one line per waiver in `.redmachine-waive` at the repository top: `FILE KEY the reason, in your own words`. Lines starting with `#` are comments.
- Copy `FILE` and `KEY` from the summary's `waive key:` text. The key is 12 characters made from the file name, the line, and the nearest non-blank lines above and below, so a waiver follows the line when other lines move and stops matching when the line or its neighbours change.
- A waived line is not broken, does not fail the run, does not use up `--max`, and is recorded in the receipt with its reason. Waivers change the proof id.
- A key that matches more than one line waives none of them. A key that matches no line gets a note.
- A waiver already in `.redmachine-waive` at the base does not cover a line changed since: change the waiver's reason to renew it.
- A line with no file, key or reason, the same key twice, or a file that cannot be read stops the run (exit 2).

```
# FILE KEY reason
tally.py 0624f9a86ca3 the log is kept for a later release; nothing reads it yet
```

```
WAIVED    tally.py:9  the log is kept for a later release; nothing reads it yet
```

## 10. The receipt and verify

### 10.1 Records

Every run that reaches its summary writes the receipt. `--list`, an interrupted run and a run that stops write none. An earlier receipt at the path is removed when a run starts; a path that holds anything else stops the run (exit 2). A receipt that git tracks gets a note to add it to `.gitignore`.

The receipt is one JSON record per line: a `header`, one `baseline` per project, one `line` per counted line, one `failed` per killed line that has test names, a `result`, then a `meta`. Every kind but `meta` ends with `prev`, the SHA-256 of the record before it (64 zeros for the header). The proof id is the SHA-256 of the `result` record.

| Record | Fields |
| --- | --- |
| `header` | `tool`, `version`, `base` (the common commit; the empty tree for `ROOT`), `head`, `worktree` (a fingerprint of changed and untracked source files), `profile`, `check`, `test`, `exts`, `only`, `exclude`, `max`, `strict`, `hash_comments`, `timeout`, `max_memory`, `confirm_kills` |
| `baseline` | `project`, `runner`, `names` (`listed`, `partial` or `unknown`), `source` (`record` or `printed`), `reason` (when not `listed`), and the test names `passed`, `failed`, `skipped` |
| `line` | `verdict`, `how` (the break, or `timeout`, `memory limit`, `every break`, `before any test ran`, `check timeout`, `check memory limit`, `kill not confirmed`, `waived`; Xcode NOT_TRIED: `build step`, `build failed`, `built then not`, `test run stopped`, `low memory`), `line`, `text_sha256` (of the line's text), `file`, `note` (a waiver's reason, or a reason line), and `why` for a line counted although not edited |
| `failed` | `file`, `line`, `failed` (the tests that failed with that line broken), `source` (`record` or `printed`), and `stops_at` when the runner stops at its first failure |
| `result` | `verdict`, `counted`, `killed`, `survived`, `unviable`, `not_tried`, `waived`, `left_out`, `removed_lines`, `test_files`, `other_files`, `filtered_files`, `restored`, `left_behind`; when above 0, `became_code`, `replaced_imports`, `shifted_members`, `crash_only`, and `no_project_files` (changed source files not judged) |
| `meta` | `proof_id`, `finished_unix`, `seconds`, `timeout_seconds`; outside the chain |

The receipt holds no source text: only paths, line numbers, hashes, commit ids, commands, options and waiver reasons.

### 10.2 The proof id

- It changes with the RedMachine version, `HEAD`, the common commit, the changed and untracked source files, the profile, commands and options, each line's file, number, text hash, verdict, break and note, the totals and the restore.
- It does not depend on the time taken, the receipt path or untracked files that are not source.
- The same commit, files, commands, options and test results on the same kind of machine give the same id. `--expect ID` makes a run fail unless it gives ID.
- It is not a signature. Anyone can write a receipt with a consistent chain. Trust an id only when you produced it, your CI printed it, or a run on the same commit with `--expect ID` reproduces it.

### 10.3 Verify

`redmachine verify RECEIPT` checks that every record links to the one before, that the first is a header and the last a result, that the counts equal the line records, and that PASS or FAIL agrees with the counts and the restore. It needs no repository and runs nothing.

```
receipt: the chain and the counts are consistent
note: anyone can rebuild a consistent chain. To confirm this run, run redmachine on the same commit with --expect 6a82a286852415465e74c214d51410919f8a85c01136e4992bac91aa4976d2f1
recorded: base 9a6e18a68138 head 1b48c33897d5 profile rust
recorded test: cargo test --workspace --quiet
counted 9: killed 8, survived 0, unviable 1, not tried 0
tree restored: yes
proof id: 6a82a286852415465e74c214d51410919f8a85c01136e4992bac91aa4976d2f1
recorded result: PASS
```

- Waived lines are listed with their reasons; `scope:` names `--only` and `--exclude`; a `note:` names `--confirm-kills`, lines counted although not edited, and lines killed only by a crash.
- A changed record gives `receipt: NOT VALID. line N: the chain is broken here (this line or the one before it was changed)`, exit 2.
- `verify RECEIPT --expect ID` compares the id without running anything: `EXPECTED proof id ID: this receipt is a different one`, exit 1.
- `verify RECEIPT --ran TEST` says whether the test named TEST ran on the unbroken code, its result there, and the broken lines it was seen to fail for. TEST is a whole test name, or the end of one name when that fits exactly one test. Exit 0: it ran. Exit 1: it did not. Exit 4: the receipt cannot tell (no names recorded, or the name fits several tests). A receipt written by your own `--test` command records no names ([10.4](#104-test-names)).

### 10.4 Test names

The `baseline` record names the tests that ran with nothing broken, and each `failed` record names the tests that failed with that line broken. Each says in `source` where the names were read from, and `verify` says it.

| Runner | `source` | What the names are read from |
| --- | --- | --- |
| `cargo test`, `pytest`, `node --test`, `go test`, `python3 -m unittest`, `swift test` | `printed` | what the runner printed |
| `xcodebuild` | `record` when every name comes from the result bundle (the tests that failed), otherwise `printed` | the result bundle, and what the runner printed |
| your own `--test` command | | no names |

- A `failed` record is `record` only when every name in it came from the result bundle. If any name in it was printed, it is `printed`.
- A `printed` name can be imitated by a test or by the code under test, because both can print the same text. `verify` prints `note: test names are read from the runner's printed output. A test that sets out to deceive can imitate them. The verdicts do not depend on them.` when any names are `printed`, and `verify --ran` prints it when its answer rests on them. Neither changes an exit code.
- The verdicts never depend on the names. They come from the exit codes of the runs.
- For `cargo test` (the built-in command) the names are checked after the baseline against `cargo test --workspace -- --list`: a printed name that no test program lists is left out, with a note. This stops a name invented from nothing. It does not show that a listed test ran or passed. If the listing fails or takes too long, the names stay as printed.

## 11. What stops a run

Exit 2 unless the table says 3. Any broken line is put back, and no receipt is written.

| Message starts | Cause | What to do |
| --- | --- | --- |
| `this machine's id cannot be read` | a machine licence, or `activate`, and no `/etc/machine-id` (a container often has none) | run RedMachine on the machine itself, or give the container a stable `/etc/machine-id`; a new random id each time would use up an activation |
| `this copy has no license, so it can list but not judge` (or `its license signature does not match`, `is not for this machine`, `is not a license`, and the like) | no valid licence for this machine ([13](#13-licence)) | `redmachine activate YOUR-KEY`; the message gives the address to buy a key |
| `unknown option` | a misspelt option | see `redmachine --help` |
| `not inside a git repository` | no repository | `git init`, commit once, run after the next change |
| `no main or master branch to compare against` | no default base | pass `--base REF` |
| `cannot find a common commit between` | a shallow clone, or a base not fetched or misspelt | fetch the full history; name the base |
| `--base X is not a commit git knows` | a misspelt base, or one before the first commit | fix it, or pass `--base ROOT` |
| `--only X matches no file of this repository` | a wrong prefix | give a path from the repository top |
| `another redmachine run is active in this repository` | a second run in one repository | wait, or use a separate clone |
| `cannot tell what kind of project this is` | no project file and no Python source | pass `--profile`, or `--ext`, `--test` and `--check` |
| `no project file above FILE` | nothing in the change has a project | `--exclude` it, or `--profile` with `--test` and `--check` |
| `N changed files are source of a kind this version has no built-in support for` | the change is only unsupported source | `--exclude`, or a run with `--ext` ([3.4](#34-source-with-no-built-in-support)) |
| `N changed files are not in this working tree` | the change is only files outside the checkout ([4.3](#43-files-set-aside)) | check them out, or `--exclude` them |
| `--test and --check replace a project's own commands` | own commands with several projects | add `--profile KIND` |
| `--ext needs --check`, `--ext needs --test` | `--ext` alone | give both |
| `FILE is read-only` | a changed file you cannot write | `chmod u+w FILE`, or `--exclude FILE` |
| `FILE is a binary file` | a source file with NUL bytes | `--exclude FILE` |
| `FILE cannot be read` | git cannot read a changed file | make it readable, or run where it is |
| `FILE:N differs on disk from what git compares` | a filter, LFS or a text conversion | `--exclude FILE`, or run where the file on disk is what git stores |
| `FILE has the skip-worktree flag set` (or `assume-unchanged`) | git hides the change | clear the flag, or commit |
| `FILE: no way to check this file` | TypeScript with no `tsconfig.json` or `tsc` | add them, or pass `--check` |
| `FILE: not in the TypeScript program of tsconfig.json` | `tsc` never checks the file | add it to `"include"`, or pass `--check` |
| `the tests of P ran no tests with nothing broken` | the built-in tests ran none, in every project left ([3.3](#33-several-projects-in-one-repository)) | add tests, pass `--test`, or `--exclude` the files |
| `--timeout N is not longer than the tests took` | the limit is below the baseline | raise `--timeout`, or leave it out |
| `--max-memory N is less than twice what` (or `is not more than`) | the limit is under twice the baseline peak | give the figure the message names, or `0` |
| `no working ps was found` | no usable `ps` | install one (Debian or Ubuntu: `apt-get install procps`) |
| `after a break of FILE:N ran past its time limit` (or `held more than`) | something a stopped run left behind fails the unbroken tests | clear it; run again |
| `with nothing broken, the tests of P took Ns` | the limit is too low for this machine | raise `--timeout` |
| `.redmachine-waive line N` | a bad waiver line, or a key twice | fix the line ([9](#9-waivers)) |
| `the receipt path PATH holds something that is not a redmachine receipt` | `--receipt` names another file | give another path |
| `redmachine works on a quick copy of your project` | the project is on another disk, or not APFS | move it to the Mac's main disk |
| `cannot copy this repository: its .git is a link` | the copy would share your git folder | run in a checkout whose `.git` is a folder |
| `FILE points at your files by their full path` | a project file names the repository's own path | make it relative, as the message says |
| `Files kept changing while redmachine copied them` | autosave, a formatter or a build | run again in a moment |
| `the folder redmachine keeps its copy in` | `REDMACHINE_CACHE` is inside the repository | set it outside |
| `This is an Xcode project, and Xcode is needed`, `This is an iPhone app, and testing it needs an iPhone simulator` (or `iPad app`, `iPad simulator`, `Apple TV app`, `Apple TV simulator`) | no usable Xcode, or no simulator runtime with a device type for the app | install what it names ([3.7](#37-xcode-projects)) |
| `This Xcode project has no tests`, `This Xcode project has a test target`, `This project has more than one scheme with tests`, `The scheme S has no test target` | no scheme with tests, or several | add a unit-test target or add it to the scheme, or pass `--test SCHEME` |
| `This folder holds more than one Xcode project` (or `workspace`) | two of one kind in one folder | `--ext swift --check CMD --test CMD` ([3.5](#35-your-own-commands)) |
| `This scheme builds for` | an Apple Watch or Vision Pro app, which are coming soon | none in this version |
| `This run has an iPad-only app and an iPhone app` (or one with an Apple TV app) | one run holds apps for two simulators | judge them one at a time |
| `The tests of this Mac app open the app on your screen` | Mac tests with a host app | quit the app, pass `--mac-app-on-screen` |
| `The simulator "redmachine tests (do not use)" is running` (or `"redmachine iPad tests (do not use)"`, `"redmachine Apple TV tests (do not use)"`) | you are using RedMachine's simulator | shut it down (Device > Shut Down) |
| `redmachine could not start its own iPhone simulator` (or `iPad simulator`, `Apple TV simulator`, or `remove the app from`) | the simulator failed | quit Simulator and Device Hub; if it repeats, restart the Mac |
| `This project sends its build output to`, `xcodebuild could not` | a custom build location, or a project Xcode cannot read | fix it in Xcode |
| `With your own --check for an Xcode project` | `--check` without `--test` | give both |
| `Your Mac has about N GB of memory free`, `Your Mac started to swap`, `Your Mac ran low on memory while` (exit 130) | too little free memory | quit apps, or a lower `--max-memory` |
| `the build of P stopped working with nothing broken` (or `the tests of P`) | the unbroken build or tests fail after a break | check signing, packages and the simulator |
| `FILE was changed by something else` | an edit during the run (exit 3) | compare with the saved copy it names |
| `an earlier run was cut off with a line broken in FILE, and the file has been edited since` | a `kill -9`, then an edit (exit 3) | put the file right, then delete `.git/redmachine/journal` |
| `the restore journal in PATH is damaged` | a broken journal (exit 3) | check `git diff`, then delete the folder |

## 12. Limits

Read these before you trust a PASS.

- **A break can do whatever the broken code can do.** If your code deletes files, kills processes, sends messages or spends money, a broken line can do it wrongly. The time and memory limits do not prevent that. Run such projects in CI, a container or a spare account.
- **Flaky tests.** A test that fails for another reason while a line is broken makes the line KILLED; `--confirm-kills` catches only a failure that repeats ([7.4](#74-the-check-after-the-run-and-confirm-kills)).
- **UNVIABLE lines pass.** They are listed and do not fail the run, and are common in Rust, Go and Swift (`x := f(n)`, `let i = n`, `let c = open(p);`, a match arm such as `_ => Size::Small,` when the type has no default). A change made only of them passes with nothing proven.
- **Removed lines are not judged**, except where the removal switches code on ([4.2](#42-lines-counted-although-not-edited)).
- **One break per line**, the first that builds, unless `--strict`. Each kind of break changes only the first place on the line it fits. KILLED does not mean every part of the line is pinned down.
- **A crash counts as noticed.** A deleted line that crashes the program is KILLED, whether or not a test checks its result; so is a break that stops an app starting, counted apart ([3.7](#37-xcode-projects)).
- **A new import that shadows an older one** of the same name is left out as an import.
- **A narrower `--test` can give a false SURVIVED**: a line only the rest of the suite notices survives.
- **A harmless break looks like a missing test.** A break that changes nothing a test can see SURVIVES; a person can waive it.
- **Time.** Each line costs at least about one second, its check and one test run ([7.1](#71-time-limits)). With `--max 60` and a ten-minute suite, the test runs alone can take ten hours. One line at a time; one run per repository.
- **Xcode** ([3.7](#37-xcode-projects)). UI tests prove no line unless the scheme has only UI tests. Not yet measured on a 16 GB Mac.
- **View layout.** Changed view text (`Text("Hello")` to `Text("Goodbye")`, two branches' texts swapped) and style values (`.opacity(0)`, `.frame(width: 0)`) are not judged ([4.4](#44-lines-left-out)).
- **The scanner reads text.** It does not parse the language. A statement over several lines is judged one line at a time, so its lines are often UNVIABLE.
- **Network.** Checking the licence opens no connection. Only `redmachine activate` does ([13](#13-licence)). Your tests, and Xcode's package downloads, may.

## 13. Licence

- **What is sold.** One payment. $79 while Swift and Xcode support is in preview. $99 when it leaves preview. Each purchase comes with two keys. Each key works on one machine at a time. Use both yourself, or give one to someone. To move a key to another machine, free the old one on your Polar order page. You do this yourself; we don't do it for you. All sales are final.
- **What runs without one.** `--version`, `--help`, `--list`, `verify`, `license` and `activate`. Judging does not: the run stops with exit 2, says why, and gives `Get a key at https://redmachine.dev/buy` and `Then run: redmachine activate YOUR-KEY`.
- **`redmachine activate KEY`.** A key is 8 to 128 letters, digits, `-` or `_`. RedMachine runs `curl` once to send the key and a one-way hash of the machine's id to `https://redmachine.dev/api/activate`, checks that the answer is a valid licence for this machine, and saves it. Exit 0 saved, 2 not. A key already active on a machine is refused; free it from your Polar order page, then run `activate` again. Running it again on a machine that already has a licence asks the service again.
- **The licence file.** A signed text file of at most 4096 bytes, bound to this machine (a hash of the macOS hardware UUID, or of `/etc/machine-id`). Checking it reads the file and verifies the signature. It never contacts the network.
- **Where it is looked for, in order.** `REDMACHINE_LICENSE`: the licence text itself (it starts with `redmachine-license`, and `\n` stands for a new line), or else the path of a licence file. Then `$XDG_CONFIG_HOME/redmachine/license`. Then `~/.config/redmachine/license`. The first place that exists is the only one read: if it holds a bad licence, the later places are not tried.
- **`redmachine license`.** Prints `license: valid` or `license: none found` or `license: NOT VALID` with the reason, where it was found, its kind and issue date, and this machine's id hash. Exit 0 when valid, 1 when not.
- **Data.** For RedMachine, XETENET LLC stores zero user data anywhere. The activation service keeps no database and no log. Which machine each key is active on is held by the seller, Polar, against your order.

## Index

| Term | Section |
| --- | --- |
| `--base` | [4.1](#41-the-change) |
| `--check`, `--check true` | [3.5](#35-your-own-commands) |
| `--confirm-kills` | [7.4](#74-the-check-after-the-run-and-confirm-kills) |
| `--exclude` | [4.3](#43-files-set-aside) |
| `--expect` | [10.2](#102-the-proof-id) |
| `--ext` | [3.5](#35-your-own-commands) |
| `--hash-comments` | [3.5](#35-your-own-commands) |
| `--list` | [1](#1-options) |
| `--mac-app-on-screen` | [3.7](#37-xcode-projects) |
| `--max` | [1](#1-options) |
| `--max-memory` | [7.2](#72-memory-limit) |
| `--only` | [4.3](#43-files-set-aside) |
| `--profile` | [3.3](#33-several-projects-in-one-repository) |
| `--ran` | [10.3](#103-verify) |
| `--receipt` | [10.1](#101-records) |
| `--strict` | [5.2](#52-order-and-strict-mode) |
| `--test` | [1](#1-options) |
| `--timeout` | [7.1](#71-time-limits) |
| `activate`, `license`, licence, `REDMACHINE_LICENSE` | [13](#13-licence) |
| `.build/`, `target/`, `node_modules/` (build output) | [3.2](#32-how-a-project-is-found) |
| `.redmachine-waive` | [9](#9-waivers) |
| `.git/redmachine/` (lock, journal, saved, run.log) | [8.1](#81-restore) |
| `.pyw`, Python `#!` scripts | [3.1](#31-built-in-languages) |
| `[before any test ran]`, `crash_only` | [3.7](#37-xcode-projects) |
| `[every break]` | [5.2](#52-order-and-strict-mode) |
| `[memory limit]` | [7.2](#72-memory-limit) |
| `[timeout]` | [7.1](#71-time-limits) |
| always-false block | [4.2](#42-lines-counted-although-not-edited) |
| base, common commit | [4.1](#41-the-change) |
| baseline, `baseline: RED` | [6.1](#61-output-lines) |
| became code, became reachable | [4.2](#42-lines-counted-although-not-edited) |
| breaks | [5.1](#51-the-breaks) |
| `broken as:` | [6.1](#61-output-lines) |
| build folders kept (`swiftpm.noindex`) | [3.6](#36-swift-the-copy) |
| build step or script phase failed | [3.7](#37-xcode-projects) |
| `Cargo.toml`, `go.mod`, `package.json`, `pyproject.toml`, `Package.swift` | [3.2](#32-how-a-project-is-found) |
| `changed files not judged:` | [4.3](#43-files-set-aside) |
| `changed lines:` | [4.4](#44-lines-left-out) |
| check command | [3.1](#31-built-in-languages) |
| copy, `copy:` | [3.6](#36-swift-the-copy) |
| crash reports (`DiagnosticReports`) | [3.7](#37-xcode-projects) |
| `redmachine verify` | [10.3](#103-verify) |
| `redmachine-receipt.jsonl`, `.gitignore` | [10.1](#101-records) |
| `REDMACHINE_CACHE` | [3.6](#36-swift-the-copy) |
| `estimate:`, `first run here:` | [3.7](#37-xcode-projects) |
| exit codes | [6.4](#64-exit-codes) |
| `EXPECTED proof id` | [6.3](#63-result-lines) |
| flaky tests | [12](#12-limits) |
| git index, `redmachine-index-*` | [4.1](#41-the-change) |
| Go workspace (`go.work`) | [3.2](#32-how-a-project-is-found) |
| import, repointed import | [4.2](#42-lines-counted-although-not-edited) |
| interrupt, `kill -9` | [8.2](#82-interrupts-and-recovery) |
| `KILLED` | [6.2](#62-verdicts) |
| left behind | [7.4](#74-the-check-after-the-run-and-confirm-kills) |
| left-out reasons | [4.4](#44-lines-left-out) |
| lines no build reads | [5.3](#53-lines-no-build-reads) |
| `low memory`, free memory (Xcode) | [7.2](#72-memory-limit) |
| `MISMATCH` | [8.1](#81-restore) |
| `NOT_TRIED` | [6.2](#62-verdicts) |
| `note:` | [6.1](#61-output-lines) |
| `nothing to prove:` | [6.3](#63-result-lines) |
| Objective-C, C, C++, Metal; `Pods/`, `Carthage/` | [3.7](#37-xcode-projects) |
| placeholders (`{simulator}`, `{cache}`) | [3.7](#37-xcode-projects) |
| preview (Swift and Xcode support) | [3.1](#31-built-in-languages) |
| `preview`, `view layout` (left out) | [4.4](#44-lines-left-out) |
| `problem:` | [6.1](#61-output-lines) |
| processes | [7.3](#73-processes) |
| project folder | [3.2](#32-how-a-project-is-found) |
| proof id | [10.2](#102-the-proof-id) |
| `ps` | [7.2](#72-memory-limit) |
| pytest, unittest | [3.1](#31-built-in-languages) |
| receipt | [10.1](#101-records) |
| `receipt: NOT VALID` | [10.3](#103-verify) |
| `recovered:` | [8.2](#82-interrupts-and-recovery) |
| `restore:` | [8.1](#81-restore) |
| `RESULT:` | [6.3](#63-result-lines) |
| `ROOT` | [4.1](#41-the-change) |
| scheme | [3.7](#37-xcode-projects) |
| several projects | [3.3](#33-several-projects-in-one-repository) |
| simulator | [3.7](#37-xcode-projects) |
| `source`, `record`, `printed` | [10.4](#104-test-names) |
| sparse checkout | [4.3](#43-files-set-aside) |
| stops (exit 2 and 3 messages) | [11](#11-what-stops-a-run) |
| `SURVIVED` | [6.2](#62-verdicts) |
| Swift Testing, XCTest | [3.7](#37-xcode-projects) |
| test command | [3.1](#31-built-in-languages) |
| test files | [4.3](#43-files-set-aside) |
| test names | [10.4](#104-test-names) |
| tests that ran nothing | [3.3](#33-several-projects-in-one-repository) |
| `tsconfig.json`, `tsc` | [3.1](#31-built-in-languages) |
| UI tests | [3.7](#37-xcode-projects) |
| unsupported source (`.kt`, `.java`, `.cs`, `.PY`) | [3.4](#34-source-with-no-built-in-support) |
| `UNVIABLE` | [6.2](#62-verdicts) |
| upstream of `main` | [4.1](#41-the-change) |
| view code | [3.7](#37-xcode-projects) |
| `waiting:` | [3.6](#36-swift-the-copy) |
| `waive key:` | [9](#9-waivers) |
| `WAIVED` | [6.2](#62-verdicts) |
| Xcode, `.xcodeproj`, `.xcworkspace` | [3.7](#37-xcode-projects) |
