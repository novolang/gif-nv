# Changelog

All notable changes to gif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- color-nv: `^0.0.1` to `^0.1.1`.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `gifread` — the load-bearing interface. The decoder NEVER READS:
  `feed` takes a slice of bytes the caller already holds and answers
  the events those bytes completed, and `finish` is the caller's
  statement that there are no more. "Not yet" is an empty event list
  and not an error; `GifTruncated` is the different statement, and it
  can only come from `finish`, because only the caller knows the
  stream ended. `GifFrameStarted` arrives before `GifFrameDecoded`
  carrying everything but the pixels, so a caller that only wants to
  know what is in a file never pays for them. `probe_size` reads the
  canvas out of thirteen bytes, which is what a size limit is checked
  against before a buffer is ever considered.
- `gifframe` — the canvas and the frames, with the four facts a
  renderer gets wrong made typed. A frame is a PATCH with its own
  position and size. The local colour table wins and a frame with
  neither table has none, which is `palette_for` and not a rule every
  caller rewrites. `GifDisposal` has four arms including
  `GifDisposeUnspecified`, kept distinct from `GifDisposeKeep` because
  a file that says nothing and a file that says keep are different
  files. `GifLoop` has THREE states, because "no NETSCAPE extension"
  and "a NETSCAPE extension saying 0" are different files and an `Int`
  cannot hold the difference. The delay is reported as the file states
  it and `browser_delay_cs` is a separate function, so that reading a
  file never silently changes what it said.
- `gifpalette` — the tables and the quantiser. A table's size is
  checked at construction, because the format stores it as a three-bit
  exponent and only eight sizes exist. `resolve` answers `Srgba8` with
  an alpha of 0 or 255 and nothing between, which is what GIF
  transparency is. `median_cut_shared` takes every frame at once,
  because a palette computed per frame makes an animation shimmer.
  `nearest_index` measures squared distance in sRGB and says so: that
  is what every GIF encoder does and it is not perceptual.
- `giflzw` — the three ways GIF's LZW differs from the textbook
  algorithm, each one named: least-significant-bit-first packing, a
  width that starts one above the minimum code size, and explicit
  clear and end codes. Exposed as a state machine, because the bytes
  arrive in sub-blocks of at most 255 and a caller streaming from a
  socket must be able to stop between any two of them.
- `gifwrite` — the mirror. `add_frame` takes indices and quantises
  nothing, so an author with a palette keeps it exactly;
  `add_frame_rgb` is the lossy path and is a different function so it
  is asked for by name. `encode_animation` takes the whole sequence,
  because the shared palette cannot be computed until every frame's
  colours are known.
- `giffault` — sixteen refusals, each with the byte offset it happened
  at. `is_incomplete` is true for `GifTruncated` alone, which is what a
  caller streaming from a network asks to decide between waiting and
  giving up. `is_strict_only` separates the deviations the
  specification lets a decoder ignore from the ones it does not.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  gif-nv.<module>.<fn>`.
- **Compositing is not declared at all.** Applying the disposal
  methods needs a canvas buffer held across frames, which is an
  allocation policy. image-nv is the package that will do it, over
  this one, and this package does not depend on it.
- **Disposal method 2 diverges between the specification and every
  browser.** The specification says fill with the background colour;
  browsers fill with transparent. This package reports the method and
  the README states the divergence rather than choosing for a caller
  that has to interoperate with one or the other.
