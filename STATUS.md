STATUS: WAITING - all gates run (no new failures; 17 tests also fail on plain main); only the README line needs your choice before I commit and push
UPDATED: 2026-10-14T02:38:40Z
BRANCH: contrib/interchange-fuzz (local only, not committed; will be renamed contrib/interchange-overflow-fuzz)
MODE: Auto

# Gates done; one README decision left

## Gate results on this branch

| Gate | Result |
|---|---|
| `cargo fmt --check` | passed |
| `cargo clippy --workspace --all-targets --release -- -D warnings` | passed |
| `cargo test --workspace` (debug, `--no-fail-fast`) | 3,419 passed, 17 failed, 44 ignored |
| `cargo xtask layers` | passed |
| `cargo xtask assets` | passed (69 assets attributed) |
| `cargo xtask wasm` | passed (49 crates) |

**Why the tests ran in debug:** `cargo xtask ci` runs its tests in release mode, and that build did not finish within the 2-hour limit on this 4-vCPU VM. In debug, building the workspace tests with full debug info then filled the session's disk, so they ran with `CARGO_PROFILE_DEV_DEBUG=0` and `CARGO_INCREMENTAL=0`. That only changes build size; the tests check the same things. This is the dev-build disk problem upstream tracks in #616 and #618.

**Every test in `filmcraft-interchange` passed.** That includes the new fuzz harness and the overflow regression tests.

**All 17 failures are pre-existing.** I re-ran each failing test binary on plain `main` without these changes and got the same 17 failures at the same lines. They are tied to this VM (no GPU, Ubuntu's ffmpeg 6.1.1) and none are in `filmcraft-interchange`:

- `filmcraft-gpu` `fx_tests`: 9 tests. The VM has no GPU.
- `filmcraft-engine`: `export_tests::every_builtin_preset_exports_and_ffprobe_confirms` and `nest_editing_tests::with_the_toggle_off_a_sequence_edits_in_as_its_clips_with_their_transitions`.
- `filmcraft-cli` `workflows`: `phone_vfr_clip_extract_gain_title_to_h264_mp4` (picture difference 19.04, limit 6).
- `filmcraft-matroska` `oracle`: `mjpeg_ac3`.
- `filmcraft-media` `exr_oracle`: 2 tests.
- `filmcraft-media` `tiff_ycbcr`: 1 test.
- `filmcraft-ui-egui` `project_panel_ui`: 1 test.

## Needs your choice

The README test list in `crates/interchange/README.md` still says "Media file URLs with non-ASCII text left unescaped import as their paths", which is no longer true on this branch. The safety check blocked my edit to it.

- **(a)** I replace that sentence with a note about the left-out #1609 panic and `FILMCRAFT_FUZZ_URL_PANICS=1` (recommended).
- **(b)** I only delete the sentence.
- **(c)** You edit it yourself after I push.

After your answer I will commit as you, with subject `interchange: …`, rename the branch to `contrib/interchange-overflow-fuzz`, push it, and write the DONE status with PR notes and the "Explain it to the owner" section.
