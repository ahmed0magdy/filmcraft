STATUS: BLOCKED - harness change done and green on main without #1609; the safety check blocked the matching README line; need your OK
UPDATED: 2026-10-13T23:28:58Z
BRANCH: contrib/interchange-fuzz (local only, not committed; will be renamed contrib/interchange-overflow-fuzz)
MODE: Auto

# Blocked: one README line needs your OK

## Done

- `crates/interchange/tests/fuzz.rs` is updated after your "try again".
  - The URL regression test, which now belongs to #1609, is removed.
  - A panic hook records where each fuzz panic happens. Only the #1609 panic (`interchange/src/common.rs:68`, "is not a char boundary") is left out of the default run. A doc comment explains why, and `FILMCRAFT_FUZZ_URL_PANICS=1` counts it again.
- **Results on current main, without #1609:**
  - The default harness passes in debug (1.6 s) and release (1.1 s).
  - A debug run of 30,000 mutations per format passes.
  - With `FILMCRAFT_FUZZ_URL_PANICS=1` the only failure is the #1609 URL panic, so the exclusion hides nothing else.
  - `cargo fmt --check` passes.
- `cargo xtask ci` is now running on this state.

## Blocked

The README test list in `crates/interchange/README.md` still says "Media file URLs with non-ASCII text left unescaped import as their paths", which is no longer true on this branch. The safety check blocked my edit replacing it with a note about the left-out #1609 panic and the env var. The README doesn't affect the gates.

**Choose one:**

- **(a)** I replace that sentence with the note about the left-out panic (recommended);
- **(b)** I only delete the sentence;
- **(c)** you make the change yourself after I push.

I won't commit until the README is settled and `cargo xtask ci` has passed.
