# RedMachine in CI and before a push

Copy-paste setups for CI and a git pre-push hook. The rules behind them are in the [reference](HOW-IT-WORKS.md).

## Contents

1. [Name the base and fetch the history](#1-name-the-base-and-fetch-the-history)
2. [Get the program](#2-get-the-program), and [a licence](#licence-in-ci)
3. [GitHub Actions](#3-github-actions)
4. [GitLab CI](#4-gitlab-ci)
5. [A git pre-push hook](#5-a-git-pre-push-hook)
6. [Exit codes for scripts](#6-exit-codes-for-scripts)
7. [Cautions](#7-cautions)
8. [Platform results](#8-platform-results)

## 1. Name the base and fetch the history

A CI checkout usually has no local `main`, and a shallow one has no common commit with the base. Then the run stops with exit 2 (`no main or master branch to compare against`, or `cannot find a common commit between 'origin/main' and HEAD`).

- Always pass `--base` with the remote branch the change goes into, for example `--base origin/main`.
- Fetch the full history: `fetch-depth: 0` on GitHub, `GIT_DEPTH: "0"` on GitLab.
- Run the job on pull requests or merge requests. On a push to `main`, the base is `main` itself, so nothing committed is judged.
- `base: HEAD` or `nothing to prove` in a log means the job judged nothing.

## 2. Get the program

The jobs below run on Linux, so they need a Linux archive: `redmachine-0.1.0-linux-x86_64.tar.gz`, or `redmachine-0.1.0-linux-arm64.tar.gz` on an ARM runner (change `name` in the job). Each job downloads the archive and `SHA256SUMS` from `https://github.com/xetenet/redmachine/releases/download/v0.1.0/`, and checks the archive against the line for its name in `SHA256SUMS`. If that line is missing, or the sum differs, `sha256sum -c` fails and the job stops. Add your toolchain and dependency steps before the RedMachine step.

### Licence in CI

Judging needs a licence, in CI as anywhere else. Without one the job stops with exit 2 and the message `this copy has no license, so it can list but not judge`.

**Not tried: a hosted runner that is a new machine for each job.** Whether its machine id stays the same from job to job has not been checked, and a container image may have no `/etc/machine-id`. A licence that does not match the machine is refused. The examples below do not claim to work on hosted runners.

- A licence is a signed file bound to one machine. A job can use it only on the machine it was activated on. One key has two activations, and each `redmachine activate` run asks the service again, so do not activate on every job.
- Primary setup: a runner you keep (a self-hosted runner). Run `redmachine activate YOUR-KEY` once on it, as the user the jobs run as. The licence is saved under that user's home (`~/.config/redmachine/license`), and later jobs find it.
- Alternative: pass the licence text from a secret. Store the text of the saved licence file (it starts with `redmachine-license`) as the secret or CI variable `REDMACHINE_LICENSE`, and give it to the judging step. This works only on the machine the licence was activated for.
- RedMachine looks for a licence in this order: `REDMACHINE_LICENSE` (the licence text, or the path of a licence file), then `$XDG_CONFIG_HOME/redmachine/license`, then `~/.config/redmachine/license`. The first place that exists is the only one read. Checking never uses the network.

## 3. GitHub Actions

Use a runner you keep, with `curl`, `tar`, `sha256sum` and `git`, where `redmachine activate YOUR-KEY` was run once (see [licence in CI](#licence-in-ci)). A hosted runner (`ubuntu-latest`) is not tried: do not copy this job onto one and expect a licence to be found.

`.github/workflows/redmachine.yml`:

```yaml
name: redmachine

on:
  pull_request:

jobs:
  redmachine:
    runs-on: self-hosted
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install redmachine
        run: |
          set -e
          base=https://github.com/xetenet/redmachine/releases/download/v0.1.0
          name=redmachine-0.1.0-linux-x86_64.tar.gz
          dir="$RUNNER_TEMP/redmachine"
          mkdir -p "$HOME/.local/bin" "$dir"
          curl -fsSL "$base/$name" -o "$dir/$name"
          curl -fsSL "$base/SHA256SUMS" -o "$dir/SHA256SUMS"
          (cd "$dir" && grep " $name\$" SHA256SUMS | sha256sum -c -)
          tar xzf "$dir/$name" -C "$dir"
          install -m 755 "$dir"/redmachine-*/redmachine "$HOME/.local/bin/redmachine"
          echo "$HOME/.local/bin" >> "$GITHUB_PATH"

      - name: Prove the changed lines
        env:
          BASE: origin/${{ github.base_ref }}
        run: redmachine --base "$BASE" --receipt redmachine-receipt.jsonl

      - name: Keep the receipt
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: redmachine-receipt
          path: redmachine-receipt.jsonl
```

- The job finds the licence that `redmachine activate` saved on the runner.
- Alternative, the licence from a secret: add `REDMACHINE_LICENSE` as a repository secret holding the licence text, and set it on the judging step. It works only on the machine the licence was activated for.

```yaml
      - name: Prove the changed lines
        env:
          BASE: origin/${{ github.base_ref }}
          REDMACHINE_LICENSE: ${{ secrets.REDMACHINE_LICENSE }}
        run: redmachine --base "$BASE" --receipt redmachine-receipt.jsonl
```

- `github.base_ref` is set only on `pull_request` events. Elsewhere `--base origin/` stops the run with exit 2.
- A pull request checkout is a merge into the base, so the common commit is the base tip and only the pull request's own lines are judged.
- A run that stops with exit 2 writes no receipt, so the upload step finds no file.

## 4. GitLab CI

Use a runner you keep that runs jobs in a shell on the machine where `redmachine activate YOUR-KEY` was run once, as the user the jobs run as (see [licence in CI](#licence-in-ci)). Give the job that runner's tag. A hosted runner is not tried. A job in a container starts with a fresh home unless you mount one, so it may not find a saved licence, and an image may have no `/etc/machine-id`; whether a container on your own runner shows the same machine id as the host has not been checked either.

`.gitlab-ci.yml`:

```yaml
redmachine:
  stage: test
  tags:
    - your-runner-tag
  timeout: 1h
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    GIT_DEPTH: "0"
  script:
    - base=https://github.com/xetenet/redmachine/releases/download/v0.1.0
    - name=redmachine-0.1.0-linux-x86_64.tar.gz
    - mkdir -p "$HOME/.local/bin" redmachine-download
    - curl -fsSL "$base/$name" -o "redmachine-download/$name"
    - curl -fsSL "$base/SHA256SUMS" -o redmachine-download/SHA256SUMS
    - (cd redmachine-download && grep " $name\$" SHA256SUMS | sha256sum -c -)
    - tar xzf "redmachine-download/$name" -C redmachine-download
    - install -m 755 redmachine-download/redmachine-*/redmachine "$HOME/.local/bin/redmachine"
    - export PATH="$HOME/.local/bin:$PATH"
    - git fetch --quiet origin "+refs/heads/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME:refs/remotes/origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME"
    - redmachine --base "origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" --receipt redmachine-receipt.jsonl
  artifacts:
    when: always
    paths:
      - redmachine-receipt.jsonl
    expire_in: 90 days
```

- The job finds the licence that `redmachine activate` saved on the runner.
- Alternative, the licence from a CI variable: add `REDMACHINE_LICENSE` under Settings, CI/CD, Variables, holding the licence text. The job reads it before any saved file. It works only on the machine the licence was activated for.
- Your toolchain, `git`, `curl`, `tar`, `sha256sum` and a working `ps` must be on the runner.
- The `git fetch` line makes `origin/<target branch>` exist under that name, whatever the runner's fetch settings.

## 5. A git pre-push hook

Save as `.git/hooks/pre-push` and run `chmod +x .git/hooks/pre-push`. `redmachine` must be on the `PATH` that `git push` uses.

```sh
#!/bin/sh
set -e
if git rev-parse --verify --quiet origin/main >/dev/null; then
  base=origin/main
else
  base=main
fi
receipt="$(git rev-parse --absolute-git-dir)/redmachine-receipt.jsonl"
redmachine --base "$base" --receipt "$receipt"
```

A FAIL stops the push (`error: failed to push some refs`); a PASS lets it through. The receipt goes inside the git folder, out of `git status`.

- It judges the checked-out branch and its uncommitted work, not the commits being pushed. Commit or stash what you do not want judged.
- `origin/main` is as fresh as your last `git fetch`.
- `git push --no-verify` skips the hook.
- It runs your tests about once per changed line. For a large suite, narrow it with `--only` and `--max`, or run it in CI.

## 6. Exit codes for scripts

| Code | Meaning in a pipeline |
| --- | --- |
| 0 | PASS, or nothing to prove. Check the `base:` line if you expected lines. |
| 1 | FAIL: a line is not proven, the tests were red, a changed source file could not be judged, or `--expect` did not match. |
| 2 | Could not run: base, history, options, project or a limit. Nothing was proven and no receipt was written. |
| 3 | The tests changed tracked or source files, or ran `git add` or `git commit` while a line was broken. The log names the file. |
| 130 | Cancelled. Every file was put back. |

All codes are in the reference, [6.4](HOW-IT-WORKS.md#64-exit-codes), with every stop message in [11](HOW-IT-WORKS.md#11-what-stops-a-run).

## 7. Cautions

- **Set a job time limit.** The baseline has no time limit of its own, and each changed line costs a test run. Allow your test time times the lines you expect ([time limits](HOW-IT-WORKS.md#71-time-limits)).
- **Set `--max-memory` under the container's limit.** The default is a quarter of the memory the machine reports, which can be more than a container is allowed. The runner needs a working `ps`; a BusyBox `ps` should be expected to stop the run unless you pass `--max-memory 0` ([memory limit](HOW-IT-WORKS.md#72-memory-limit)).
- **Flaky tests.** A test that fails at random while a line is broken makes the line KILLED. Use `--confirm-kills` where a false PASS costs more than time ([7.4](HOW-IT-WORKS.md#74-the-check-after-the-run-and-confirm-kills)).
- **Use a fresh checkout per job.** After a `kill -9`, the next run in that checkout must put the broken line back first ([recovery](HOW-IT-WORKS.md#82-interrupts-and-recovery)).
- **Xcode projects need a Mac.** On these Linux jobs, a changed Swift file in an Xcode project stops the run (exit 2, `This is an Xcode project, and Xcode is needed to build it`). Judge it on a Mac with Xcode, or leave it out with `--exclude` ([Xcode projects](HOW-IT-WORKS.md#37-xcode-projects)).
- **Compare proof ids on one commit and one kind of machine.** The id includes `HEAD`, so a pull request merge commit gives another id than your branch. A line that only one platform builds is judged differently on another. To check a saved receipt later: `redmachine verify redmachine-receipt.jsonl --expect ID`, with the id from the job log ([the proof id](HOW-IT-WORKS.md#102-the-proof-id)).

## 8. Platform results

Version 0.1.0. Each build is named by the first eight characters of the SHA-256 of the program inside its archive. On 2026-11-23 the final programs were checked as below; the macOS program was checked after it was signed and notarized.

Every platform ran a 20-step check in a scratch home: `--version`, `--help`, `--list`, the licence messages with no licence, the refusal of a run with no licence, a machine licence supplied in `REDMACHINE_LICENSE`, a small Python repository that gave one KILLED and one SURVIVED line with exit 1 and a receipt, `redmachine verify` on that receipt, and `verify` refusing an altered copy. All 20 steps passed on all four platforms below.

**Linux x86_64** (program 4f79eb5a). Run on Linux 6.6 under WSL 2, which is how it runs on Windows. The binary is a statically linked position-independent executable. It passed the 20 steps.

**Linux arm64** (program f4e9e6f0). Run under `qemu-aarch64`, not on arm64 hardware. The binary is a statically linked aarch64 executable. Only the program was emulated; `python3` and `git` ran as x86_64. It passed the 20 steps.

**macOS on Apple silicon** (program 1eb32736, the universal file, signed with the hardened runtime and accepted by Apple's notary service). Run on macOS 27.2. It passed the 20 steps. The README install lines were run as written in a scratch home and `redmachine --version` printed the version. Xcode project support was built and tested with current Xcode on Apple silicon only.

**macOS on Intel**. The Intel half of the universal program (the same program 1eb32736) passed the 20 steps under Rosetta on Apple silicon, not on an Intel Mac. It was not run on an Intel Mac, and Xcode projects were not run on Intel.
