# gif-nv

GIF is an image format that stores a picture as a grid of indices into
a table of at most 256 colours, and can store a sequence of such
pictures as an animation. It is specified in
[GIF89a](https://www.w3.org/Graphics/GIF/spec-gif89a.txt), published by
CompuServe in 1990. The reference implementations this package is
measured against are the Rust crate [`gif`](https://docs.rs/gif) and
[Pillow](https://pillow.readthedocs.io/). This package decodes GIF
streams frame by frame and encodes them.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a GIF is

A GIF file is a **header**, a **logical screen descriptor**, and then a
sequence of **blocks** each introduced by a one-byte label. The
descriptor gives the **canvas** its width and height. Every picture
after it is a **frame**: a rectangle of pixels placed somewhere on that
canvas. A frame may be smaller than the canvas and may sit anywhere
inside it, which is how an animation stores only the pixels that
changed.

A pixel is an **index**, not a colour. The colours live in a **colour
table** of at most 256 entries, each three bytes of red, green and
blue. The stream may carry one **global colour table**, and each frame
may carry a **local colour table** of its own. A frame with a local
table uses it; a frame without one uses the global table; a frame with
neither cannot be drawn.

There is no alpha channel anywhere in the format. What GIF89a has
instead is one index per frame nominated as **transparent** by its
**Graphic Control Extension**: a pixel holding that index is not drawn,
and whatever was underneath shows through. Transparency is therefore
binary. A frame that fades cannot be expressed.

The Graphic Control Extension also carries the frame's **delay**, in
hundredths of a second, and its **disposal method** — what a renderer
does with the frame's area once the delay has elapsed and before the
next frame is drawn.

| Code | Method | What it means |
| --- | --- | --- |
| 0 | Unspecified | The file says nothing. Renderers treat it as "keep". |
| 1 | Keep | Leave the frame where it is. The next frame draws over it. |
| 2 | Background | Clear the frame's rectangle. |
| 3 | Restore | Put back what was there before this frame was drawn. |

The pixels themselves are compressed with **LZW**, a dictionary
compressor. GIF's variant packs its codes least significant bit first,
starts at a code width one bit above a **minimum code size** stored in
the image block, and grows that width by one each time the dictionary
fills, to a ceiling of twelve bits. It has an explicit **clear code**
that resets the dictionary and an explicit **end code**.

A frame's rows may be stored **interlaced**: four passes over the
image, every eighth row from row 0, then every eighth from row 4, then
every fourth from row 2, then every second from row 1. A viewer that
draws each pass as it arrives shows a whole blurry picture early rather
than a sharp top half.

An animation's repeat count is not in the format at all. It is carried
by the **NETSCAPE 2.0 Application Extension**, a convention every
renderer follows: a two-byte count where 0 means forever.

## Install

```
novo pkg add gif-nv
```

## Example

```novo
use std.bytes
use std.list
use gifread

fn main() [io]
    // The bytes of a file the caller already holds. This package never
    // opens anything.
    let raw = bytes.from_byte_list([0x47, 0x49, 0x46, 0x38, 0x39, 0x61])

    // Decode every frame at once. `feed` and `finish` are the
    // streaming form, for bytes that are still arriving.
    match gifread.frames_of(raw, gifread.default_options())
        Err(e)     => println("not a GIF: ${e.message()}")
        Ok(frames) =>
            println("${list.len(frames)} frame(s)")
            // A frame is a rectangle placed on the canvas, not the
            // whole picture: an optimised animation's later frames are
            // often a few pixels.
            let first = list.get(frames, 0)
            println("${first.width} by ${first.height} at ${first.left},${first.top}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: gif-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `giffault` | Every reason a stream is refused, each with the byte offset it happened at. |
| `gifpalette` | Colour tables, the transparent index, and the median-cut quantiser with its three dithers. |
| `gifframe` | The canvas, a frame, the four disposal methods, interlace row order and the loop extension. |
| `giflzw` | GIF's variable-width LZW, as a decoder and an encoder the caller drives byte by byte. |
| `gifread` | The decoder: bytes in, events out. |
| `gifwrite` | The encoder: frames in, bytes out. |

## How to choose an entry point

**`gifread.frames_of` decodes a whole file at once.** Use it when the
bytes are already in memory and you want the pictures.

**`gifread.feed` and `gifread.finish` decode a stream.** Use them when
the bytes arrive a few at a time — from a socket, from a file read in
pieces, from an archive member. `feed` answers the events those bytes
completed, and `finish` is how you say there are no more.

**`gifread.probe_size` reads only the canvas size.** Use it before
deciding to decode at all. It costs thirteen bytes and no allocation.

**`gifwrite.encode_indexed` writes one image whose pixels are already
palette indices.** Nothing is quantised, so an image drawn from a
chosen palette is written exactly.

**`gifwrite.encode_rgb` and `gifwrite.encode_animation` take true
colour and quantise.** Use the animation form for a sequence: it builds
one palette from every frame, and one palette is what keeps the colours
from drifting between frames.

## The rules a user needs

1. **A frame is a patch, not a picture.** It has its own position and
   size inside the canvas (section 20), and an optimised animation's
   second frame is often a few pixels. Code that assumes every frame
   fills the canvas breaks on the first optimised file it meets.

2. **The local colour table wins.** A frame uses its own table if it
   has one, the global table otherwise, and a frame with neither cannot
   be drawn (sections 19 and 21). `gifframe.palette_for` is that rule.

3. **Transparency is one index, not a channel.** A pixel is drawn or it
   is not (section 23). Alpha is 0 or 255 and nothing between.

4. **Disposal method 2 is not what the specification says.** The
   specification says fill the frame's rectangle with the background
   colour. Every browser fills it with transparent instead. A file that
   relies on either behaviour looks wrong under the other.

5. **A delay below 2 is not honoured by browsers.** The field is
   hundredths of a second, and every major browser substitutes 10 for
   anything below 2. This package reports what the file says;
   `gifframe.browser_delay_cs` computes what a browser will do.

6. **The minimum code size has a floor of 2.** Even a two-colour image
   uses 2, because a one-bit code width leaves no room for the clear
   and end codes (appendix F).

7. **A table size is a power of two between 2 and 256.** The size is
   stored as a three-bit exponent (section 18), so a table of 19
   colours is written as a table of 32.

8. **The canvas size is two two-byte fields.** Their product is four
   thousand million, so a thirteen-byte header can ask for a buffer no
   machine has. `gifread.probe_size` and the `max_pixels` option are
   how a decoder refuses before allocating.

9. **The repeat count lives in an extension, and "absent" is not
   "once".** A file with no NETSCAPE extension and a file with one
   saying 1 are different files. `GifLoop` has three states for that
   reason.

10. **Every buffer is a `Bytes`.** The compressed stream, a frame's
    colour indices and the encoder's output are all `Bytes`, which is
    the type png-nv, qoi-nv and image-nv already use for a file image
    and for pixel samples. A caller moving a picture between them
    copies nothing.

## What is not included

- **Compositing.** This package decodes the disposal methods and does
  not apply them. Compositing needs a canvas buffer held across frames,
  which is an allocation policy that belongs to whoever owns the
  screen. image-nv is the package that will composite, over this one.
- **Colour space conversion.** A GIF colour table is sRGB by
  convention and carries no profile. color-nv converts.
- **Perceptual quantisation.** `gifpalette.nearest_index` measures
  squared distance in sRGB, which is what every GIF encoder does and is
  not perceptual. A caller that wants the perceptual answer converts to
  Oklab with color-nv and quantises there.
- **Reading the Plain Text Extension as text to render.** The block is
  reported as an unknown extension. Nothing has rendered it since 1995.
- **Writing GIF87a.** Both versions are read. Everything written is
  GIF89a, because the extensions this package writes are GIF89a's.
- **Frame differencing.** An encoder that worked out which pixels
  changed between two frames and wrote only those would be an
  optimiser. `gifwrite.add_frame` writes the rectangle it is given, and
  a caller that has computed a difference passes the smaller rectangle.

## Related packages

**color-nv** owns `Srgb8`, which is what a colour table entry is, and
`Srgba8`, which is what a pixel becomes once the transparent index has
been applied.

**image-nv** is the multi-format front. It will depend on this package
for the animated format; this package does not depend on it.

**png-nv** and **qoi-nv** are the other still formats. PNG carries a
real alpha channel and up to sixteen bits a channel, so a picture that
needs either is a PNG.

**image-io-nv** sniffs a file's format and decides what can decode it.

## Test vectors

The suite is written against the GIF89a specification itself.

- **Appendix F** works one LZW stream end to end and is asserted as a
  decode, as an encode, and as a round trip.
- **Section 18** fixes the eight colour table sizes and the three-bit
  exponent that stores them.
- **Section 20** fixes the four-pass interlace row order, which is
  asserted row by row for an eight-row frame.
- **Section 23** fixes the disposal codes, the transparent index and
  the delay field.
- The **NETSCAPE 2.0 Application Extension** fixes the loop count's two
  bytes, and the three states `GifLoop` distinguishes are asserted
  separately.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| Colour tables, transparency, median cut, dithering | declared, not implemented |
| Canvas, frames, disposal, interlace, loop extension | declared, not implemented |
| LZW decoding and encoding | declared, not implemented |
| Feed-and-drain decoding | declared, not implemented |
| Encoding, still and animated | declared, not implemented |
| Compositing | not declared — image-nv's |
| Frame differencing | not declared |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
