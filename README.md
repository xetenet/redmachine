# RedMachine

RedMachine takes the lines your git change added or edited, breaks each one in turn, and runs your project's check and tests. This is mutation testing on your changed lines. A line counts as proven when a test goes red while it is broken: one test noticed one break. A changed line that no test notices fails the run.

It is one program with no dependencies, for macOS (Apple silicon and Intel), Linux (x86_64 and arm64) and Windows through WSL 2. It needs `git`, `sh` and `ps`, plus what your tests need, and `curl` once, to activate your licence. On Linux the licence is bound to the machine id in `/etc/machine-id`. Built in: Rust, Python, JavaScript and TypeScript, Go, and Swift, including Xcode apps (preview); other languages run with your own commands. Preview means tried on small projects only; its SwiftUI rules read text rather than parse code.

RedMachine runs your tests against code it has broken on purpose. If your code deletes files, kills processes or sends anything, a broken line can do that wrongly. Run such projects in CI, a container or a spare account.

Contents: [Install](#install), [Licence and price](#licence-and-price), [First run](#first-run), [Verdicts](#verdicts), [A SURVIVED line](#a-survived-line), [Receipts](#receipts), [Have your assistant run it](#have-your-assistant-run-it), [Xcode apps](#xcode-apps), [Everyday options](#everyday-options), [Exit codes](#exit-codes), [More](#more).

## Install

Three archives come with a `SHA256SUMS` file: `redmachine-0.1.0-macos-universal.tar.gz` (Xcode projects: Apple silicon only), `redmachine-0.1.0-linux-x86_64.tar.gz` and `redmachine-0.1.0-linux-arm64.tar.gz`. Download them from [the releases page](https://github.com/xetenet/redmachine/releases/latest), or with `curl` as below.

```
curl -LO https://github.com/xetenet/redmachine/releases/download/v0.1.0/redmachine-0.1.0-macos-universal.tar.gz
curl -LO https://github.com/xetenet/redmachine/releases/download/v0.1.0/SHA256SUMS
grep macos-universal SHA256SUMS | shasum -a 256 -c
tar xzf redmachine-0.1.0-macos-universal.tar.gz
mkdir -p ~/.local/bin
cp redmachine-0.1.0-macos-universal/redmachine ~/.local/bin/
export PATH="$HOME/.local/bin:$PATH"
redmachine --version
```

On Linux, the first four lines are these (for arm64, write `linux-arm64` in place of `linux-x86_64`), and the rest are the same with the Linux folder name:

```
curl -LO https://github.com/xetenet/redmachine/releases/download/v0.1.0/redmachine-0.1.0-linux-x86_64.tar.gz
curl -LO https://github.com/xetenet/redmachine/releases/download/v0.1.0/SHA256SUMS
grep linux-x86_64 SHA256SUMS | sha256sum -c
tar xzf redmachine-0.1.0-linux-x86_64.tar.gz
```

On Windows, do the Linux steps inside a WSL 2 shell. The program is at `~/.local/bin/redmachine`, and that full path always works, also for an assistant that starts a new shell for every command. The `export` line lasts for one shell. To keep it, add it to your shell's startup file yourself; an assistant should use the full path and leave that file alone.

## Licence and price

$79 while Swift and Xcode support is in preview. $99 when it leaves preview. One payment, no subscription. All sales are final.

1. Buy a key at [redmachine.dev/buy](https://redmachine.dev/buy).
2. Run `redmachine activate YOUR-KEY` on each machine that will judge changes. One key works on two machines: two activations, flat.
3. Run `redmachine license` to see what it found. It exits 0 for a valid licence and 1 otherwise.

The licence is a small signed file bound to the machine. It is saved in `$XDG_CONFIG_HOME/redmachine/license`, or `~/.config/redmachine/license` when that is not set. Checking it never contacts the network. Activation contacts redmachine.dev once, with `curl`, and sends your key and a one-way hash of the machine's id. RedMachine never sends your code, your tests or your results to any network or server. XETENET LLC stores zero user data anywhere.

A container needs a stable `/etc/machine-id`. The licence is bound to the machine id, a container often has none, and a new random id each time would use up an activation. Without a readable id, `redmachine activate` stops before it contacts the service.

Without a licence, `--version`, `--help`, `--list`, `verify`, `license` and `activate` still run. Judging needs a key: the run stops with exit 2 and points to https://redmachine.dev/buy and `redmachine activate YOUR-KEY`. The `REDMACHINE_LICENSE` variable can hold a licence, or the path of one, and is read before the saved file ([CI](docs/CI.md#licence-in-ci)).

## First run

In a git repository whose tests pass, on a branch made from `main` (or `master`), run:

```
redmachine
```

Commits on the branch, staged and unstaged edits, and new untracked files all count. `redmachine --list` shows what would be judged and changes nothing. Add `redmachine-receipt.jsonl` to `.gitignore`: every run rewrites it. That edit then shows as `changed files not judged: 1 not source for this profile`, which is normal.

Here a Rust library gained `discount` and `label` functions and two tests (path shortened):

```
redmachine 0.1.0: every changed line proven by a forced fail
base: main (common commit cf1c278cee46)   profile: rust
check: cargo test --workspace --no-run --quiet
test:  cargo test --workspace
changed lines: 13 counted, 21 left out (blank 4, declaration 7, structural 10)
changed test files, run but not themselves judged: 1
baseline: green (0.1s). One run of a broken line may take 60s.
KILLED    src/lib.rs:6  [> to <]
KILLED    src/lib.rs:7  [- to +]
KILLED    src/lib.rs:8  [condition negated]
SURVIVED  src/lib.rs:9  [- to +]  every test passed with this line broken
          line: return cut - 5;
          broken as: return cut + 5;
KILLED    src/lib.rs:11  [delete]
KILLED    src/lib.rs:13  [value replaced with default]
UNVIABLE  src/lib.rs:17  no way of breaking this line still builds
          line: let size = Size::from(n);
UNVIABLE  src/lib.rs:18  no way of breaking this line still builds
          line: match size {
KILLED    src/lib.rs:19  [string changed]
SURVIVED  src/lib.rs:20  [string changed]  every test passed with this line broken
          line: Size::Large => "large",
          broken as: Size::Large => "zz-mutated-zz",
KILLED    src/lib.rs:31  [< to >]
UNVIABLE  src/lib.rs:32  no way of breaking this line still builds
          line: Size::Small
UNVIABLE  src/lib.rs:34  no way of breaking this line still builds
          line: Size::Large

================ redmachine summary ================
counted 13: killed 7, survived 2, unviable 4, not tried 0
survived (no test notices when the line is wrong):
  src/lib.rs:9   waive key: src/lib.rs f03852e9cf36
  src/lib.rs:20   waive key: src/lib.rs 97a52e99e3f2
unviable (cannot be broken and still build; never hidden, a person decides):
  src/lib.rs:17
  src/lib.rs:18
  src/lib.rs:32
  src/lib.rs:34
restore: the working tree is verified identical to how it started
receipt: /Users/you/shop/redmachine-receipt.jsonl
proof id: bd1e21d3a1c7789fd8d2ca2be86ab77f35957bdc830a6c1cb0ae3c020881db15
RESULT: FAIL. 2 changed lines are not proven by any test.
```

- `base:` is what the change is compared with. `check:` says whether a broken line still builds; `test:` runs your tests. `baseline:` is one run with nothing broken; if it is red, nothing is judged.
- `restore:` says your files are as they were. The original line is printed under each SURVIVED and UNVIABLE verdict.

## Verdicts

- `KILLED`: a test went red with the line broken. Proven. `KILLED [before any test ran]` (Xcode): the broken line stopped the app starting, so no test checked it; the RESULT counts these apart.
- `SURVIVED`: every test passed with the line broken, or no build on this machine reads the line. Fails the run.
- `UNVIABLE`: no break of the line still builds. Listed and not proven; does not fail the run. A person decides.
- `NOT_TRIED`: no verdict, for a reason the output names (`--max` ran out, a check ran past its limits, a red baseline). Fails the run.
- `WAIVED`: a person decided the line needs no proof and wrote down why. Does not fail the run.

## A SURVIVED line

1. Read its `line:` text, the line as you wrote it, and its `broken as:` line: every test passed with that text.
2. Write a test that fails with the broken text and passes with the real line. Above, `return cut - 5;` became `return cut + 5;` and nothing failed: no test gave a loyal customer a big order.
3. Run `redmachine` again. The line now reads KILLED.

If the line does nothing, delete it. If a person decides no test should notice it, add `FILE KEY reason` to `.redmachine-waive` at the repository top, with FILE and KEY from its `waive key:` text.

## Receipts

Every run writes `redmachine-receipt.jsonl`: one record per counted line, each chained to the one before by a SHA-256 hash. Each line ends as KILLED, SURVIVED, UNVIABLE, NOT_TRIED or WAIVED. The receipt also names the tests that ran with nothing broken and the tests that failed for each KILLED line.

- `redmachine verify RECEIPT` checks that the chain and the counts are consistent and prints the proof id. It is not a signature: confirm a run with `--expect ID`.
- `redmachine verify RECEIPT --ran TEST` says whether the test named TEST ran. Exit 0 means it ran, 1 means it did not, 4 means the receipt cannot tell.
- Each test name is labelled by where it came from. `printed` names are read from the runner's output (cargo, unittest, pytest, node, go). `record` names come from an Xcode result bundle. A test that sets out to deceive can imitate a printed name, so `verify` says so in a note. The verdicts do not depend on the names.
- For cargo, the names are checked against the tests that exist. Your own `--test` command gives no names.

## Have your assistant run it

Paste this into your AI assistant's chat, in the project where it works:

```text
Use redmachine to check that our tests would catch a mistake in the lines we changed.

1. Run `~/.local/bin/redmachine --version`. If that file is not there, try
   `redmachine --version`. Use the one that works for every command below. If
   neither works, stop and tell me.
   Work in a git repository. Run `git status`. Only if it says this is not a git
   repository, run `git init` and commit everything before you change any code,
   so redmachine can see what you changed. Add `redmachine-receipt.jsonl` to
   `.gitignore` if it is not already there.
   If any command says this copy has no license, or that its license is not
   valid, stop and tell me. Buying a key and running `redmachine activate` are
   mine to do.
2. Run `redmachine --list`. Keep its `base:` line, its `test:` line and its
   `changed lines:` line for your final answer. RedMachine judges what changed
   since this branch left main (or master), plus work not committed yet. If it
   counts 0 lines, stop and tell me.
3. Run `redmachine`. Exit code 0 means PASS and 1 means FAIL. For any other exit
   code, show me the message and stop. If it prints `baseline: RED`, our tests
   already fail with nothing broken: fix that first, then run it again.
4. For every line marked SURVIVED, read the `line:` and `broken as:` lines under
   it. `broken as:` is a broken version of that line that every test passed.
   Write a test that fails with that broken version and passes with the real
   line. Do not change the line itself, do not weaken or delete tests, and do
   not add anything to .redmachine-waive.
5. Run `redmachine` again and repeat step 4 until the RESULT line says PASS, or
   until the only lines left are UNVIABLE. If a line is still SURVIVED after two
   tries at a test, stop and tell me which line and what you tried.
6. In your final answer, show me the three lines you kept in step 2 and the last
   RESULT line. List every line marked UNVIABLE with its `line:` text: those are
   not proven, and I decide what to do about them. Leave
   `redmachine-receipt.jsonl` as it is. Do not edit or delete it, and do not
   commit it unless I ask.

Only when it applies:
- The base in step 2 is not what we branched from: add `--base BRANCH` (the
  branch we did branch from) to every command. If you do not know it, stop and
  tell me.
- A run is slow because the workspace is large: give a test command narrowed to
  the code you changed (for example one package or one test file) with --test,
  and a matching --check, so each broken line costs seconds. Give the same
  options to every later run. On a small project, add no options.
- Lines are NOT_TRIED because of the --max limit: run `redmachine --max 200`.
- A note under a SURVIVED line says no test here can reach it, or you cannot
  write such a test: tell me why.
- A SURVIVED line cannot be made to fail any test because it is redundant (other
  code already does its job): do not weaken or delete tests and do not waive it
  yourself. Say which line, why it is redundant, propose deleting it, and stop
  for me to decide.
```

## Xcode apps

You need Xcode, a simulator runtime for your app (Xcode > Settings > Components) and a unit-test target with one passing test. RedMachine finds the project and the scheme with tests. In a copy of your project, it breaks one changed Swift line at a time, builds, and runs your unit tests on its own simulator, "redmachine tests (do not use)". Each line costs about one build plus one test run.

The simulator is RedMachine's own. It is created to match the device model and OS version of the simulator you last used, and it never touches your simulators. With no simulator of yours to match, it uses the newest OS and a default device. A `simulator:` line in the output says which.

RedMachine is most useful on Swift model and logic code: unit tests cannot reach SwiftUI view code, so its judged lines usually survive; move the logic into a model type, waive them, or `--exclude` the view file.

A unit test that renders a SwiftUI view with `ImageRenderer` and compares the picture with a stored PNG can kill view lines. On one small view tried, every counted line whose break builds was KILLED.

Limits: iPhone, iPad and Mac apps are supported; an iPad-only app is tested on RedMachine's own iPad simulator. Apple TV apps are supported, tested on RedMachine's own Apple TV simulator; Apple Watch and Vision Pro apps are coming soon. UI tests are not run for each line. Objective-C, C, C++ and Metal files fail the run until left out with `--exclude`. Details: [3.7](docs/HOW-IT-WORKS.md#37-xcode-projects).

## Everyday options

| Option | What it does |
| --- | --- |
| `--list` | Show what would be judged; change nothing. |
| `--base REF` | Compare with this branch or commit (default `main`, else `master`). |
| `--exclude PATH` | Leave out files under this path. May repeat. |
| `--test CMD` | Run your tests with this command. Python tests in a `tests/` folder with no `__init__.py`: `--test 'python3 -m unittest discover -s tests'`. |
| `--max N` | Run the tests for at most N lines (default 60). |
| `--max-memory MB` | Memory limit (default a quarter of memory). |

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | PASS, or nothing to prove; `--list` too, even with `problem:` lines |
| 1 | FAIL: a line is not proven, the tests were red, a changed file could not be judged, or `--expect` did not match |
| 2 | RedMachine could not run; the message says why |
| 3 | Your working tree needs attention: run `git status` first |
| 130 | Interrupted, or short of memory in an Xcode run; your files are as they were |

## More

- [docs/HOW-IT-WORKS.md](docs/HOW-IT-WORKS.md): the reference, indexed. Read its [limits](docs/HOW-IT-WORKS.md#12-limits) before you trust a PASS.
- [docs/CI.md](docs/CI.md): GitHub Actions, GitLab CI and a git pre-push hook.
- RedMachine is paid software, sold at [redmachine.dev](https://redmachine.dev). Licence and price: [above](#licence-and-price).
