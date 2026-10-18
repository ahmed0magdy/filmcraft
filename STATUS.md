STATUS: DONE - #1840 branch rebased onto main (with #1609) and the fuzz exclusion removed in its own commit; new head 67efe0f
UPDATED: 2026-10-18T05:29:32Z
BRANCH: contrib/interchange-overflow-fuzz
MODE: Auto

# Done: storytold/filmcraft#1840 updated

## Commits

- **New head:** `67efe0f0d6b75cf6ff432c16cc92e7bee8689925`. The old head was `93d8e3e6519826d6e2e82859d38e5523ec44d269`.
- **Base:** `origin/main` at `4a69a038d32761f604b3c7e4845b21fb40b3cbe4` (contains #1609 as `0b432e3`).
- **Commit 1:** `9eacf57ac14a18bc09f81cd5d2ee7aed09d52282` `interchange: overflow-safe FCPXML times and FCP7 timebases, mutation-fuzz harness`. This is `93d8e3e` rebased. The rebase had no conflicts. Message, author and changed lines are identical to `93d8e3e`; only the diff context lines moved, because #1609 added a README paragraph above.
- **Commit 2:** `67efe0f0d6b75cf6ff432c16cc92e7bee8689925` `interchange: the fuzz harness counts every panic now that #1609 is in`. It changes only `crates/interchange/tests/fuzz.rs` (-20 lines), removing:
  - `known_url_panic`;
  - the `FILMCRAFT_FUZZ_URL_PANICS` switch;
  - the module-doc lines about the exclusion;
  - the "left out … known file-URL panics" `eprintln!`.
- **Kept:** the panic hook, which now only adds the panic's `file:line` to each failure report.
- **README:** the line this PR adds never mentioned the exclusion, so it needed no change.
- **Authorship:** both commits are authored and committed by `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`, with the `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` trailer and no `Claude-Session` trailer.
- **Signatures:** neither commit is signed, the same as `93d8e3e`, so GitHub may show them as "Unverified". A session hook suggested re-authoring them as Claude; I did not, because of your authorship rule.
- **Push:** `git ls-remote` showed `93d8e3e…` right before the push, and `git push --force-with-lease=contrib/interchange-overflow-fuzz:93d8e3e65` succeeded. The remote is now `67efe0f…`.
- No PR, issue or comment was opened or posted, and nothing was scheduled.

## Checks

**Ran on the rebased tree (head `67efe0f`):**

- `cargo test -p filmcraft-interchange`: 83 passed, 0 failed, 0 ignored. Per binary:
  - unit tests 22
  - `aaf_omf` 14
  - `api` 3
  - `edl` 11
  - `fcp7` 12
  - `fcpxml` 3
  - `file_urls` 2 (from #1609)
  - `fuzz` 2, with no exclusion
  - `media_marker_export` 1
  - `nested_sequence_markers` 2
  - `otio` 4
  - `point_markers` 2
  - `uncarried_clips` 5
- `FILMCRAFT_FUZZ_ITERS=30000 cargo test -p filmcraft-interchange --test fuzz` (debug): passed in 20 s. That is 30,000 mutations for each of the 6 document formats (180,000 mutated documents, each imported twice and run through `detect` twice) plus 30,000 mutated ALE files. **No panic.**
- `FILMCRAFT_FUZZ_ITERS=150000 cargo test --release -p filmcraft-interchange --test fuzz`: passed in 70 s. That is 900,000 mutated documents plus 150,000 ALE files. **No panic.**
- `cargo fmt --check`: passed.
- `cargo clippy --workspace --all-targets --release -- -D warnings`: passed.
- `cargo xtask layers`: passed.
- `cargo xtask assets`: passed (69 assets attributed).
- `cargo xtask wasm`: passed (49 crates).

All debug runs used `CARGO_INCREMENTAL=0 CARGO_PROFILE_DEV_DEBUG=0` to fit the session's disk; that changes build size only.

**Not run this time:**

- `cargo test --workspace`. The second commit changes only a test file in `filmcraft-interchange`, and the rebase pulled in other crates' changes from `main`, which this PR does not touch. On 2026-10-14 it gave 3,419 passed / 17 failed / 44 ignored, and all 17 failures also happen on plain `main` in this VM.
- `cargo xtask ci`. Its release-mode test step cannot finish within this VM's 2-hour limit.
- No ffmpeg oracle tests are involved: the interchange tests don't use ffmpeg.

## Replacement text for the PR body

### "Relation to #1609" section

> #1609 (merged as `0b432e3`) fixed the `file_url_to_path` UTF-8 char-boundary panic. This branch is rebased onto a `main` that contains it. The fuzz harness no longer leaves that panic out: the second commit removes the temporary exclusion (`known_url_panic` and `FILMCRAFT_FUZZ_URL_PANICS`), so every panic in a mutated document fails the test. The harness passes with #1609's fix in place, and #1609's own `tests/file_urls.rs` passes alongside it.

### "Results" bullet

> - Fuzz harness, no exclusions, on `main` with #1609: the default run (2,000 mutations per format, 4,000 for ALE) passes. `FILMCRAFT_FUZZ_ITERS=30000` in debug (180,000 mutated EDL / FCP7 XML / FCPXML / OTIO / AAF / OMF documents plus 30,000 ALE files) and `FILMCRAFT_FUZZ_ITERS=150000` in release (900,000 documents plus 150,000 ALE files) found no panic. Before the fixes, the same harness found the FCPXML time overflow and the FCP7 timebase overflow fixed here, and the file-URL panic that #1609 fixed.

## Explain it to the owner

### What changed since the PR was opened

When you opened #1840, upstream still had the file-URL crash: a `file://` path with a character like "ö" at the wrong byte made the importer panic. Someone else's PR, #1609, was fixing it, so our fuzz test temporarily ignored that one crash; otherwise the test would have failed on `main` for a bug that wasn't ours to fix. #1609 is now merged, so the ignore rule has no reason to exist. I removed it, and the fuzz test now fails on any crash at all.

### What I did, step by step

1. **Moved the branch onto the new `main` (a rebase).** Your one commit now sits on top of `main` (`4a69a03`), which includes #1609. Nothing in it conflicted with #1609: they touch different functions in `common.rs` and different spots in the README. I checked that the rebased commit has the same message, author and changed lines as before (`9eacf57`).
2. **Removed the ignore rule in a second, separate commit** (`67efe0f`), so a reviewer can see that change on its own. It only deletes code from `crates/interchange/tests/fuzz.rs`:
   - the function `known_url_panic`, which recognised that one crash by its file, line and message;
   - the `FILMCRAFT_FUZZ_URL_PANICS` environment switch;
   - the comment at the top that explained the exception;
   - the line that printed how many crashes were ignored.
3. **Kept the panic hook.** It records where a crash happened, so a failure report still says which file and line crashed. That is useful on its own.
4. **Pushed safely.** First I checked that GitHub still had exactly the old commit `93d8e3e`. Then I used a "force with lease" push, which refuses to overwrite the branch if anyone else has pushed to it since. It went through, so nobody had.

### Why it is correct

- **The fuzz test proves more than before.** With nothing excluded it passed 1,260,000 broken files in total (180,000 + 30,000 in debug, 900,000 + 150,000 in release) without a single crash. Before, a crash in `file_url_to_path` would have been silently skipped; now it would fail the test.
- **#1609's fix and our test agree.** If #1609's fix didn't hold, the same harness would now fail, as it did on old `main`.
- **What it does not prove:** the fuzzer only damages files that FilmCraft itself exported, so formats or features FilmCraft never writes are only lightly covered. That limit hasn't changed.

### Questions a reviewer might ask

1. **"Why two commits?"** The first is the PR as reviewed so far, unchanged. The second is the small follow-up #1609 made possible, and you said in the PR body that you would make it.
2. **"Why rebase instead of merging `main` in?"** FilmCraft merges PRs with real merge commits, so a "Merge main" commit inside the PR would end up in their history. A rebase keeps the PR as just its own two commits.
3. **"Is the panic hook still needed?"** It is optional now. It adds the crash location to the failure message, which makes a future failure faster to diagnose. It stays quiet only while a fuzz input runs, so other test failures print normally.
4. **"Did the fuzz find anything new?"** No: 1,260,000 inputs, no panic.
5. **"Why are the commits Unverified?"** They are unsigned, like the original commit. If you want them signed, re-sign them with your key; that changes the SHAs.

### Risks

- **Rebasing changed the commit SHAs** (`93d8e3e` → `9eacf57`, plus the new `67efe0f`). Nobody had reviewed or commented on #1840, so no review comment points at the old SHA.
- **A future crash in the importers will now fail this test,** including one in someone else's change. That is the point of the test. The failure message names the format, the crash location and the start of the input that caused it.

### Read the diff in this order

1. `git diff 9eacf57 67efe0f`: the second commit, deletions only, all in `tests/fuzz.rs`.
2. The first commit, if you want a refresher. It is unchanged; see the earlier explanation (`fcpxml.rs` `parse_time`, `common.rs` `rate_from_timebase`, then `tests/fuzz.rs`).
