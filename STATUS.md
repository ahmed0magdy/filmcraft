STATUS: WAITING - three candidates from the free-issue list; owner to pick one (recommended #1583) and decide on the paused interchange fix
UPDATED: 2026-10-13T07:30:29Z
BRANCH: contrib/interchange-fuzz (local only, paused, not pushed)
MODE: Auto

# FilmCraft upstream contribution: status

## Waiting on you

1. **Which candidate should I do?** I recommend **#1583**.
2. **Paused interchange fix: push it as a parked branch, or drop it?**

## Paused work: interchange importer crash fixes (not in the free list)

Before your free-issue list arrived I had started on your earlier pick. It is uncommitted on the local branch `contrib/interchange-fuzz` and has not been pushed. It fixes three importer crashes found by fuzzing:

- `file_url_to_path` panicked on a `file://` URL with non-ASCII text left unescaped, such as `file:///Users/Jöran/clip.mov`. This affects FCP7 XML, FCPXML, OTIO and AAF imports.
- FCPXML `parse_time` overflowed on very long times. Debug builds panicked and release builds silently wrapped the value.
- An FCP7 NTSC `timebase` overflowed in `timebase * 1000`.

Each fix has a regression test that panicked before the fix. A new `crates/interchange/tests/fuzz.rs` runs mutated EDL, FCP7 XML, FCPXML, OTIO, AAF, OMF and ALE files under `catch_unwind`; 100,000 mutations per format came out clean. The full `cargo xtask ci` run is still in progress. No open issue covers these crashes. If the gates pass I will commit it on the local branch only, so the work isn't lost; I will not push it unless you say so.

## Candidates (all from your free list)

### 1. #1583: still images ignore EXIF orientation (recommended)

- **What and why:** portrait phone photos import sideways. `crates/media/src/still.rs` decodes with `image::load_from_memory`, which never applies EXIF orientation. The locked `image` 0.25.10 can read the orientation tag and rotate the picture itself, so no new dependency is needed. The fix sits in the one function that decodes JPEG/TIFF/WebP stills, so imports, image sequences and thumbnails all get it. This fits G10 (import gaps).
- **Files:** `crates/media/src/still.rs`, tests in `crates/media`, a README line.
- **Verification:** synthetic JPEG and TIFF files with all 8 orientation values must decode with the right dimensions and pixel placement. Damaged EXIF must still import, since the tag is optional and just gets ignored. Then the full gates.
- **Overlap check:** #1570 and #1312 touch other still or image-sequence code (filenames, Y'CbCr TIFF), not orientation. No open PR mentions EXIF.
- **Size:** small.

### 2. #985: confine the MCP server with `--automation-read-root` / `--automation-write-root`

- **What and why:** today an agent driving `filmcraft-cli mcp` can read and write any path. The issue asks for the folder confinement VectorCraft and EffectCraft already have, and a CLI test currently asserts the flag is unknown. It sits next to #163, whose open PR #1172 gates the control server with a token. Folder confinement is a different control from that token, so it's not a duplicate.
- **Files:** `apps/filmcraft-cli/src`, `crates/automation/src/server.rs`, docs.
- **Verification:** headless MCP tests: paths outside the roots, `..`, symlinks and path parameters inside `engine.execute` must be refused; paths inside must work.
- **Caveat:** "as in VectorCraft" implies a design to match. I can't read those repos from this session, so the flag semantics would be my own unless you can point me to theirs.
- **Size:** medium.

### 3. #1535: right-clicking the Timeline time ruler opens the clip menu instead of a ruler menu

- **What and why:** in `crates/ui-egui/src/panels/timeline.rs` the right-click handler checks gaps, markers and clips but has no branch for the ruler. Fix: a ruler menu with Mark In, Mark Out, Mark Selection and the related clear commands, all of which already exist as engine commands. This is UI parity (G8).
- **Verification:** the gates plus a check through the control channel. Being UI work, it ranks below the other two.
- **Overlap check:** #1265 and #1219 change other timeline context menus (clip Add Marker, marker Label submenu), not the ruler.
- **Size:** small to medium.

### Considered and dropped

- #314: VP9 alpha in WebM is already implemented, so the issue looks stale.
- #1504: 3 comments and an unclear expected result.
- #1542 and #1544: the bug can't be pinned down without the issue body; for #1544, `jobs.list` already works over the bridge.
- #1176, #1370 and #1381: I couldn't reproduce them from the titles.
- #285: the ROADMAP progress log says it's already fixed.

## Setup facts

- **Checkout:** a GitHub clone of `ahmed0magdy/filmcraft` made through the session's git proxy (shallow), with `main` at `ade0f36`.
- **Push access:** `git push --dry-run origin HEAD:refs/heads/contrib/probe` succeeded.
