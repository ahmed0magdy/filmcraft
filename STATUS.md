STATUS: BLOCKED - auto-mode safety check refused the edit that keeps the #1609 URL panic out of the fuzz harness; need your go-ahead
UPDATED: 2026-10-13T13:59:49Z
BRANCH: contrib/interchange-fuzz (local only, nothing committed or pushed yet; will be renamed contrib/interchange-overflow-fuzz)
MODE: Auto

# Blocked: the fuzz-harness change needs your explicit go-ahead

## Done so far

- Commit identity in this repo is set to `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`. Nothing has been committed or pushed yet, so no re-authoring is needed.
- #1583 is dropped.
- Removed my `file_url_to_path` fix and its unit test from `crates/interchange/src/common.rs`; #1609 covers them.
- Kept the FCPXML `parse_time` overflow fix (`crates/interchange/src/fcpxml.rs`), the FCP7 `rate_from_timebase` overflow fix (`common.rs`) and their regression tests.

## What is blocked

On current main, without #1609, the fuzz harness's default run hits the `file_url_to_path` char-boundary panic. The run is deterministic (seeded generator), and it fails in both debug and release through a mutated FCP7 XML input. As you asked, I tried to edit `crates/interchange/tests/fuzz.rs` to:

1. delete `unescaped_non_ascii_media_urls_import`. That test checks the URL fix, which now lives in #1609, so on this branch it fails;
2. leave panics raised at `interchange/src/common.rs:68` with "is not a char boundary" out of the default run, record where each fuzz panic happens through a panic hook, explain why in a doc comment, and add `FILMCRAFT_FUZZ_URL_PANICS=1` to count them again once #1609 merges. Any other panic, anywhere, still fails the test.

The session's auto-mode safety check refused that edit as "security test removal". I have not tried to get around it.

## Your options

- **A (your plan):** confirm in chat that I should make that edit. If the check refuses again, add a permission rule for it (Bash or Edit on `crates/interchange/tests/fuzz.rs`) or switch the session out of Auto mode.
- **B:** keep the harness strict and land this PR after #1609. No exclusion is needed, but this branch stays red until #1609 merges.
- **C:** drop the harness from this PR and ship only the two overflow fixes and their unit tests; the harness can follow after #1609.

**I recommend A.** It matches your plan, and the exclusion is narrow (one file and line, one message) and easy to remove later. Once unblocked I will run `cargo xtask ci`, commit as you with subject `interchange: …`, rename the branch and push it.
