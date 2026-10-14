STATUS: DONE - contrib/interchange-overflow-fuzz pushed (93d8e3e): FCPXML time and FCP7 timebase overflow fixes plus an interchange mutation-fuzz harness; complements #1609
UPDATED: 2026-10-14T06:53:50Z
BRANCH: contrib/interchange-overflow-fuzz
MODE: Auto

# Done: `contrib/interchange-overflow-fuzz`

- **Commit:** `93d8e3e`, authored and committed by `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`. The message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` and has no `Claude-Session` trailer.
- **Base and push:** branched from `main` at `ade0f36`; pushed with a normal push. No PR or issue was opened.
- **Subject:** `interchange: overflow-safe FCPXML times and FCP7 timebases, mutation-fuzz harness`. There is no issue to name, so this follows docs/contributing.md §4 ("If your change has no task id, use the area") and upstream's existing lowercase area subjects, such as `export: clip caption sidecars to the exported range`. The work belongs under gaps.md G4 (stability).
- **Files:** 4 files, +237 / -6: `crates/interchange/src/fcpxml.rs`, `crates/interchange/src/common.rs`, `crates/interchange/tests/fuzz.rs` (new) and `crates/interchange/README.md`.

## PR notes

- Fixes an integer overflow in FCPXML time parsing (`parse_time`, `crates/interchange/src/fcpxml.rs`). A time such as `"99999999999999999999s"` or `"<huge>/1s"` panicked in debug builds and silently wrapped to a wrong tick count in release builds. It now uses checked arithmetic and returns `None`, so the time is treated as unreadable like any other bad time.
- Fixes an integer overflow in FCP7 XML rate parsing (`rate_from_timebase`, `crates/interchange/src/common.rs`): a hostile NTSC `<timebase>` overflowed `timebase * 1000`. It saturates now.
- Each fix has a regression test that panicked before the fix: `fcpxml::tests::times_too_long_are_rejected` and new assertions in `common::tests::rates`.
- Adds `crates/interchange/tests/fuzz.rs`, the mutation-fuzz tests AGENTS.md §0.8 asks parsers to have. Mutated EDL, FCP7 XML, FCPXML, OTIO, AAF, OMF and ALE files must never panic under `catch_unwind`. Mutations: truncation, bit flips, hostile numbers, removed or duplicated spans, hostile 32-bit sizes. 2,000 mutations per format by default (about 2 s in debug); `FILMCRAFT_FUZZ_ITERS=<n>` runs deeper.
- Complements #1609, which fixes the `file_url_to_path` UTF-8 char-boundary panic. That fix is not in this PR. Its panic is left out of the harness's default run, matched narrowly by `interchange/src/common.rs:68` plus "is not a char boundary". `FILMCRAFT_FUZZ_URL_PANICS=1` counts it again. Once #1609 merges, the exclusion (`known_url_panic`) can be removed so the harness covers that case too.
- The fuzz harness found the two overflows. With the #1609 class left out, a 30,000-mutation-per-format debug run and a 150,000-per-format release run found nothing else.
- **Verification (Linux x86_64, 4 vCPU, rustc 1.97.0, ffmpeg 6.1.1 from Ubuntu):**
  - `cargo fmt --check`: passed
  - `cargo clippy --workspace --all-targets --release -- -D warnings`: passed
  - `cargo xtask layers`: passed
  - `cargo xtask assets`: passed (69 assets attributed)
  - `cargo xtask wasm`: passed (49 crates)
  - `cargo test -p filmcraft-interchange`: 81 passed, 0 failed
  - `cargo test --workspace --no-fail-fast` (debug): 3,419 passed, 17 failed, 44 ignored. All 17 failures are on `main` too: I re-ran each failing test binary on unmodified `main` and got the same 17 failures at the same lines. None are in `filmcraft-interchange`. They are tied to this VM: no GPU, and Ubuntu's ffmpeg 6.1.1 as the oracle.
    - `filmcraft-gpu` (lib): `fx_tests::alpha_invert_does_not_amplify_invented_opacity_at_odd_source_pixels`, `effect_source_identity_preserves_odd_sized_half_float_texels`, `effect_source_integer_decimation_preserves_cpu_box_oracle`, `gpu_effect_chains_match_cpu`, `gpu_effects_match_cpu_exactly`, `gpu_extract_zero_softness_has_no_fractional_edge`, `hostile_parameters_are_bounded`, `keyframed_effects_follow_time`, `plain_layers_skip_the_effect_stage`
    - `filmcraft-engine` (lib): `export_tests::every_builtin_preset_exports_and_ffprobe_confirms`, `nest_editing_tests::with_the_toggle_off_a_sequence_edits_in_as_its_clips_with_their_transitions`
    - `filmcraft-cli` (`tests/workflows.rs`): `phone_vfr_clip_extract_gain_title_to_h264_mp4` (export frame difference 19.04 against a limit of 6)
    - `filmcraft-matroska` (`tests/oracle.rs`): `mjpeg_ac3`
    - `filmcraft-media` (`tests/exr_oracle.rs`): `exr_sequence_plays_linear_float_frames`, `exr_stills_match_ffmpeg`
    - `filmcraft-media` (`tests/tiff_ycbcr.rs`): 1 test (panics at `tiff_ycbcr.rs:154`)
    - `filmcraft-ui-egui` (`tests/project_panel_ui.rs`): 1 test (panics at `project_panel_ui.rs:793`)
  - **Not a full `cargo xtask ci` pass.** Its release-mode test step did not finish within the 2-hour background limit on this VM: it never got past compiling, so no release test ran. The workspace tests were run in debug instead, with `CARGO_PROFILE_DEV_DEBUG=0` and `CARGO_INCREMENTAL=0`, because full debug info filled the session's disk (the problem tracked in #616 / #618). That changes only build size, not what the tests check. Please run `cargo xtask ci` on a larger machine before opening the PR, or note this in it.

## Explain it to the owner

### 1. The bug

FilmCraft imports edits from other editors: Final Cut Pro X writes FCPXML, Premiere and Final Cut Pro 7 write "FCP7 XML". Times and frame rates in those files are numbers written by someone else, and two places multiplied them without checking whether the result fits in a 64-bit integer.

- **FCPXML time.** FCPXML writes times as seconds, such as `start="5s"` or `duration="1001/24000s"`. FilmCraft stores time in "ticks", 254,016,000,000 per second. `start="99999999999999999999s"`, or any whole-second value above about 36.3 million (`i64::MAX` / 254,016,000,000), overflows that multiplication.
  - **Before:** a debug build panics ("attempt to multiply with overflow"). A release build silently wraps around and gets a meaningless, possibly negative time.
  - **After:** `parse_time` returns `None` ("not a usable time"), and the importer treats the time as missing, as it already did for text like `start="abc"`.
- **FCP7 timebase.** A file says `<rate><timebase>9223372036854775807</timebase><ntsc>TRUE</ntsc></rate>`. For NTSC rates the code computes `timebase * 1000 / 1001` (24 → 23.976).
  - **Before:** the `* 1000` overflows: a debug panic, or a wrapped rate in release.
  - **After:** the multiplication saturates at the largest `i64`, which gives an absurd but valid rate and no crash.

### 2. How the fix works

1. **`crates/interchange/src/fcpxml.rs`, `parse_time` (line 49).** It has two numeric paths:
   - **Fractions `a/b`:** `a * TICKS`, then a rounded divide by `b`. Each step is now a checked operation (`checked_mul`, `checked_div_euclid`, `checked_rem_euclid`, `checked_abs`, `checked_add`). Any overflow ends the function with `None` through `?`. The final cast from `i128` to `i64` used to be `as i64`, which silently truncates; it is now `i64::try_from(q).ok()`, which also gives `None` when the value doesn't fit.
   - **Whole seconds:** `i * TICKS_PER_SECOND` became `i.checked_mul(TICKS_PER_SECOND)`.
   - **Decimal seconds** such as `"3.5s"` were already safe: they go through floating point, and Rust's float-to-int cast saturates.
2. **`crates/interchange/src/common.rs`, `rate_from_timebase` (line 201).** `tb * 1000` became `tb.saturating_mul(1000)`. It already clamped negative and zero timebases to 1 with `max(1)`.
3. **Why this approach:**
   - Returning `None` for a time is the function's existing way of saying "unreadable". All 13 callers already handle it: they use `Tick::ZERO` or skip the element, as `attr_time(..).unwrap_or(Tick::ZERO)` does. No caller needed to change.
   - The alternative, saturating the time to the maximum tick, would place a clip about 1,000 years into the timeline. A missing time is more honest.
   - For the frame rate, `rate_from_timebase` returns a `FrameRate`, not an `Option`. Saturating keeps its signature and every caller unchanged, so the PR stays small.
4. **`crates/interchange/tests/fuzz.rs` (new).** It builds a small project (two clips, linked audio, two video tracks, a cross dissolve, a non-ASCII media path) and exports it in all six formats with FilmCraft's own exporters. Those files are the "seeds".
   - `mutate` makes 1 to 4 random changes to a copy of a seed: cut it short, flip a bit, write a special byte, replace a number with a hostile one, delete or duplicate a chunk, or write a huge 32-bit value.
   - `never_panics` imports every mutated file inside `catch_unwind` and fails the test with a list of each distinct panic. An error result is fine; only a panic fails.
   - The random generator is a fixed-seed xorshift, so every run tests exactly the same inputs and failures reproduce.
   - The ALE parser gets its own seed and test.
5. **The #1609 exclusion.** `install_hook` sets a panic hook that, only while a fuzz input runs, records the file and line where the panic happened; any other panic, such as a failing assertion, still prints normally. `known_url_panic` matches only `interchange/src/common.rs:68` with "is not a char boundary", and only when `FILMCRAFT_FUZZ_URL_PANICS` is unset.

### 3. Why it is correct

**What the tests prove:**
- `times_too_long_are_rejected` covers the largest whole-second value that fits (still parsed exactly), the first one that doesn't (`None`), `i64::MIN` seconds, `i128` extremes in the numerator and the denominator, and that a normal negative fraction still parses. It panicked before the fix.
- `rates` now checks that `i64::MAX` and negative NTSC timebases don't panic. It also panicked before the fix.
- The existing `rational_times_are_exact` test still passes, so normal times parse exactly as before.
- Every `filmcraft-interchange` test passes: 81 tests, including the round trips in `aaf_omf.rs`, `fcp7.rs`, `fcpxml.rs`, `otio.rs` and `edl.rs`.
- The fuzz harness passes in debug and release. Deeper runs found nothing else: 30,000 mutations per format in debug, 150,000 per format in release.

**What they do not cover:**
- The fuzzer only mutates files FilmCraft itself exported. Structures FilmCraft never writes, such as some Premiere or Resolve FCPXML features, are only reached when a random mutation happens to create them.
- Overflow panics only in debug builds. Release builds wrap silently, so a release-only fuzz run would miss overflow bugs; that's why the harness should also be run in debug.
- Later arithmetic on parsed times, such as `origin + offset` at fcpxml.rs:339, was not changed. Values that each fit could in theory still overflow when added. The fuzzer didn't find a case.
- AAF and OMF are binary formats, and the mutations aren't format-aware there, so coverage of their deeper structures is shallow.

### 4. Likely maintainer questions

1. **"Why `None` and not an error message?"** `parse_time` already returns `Option`, and every caller already treats `None` as "missing, use zero". Adding a warning to the import report would touch every caller. That could be a follow-up.
2. **"Why saturate the frame rate instead of rejecting it?"** `rate_from_timebase` returns a `FrameRate`, not an `Option`. Saturating avoids a panic without changing the signature or its callers. A real file never has a timebase anywhere near this size.
3. **"Why is there a panic exclusion in the fuzz test?"** Main still has the URL panic, and #1609 fixes it in a separate PR. Without the exclusion this harness would fail on main until #1609 merges. It is narrow (one file and line, one message), can be switched off with `FILMCRAFT_FUZZ_URL_PANICS=1`, and should be deleted after #1609 lands.
4. **"How long does the fuzz test take?"** About 1 to 2 seconds per run in this VM: 2,000 mutations per format, 4,000 for ALE.
5. **"Is it deterministic?"** Yes. The random generator always starts from the same seed, so a failure reproduces exactly. Deeper runs use `FILMCRAFT_FUZZ_ITERS`.

### 5. Risks

- **A valid file now imports differently.** Only times above about 36 million seconds (over a year) changed, from a wrapped garbage value to "missing". No real edit is that long. Normal times are unchanged: the existing exactness test and the round-trip tests pass.
- **The panic hook affects other tests.** The hook is installed only in the `fuzz` test binary, and it only stays silent while a fuzz input runs on that thread. Assertion failures still print, which I confirmed when the test failed during development.
- **Line-number matching becomes stale.** If someone edits `common.rs` above line 68 before #1609 merges, the exclusion stops matching and the harness fails loudly; it never hides a panic by mistake. #1609 changes that code anyway, after which the exclusion should be removed.
- **Merge conflict with #1609.** Both PRs touch `crates/interchange/src/common.rs`, but in different functions (`file_url_to_path` versus `rate_from_timebase`) and different test functions. Git should merge them without conflict.

### 6. Read the diff in this order (about 10 minutes)

1. `crates/interchange/src/fcpxml.rs`, `parse_time`: the checked arithmetic and the final `try_from`.
2. `crates/interchange/src/fcpxml.rs`, test `times_too_long_are_rejected`: the boundary values.
3. `crates/interchange/src/common.rs`, `rate_from_timebase`, and the two new lines in test `rates`.
4. `crates/interchange/tests/fuzz.rs`, top doc comment: what it does and why the #1609 panic is excluded.
5. `fuzz.rs`, `mutate`: the kinds of damage applied.
6. `fuzz.rs`, `install_hook`, `known_url_panic` and `never_panics`: how panics are caught, located and reported.
7. `fuzz.rs`, `project` and the two `#[test]` functions: the seeds.
8. `crates/interchange/README.md`: the one new bullet.

## A possible next item (not started)

**#985: `--automation-read-root` / `--automation-write-root` for `filmcraft-cli mcp`.**
- **What:** confines the MCP server's path parameters to given folders. It's headless and testable: paths outside the roots, `..`, symlinks and paths inside `engine.execute` must be refused.
- **Size:** medium.
- **Caveat:** the issue says to match VectorCraft and EffectCraft, whose design I can't read from this session, so a pointer to their flag semantics would help.

The alternative is **#1535** (a right-click menu for the Timeline ruler). It's smaller, but it's UI work that needs a visual check.
