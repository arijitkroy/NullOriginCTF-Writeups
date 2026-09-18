# Null0rigin --- STEGO CHAIN

## Complete Solution Write-up

> Eight chained steganography stages. Each stage uses the exact flag
> recovered from the previous stage as the key for the next stage.

------------------------------------------------------------------------

## 1. Challenge Overview

The challenge consisted of eight sequential stages:

``` text
31-driftwood
32-safelight
33-moire
34-undertone
35-pentimento
36-winnow
37-mordant
38-quietus
```

The README established the most important rule of the entire chain:

``` text
The previous stage's flag is the key to the next one.
```

The operator's keycard then defined exactly how that key is transformed:

``` text
k = SHA-256(previous seal exactly as written)

number:
    first four octets of k, high octet first

stream:
    SHA-256(k || counter_be32)
    SHA-256(k || 1_be32)
    SHA-256(k || 2_be32)
    ...
```

The resulting stream is XORed against the relevant carrier data.

This meant that solving a stage was not enough. The recovered flag had
to be preserved exactly, including:

-   `Null0rigin`
-   braces
-   lowercase letters
-   underscores
-   no additional whitespace
-   no newline when hashing it

A single-character mistake invalidated the next stage.

The challenge also warned that several carriers were fragile. Re-saving
or re-encoding them could destroy the hidden information. We therefore
treated the original files as immutable evidence and created working
copies for analysis.

------------------------------------------------------------------------

# 2. Flag Chain

The complete chain we recovered was:

  -------------------------------------------------------------------------------------------------------
  Stage                   Challenge               Recovered flag
  ----------------------- ----------------------- -------------------------------------------------------
  31                      driftwood               `Null0rigin{pit_head_plate_number_four}`

  32                      safelight               `Null0rigin{what_the_print_kept_from_the_negative}`

  33                      moire                   `Null0rigin{a_carrier_laid_across_the_whole_room}`

  34                      undertone               `Null0rigin{the_order_of_the_colours_is_the_message}`

  35                      pentimento              `Null0rigin{keep_the_grain_and_burn_the_chaff}`

  36                      winnow                  `Null0rigin{nothing_takes_until_it_is_fixed}`

  37                      mordant                 `Null0rigin{he_set_two_sorts_where_one_would_do}`

  38                      quietus                 `Null0rigin{the_channel_closes_from_the_inside}`
  -------------------------------------------------------------------------------------------------------

The interesting part was that every stage used a different physical
metaphor for the same general idea: information was hidden in something
that looked irrelevant to a normal reader.

------------------------------------------------------------------------

# 3. Stage 31 --- driftwood

## Carrier

``` text
driftwood.png
plate-card.txt
```

The image was a 900×620 RGBA PNG.

The plate card explained that genuine information extracted from the
plate would appear as a `PL8` works entry. It also warned that
apparently readable text could simply be handwriting and should not be
trusted without validation.

The first mistake we made was trusting visible strings too quickly.

Running `strings` against the PNG immediately produced:

``` text
NullOrigin{the_tide_returns_what_it_took}
```

There was also an appended ZIP archive containing:

``` text
Null0rigin{recovered_from_the_shingle}
```

Neither was the actual answer.

The important lesson from Stage 31 was established immediately:

> A readable string is not automatically a flag.

## Inspecting the PNG

The PNG contained several `tEXt` chunks and, after the `IEND` chunk, an
appended ZIP archive.

The appended archive could be extracted with:

``` bash
python3 - <<'PY'
d = open("driftwood.png", "rb").read()
i = d.find(b"PK\x03\x04")
open("/tmp/drift.zip", "wb").write(d[i:])
PY

unzip -l /tmp/drift.zip
unzip -p /tmp/drift.zip shingle.txt
```

The archive explicitly called its recovered slip something that should
not be entered into the index.

That left the actual pixel data.

## Pixel extraction

Because the image was RGBA, we tested the individual channels and bit
planes.

The decisive extraction was the least significant bit of the first
channel, read in normal raster order and packed eight bits per byte.

Conceptually:

``` python
channel = image[:, :, 0]

bits = channel & 1

for every 8 bits:
    convert to one byte
```

The beginning of the resulting byte stream was:

``` text
Null0rigin{pit_head_plate_number_four}
```

This was a clean flag-shaped result rather than an incidental metadata
string.

## Result

``` text
Null0rigin{pit_head_plate_number_four}
```

This became the key for Stage 32.

------------------------------------------------------------------------

# 4. Stage 32 --- safelight

## Carriers

``` text
negative.png
print.png
keycard.txt
```

Both images were 900×620 grayscale PNGs.

The important clue was that the two images were complementary versions
of the same visual material:

-   `negative.png`
-   `print.png`

The print was derived from the negative, so visually comparing the two
images was more useful than treating either one independently.

## Deriving the Stage 32 key

The Stage 31 flag was hashed:

``` python
import hashlib

previous = b"Null0rigin{pit_head_plate_number_four}"
k = hashlib.sha256(previous).digest()

print(k.hex())
```

This gave:

``` text
8bb7b37b8238daf1d77062fd0ada0988014aab45acd3b65cb0db55a58b5af3de
```

The first four octets, interpreted big-endian, were:

``` text
8b b7 b3 7b
```

or:

``` text
2344072059
```

The useful property for this stage was the value modulo eight:

``` text
2344072059 % 8 = 3
```

This pointed us toward bit plane 3.

## Comparing the two bit planes

We extracted bit plane 3 from both images:

``` python
neg = (negative >> 3) & 1
prt = (print >> 3) & 1
```

Then XORed them:

``` python
mask = neg ^ prt
```

The difference image suddenly contained a readable line.

For visualization, the binary mask was scaled and enlarged. The
recovered text was:

``` text
Null0rigin{what_the_print_kept_from_the_negative}
```

The important conceptual point is that the flag was not simply hidden in
one image. It was encoded in what changed between the negative and the
print.

## Result

``` text
Null0rigin{what_the_print_kept_from_the_negative}
```

This became the key for Stage 33.

------------------------------------------------------------------------

# 5. Stage 33 --- moire

## Carrier

``` text
moire.png
screen-notes.txt
```

The PNG was:

``` text
1024 × 768
1-bit grayscale
```

The screen notes were the major clue.

They said the screen was:

``` text
FOUR TO THE CELL
```

and that each cell's weight was the number of black dots in it.

The crucial observation was that only the middle weights had two
possible settings.

## Splitting the image into 4×4 cells

Since the image dimensions are divisible by four:

``` text
1024 / 4 = 256 cells per row
768 / 4  = 192 cells per column
```

So there were:

``` text
256 × 192 = 49,152 cells
```

For every 4×4 cell we counted the black pixels.

The observed weights ranged through the expected halftone values.

The important discovery was that weights:

``` text
5, 6, 7, 8, 9, 10, 11
```

each had exactly two distinct cell arrangements.

For example, a weight-5 cell could appear in two different
configurations while still containing exactly five black pixels.

That was precisely the "same weight, different setting" described by the
notes.

## Recovering the compositor's choices

For each cell:

1.  Count the black dots.
2.  Ignore weights that have only one possible arrangement.
3.  For weights 5--11, identify which of the two arrangements was used.
4.  Record that choice as one binary value.

The cells were not read as ordinary rows.

The notes described the machine as moving:

``` text
down one furrow
back along the next
down again
```

which means a boustrophedon/serpentine traversal:

``` text
row 0: left → right
row 1: right → left
row 2: left → right
...
```

This traversal was essential.

## Applying the run key

The previous flag was:

``` text
Null0rigin{what_the_print_kept_from_the_negative}
```

and therefore:

``` python
k = SHA256(previous_flag)
```

The card's key schedule was then used to resolve the orientation of the
two possible settings.

The extracted choice stream was converted into bytes and validated
against the expected works-entry structure.

The recovered flag was:

``` text
Null0rigin{a_carrier_laid_across_the_whole_room}
```

## Result

``` text
Null0rigin{a_carrier_laid_across_the_whole_room}
```

This became the key for the audio stage.

------------------------------------------------------------------------

# 6. Stage 34 --- undertone

## Carriers

``` text
undertone.wav
linetest.wav
line-notes.txt
```

The audio files were stereo, 44.1 kHz, 16-bit PCM.

The long carrier was exactly 40 seconds:

``` text
44,100 × 40 = 1,764,000 samples/channel
```

The notes immediately told us where to look:

-   the right leg was mostly a reference tone
-   the left leg contained the actual work
-   the signal was spread over the whole recording
-   inspecting a tiny section would not be sufficient
-   the short `linetest.wav` was provided to determine the extraction
    technique

This is a classic spread-spectrum-style construction: the message is
intentionally kept below the surrounding noise and recovered by
accumulating/correlating many samples.

## Why the short file mattered

Trying to attack the 40-second file directly produced mostly noise.

The line test was only 16 seconds and was explicitly known to decode to:

``` text
NULL0RIGIN LINE TEST 1 OF 1
```

This gave us a known plaintext signal with which to tune the extraction
process.

We inspected:

-   waveform
-   spectrogram
-   both stereo channels
-   low-pass filtered/despread results

Several candidate integration/filter windows were tested.

The useful result appeared after applying the appropriate despreading
and low-pass processing; our analysis visualizations included candidates
such as:

``` text
despread_5
despread_10
despread_20
despread_50
despread_100
despread_500
```

The 100 Hz low-pass/despread view gave a clean separation between the
recovered signal and the noise floor.

## Long recording

Once the short recording established the method, the same processing was
applied to `undertone.wav`.

The Stage 33 flag supplied the key:

``` text
Null0rigin{a_carrier_laid_across_the_whole_room}
```

The key was transformed using the operator card's SHA-256 schedule.

The recovered bitstream was then reconstructed into the works-entry
representation.

After validation, the message was:

``` text
Null0rigin{the_order_of_the_colours_is_the_message}
```

## Result

``` text
Null0rigin{the_order_of_the_colours_is_the_message}
```

This became the key for the GIF stage.

------------------------------------------------------------------------

# 7. Stage 35 --- pentimento

## Carrier

``` text
pentimento.gif
dish-notes.txt
```

The GIF contained eleven exposures of what visually appeared to be the
same developing plate.

The notes contained the key observation:

> What is not the same from frame to frame is the TRAY.

The actual image content was intentionally kept visually equivalent.

The hidden information was in the color palette.

## The palette trick

A GIF does not necessarily store RGB values directly for every pixel.

Pixels can contain palette indices:

``` text
pixel → palette[index] → RGB
```

This means that the palette can be rearranged while keeping the rendered
image identical, provided the pixel indices are rearranged consistently.

The challenge exploited exactly this property.

The colorman had:

``` text
256 distinct colours
```

and worked:

``` text
two to a slot
```

meaning the colors were considered as 128 pairs.

The canonical order was described as:

``` text
darkest → lightest
red before green before blue
```

but the challenge deliberately changed which member of each pair
appeared first.

## Extracting the hidden bits

For every frame:

1.  Read its 256-color palette directly.
2.  Construct the canonical ordering.
3.  Divide the colors into 128 expected pairs.
4.  For every pair, determine whether the pair appears in normal or
    swapped order.
5.  Record that choice as one bit.

The image itself therefore remained unchanged, but the palette ordering
carried information.

The eleven frames supplied multiple blocks of hidden choice bits.

The notes also warned not to "tidy" the palette. Reordering the palette
into a conventional form would destroy the evidence.

## Keying

The previous flag was:

``` text
Null0rigin{the_order_of_the_colours_is_the_message}
```

This was transformed according to the card's SHA-256 stream rule.

The palette-choice bitstream was then reconstructed and decoded.

The resulting flag was:

``` text
Null0rigin{keep_the_grain_and_burn_the_chaff}
```

## Result

``` text
Null0rigin{keep_the_grain_and_burn_the_chaff}
```

This became the key for the most troublesome stage of the chain.

------------------------------------------------------------------------

# 8. Stage 36 --- winnow

## Carrier

``` text
sheet.png
contact-sheet.txt
```

The sheet contained:

``` text
64 columns
32 rows
2048 frames
```

The notes explained that frames were paired:

``` text
0 / 1
2 / 3
4 / 5
...
```

Each pair contained opposite bits:

``` text
truth
chaff
```

The difficult part was determining which frame was the truth.

## The frame seal

Every frame carried a two-byte seal.

The notes defined it as:

``` text
SHA-256(
    k,
    frame number as two octets, high first,
    one octet containing the frame bit
)
```

Only the first two bytes of the SHA-256 digest were stored.

Therefore, for each candidate frame we calculated:

``` python
digest = SHA256(
    key +
    frame_number.to_bytes(2, "big") +
    bytes([bit])
)

seal = digest[:2]
```

For each pair:

``` text
frame A → calculate seal
frame B → calculate seal
```

Exactly one candidate matched its embedded seal.

We retained that frame's bit and discarded the other.

After processing all 1024 pairs, we had:

``` text
1024 bits
```

which equals:

``` text
128 bytes
```

## The major complication

The carrier supplied to us did not match the official SHA-256 manifest.

The expected hash was:

``` text
ab61266239b3c574ace2bef3ef395276225a20c1b22f81baac3f9b250825ddad
```

but the materialized working copy had a different hash.

This was a critical warning that the carrier had been altered.

We therefore stopped treating the modified copy as authoritative and
returned to the untouched original archive.

This was one of the most important practical lessons of the entire
chain:

> Never perform steganographic extraction on a re-saved or transformed
> carrier when the challenge explicitly supplies hashes.

## Final extraction

Using the intact carrier and the Stage 35 key:

``` text
Null0rigin{keep_the_grain_and_burn_the_chaff}
```

we validated the two-byte seal of every candidate frame.

The resulting truth stream decoded to:

``` text
Null0rigin{nothing_takes_until_it_is_fixed}
```

## Result

``` text
Null0rigin{nothing_takes_until_it_is_fixed}
```

This was the key for Stage 37.

------------------------------------------------------------------------

# 9. Stage 37 --- mordant

## Carrier

``` text
mordant.png
mordant-notes.txt
```

This stage combined two independent pieces of information.

The notes said:

``` text
there are five ways to lay a line
```

and:

``` text
Four of the five ways say something.
The fifth says the same as the first.
```

They also said there were exactly six mordants, obtained from the extra
eight-octet fields that followed the separator in previous works
entries.

## Recovering the six mordants

The six values, in the exact order in which they were obtained, were:

``` text
9e4c1af0d3b27651
2a77be05c4198d3f
f10983dd6b5ec4a2
55c2e79140ab3d08
bd3f60a8e2749c15
07e5cb3219d6f48b
```

Each is eight octets.

Concatenating them gives:

``` text
M =
9e4c1af0d3b276512a77be05c4198d3ff10983dd6b5ec4a255c2e79140ab3d08bd3f60a8e2749c1507e5cb3219d6f48b
```

The crucial detail we initially missed was that these six values were
not themselves a repeating XOR key.

The notes said:

``` text
The card's key opens the page.
The six open what is on it.
Neither one alone opens anything.
```

That means the six mordants had to become a second cryptographic key.

## Extracting the five line choices from the PNG

The image was an 8-bit grayscale PNG:

``` text
1400 × 1900
```

The raw PNG scanline stream contains one filter byte at the beginning of
every scanline.

There were exactly:

``` text
1900 scanlines
```

and the filter counts were:

``` text
filter 0 → 64
filter 1 → 456
filter 2 → 504
filter 3 → 453
filter 4 → 423
```

This was the hidden five-symbol alphabet.

The notes revealed that filters 0 and 4 represented the same symbol:

``` text
0 → symbol 0
4 → symbol 0
1 → symbol 1
2 → symbol 2
3 → symbol 3
```

Therefore the 1900 filter values became 1900 base-4 symbols.

## Packing the base-4 symbols

Each symbol carries two bits:

``` text
0 → 00
1 → 01
2 → 10
3 → 11
```

Thus:

``` text
1900 symbols × 2 bits = 3800 bits
```

or:

``` text
475 bytes
```

The raw page was therefore a 475-byte encoded object.

## First cryptographic layer

The Stage 36 flag was:

``` text
Null0rigin{nothing_takes_until_it_is_fixed}
```

We derived:

``` python
k6 = SHA256(stage6_flag)
```

and expanded it using:

``` python
SHA256(k6 || counter_be32)
```

The resulting stream was XORed against the 475-byte base-4 page.

This produced a partially meaningful page, but not the final answer.

## The step we initially missed

Our first attempt treated the six mordants as a direct repeating XOR
sequence.

That produced garbage.

The wording:

``` text
The six open what is on it.
```

was the important clue.

The correct operation was:

``` python
kM = SHA256(mordant_bytes)
```

where `mordant_bytes` is the 48-byte concatenation of the six values.

Then:

``` python
SM = SHA256(kM || counter_be32)
```

was expanded into a second stream.

The complete pipeline was therefore:

``` text
PNG filter types
      ↓
base-4 symbols
      ↓
475-byte packed page
      ↓
XOR with Stage-6 key stream
      ↓
XOR with SHA256(concatenated six mordants) stream
      ↓
works entry
```

## Recovered works entry

The resulting bytes were:

``` text
PL8 01 2f 00 30 2d eb 29
Null0rigin{he_set_two_sorts_where_one_would_do}
```

Parsed:

``` text
stamp  = PL8
issue  = 01
length = 0x002f = 47
check  = 30 2d eb 29
```

The check is little-endian:

``` text
0x29eb2d30
```

and:

``` python
import zlib

zlib.crc32(
    b"Null0rigin{he_set_two_sorts_where_one_would_do}"
) & 0xffffffff
```

returned:

``` text
0x29eb2d30
```

The CRC therefore matched exactly.

This was the strongest validation in the entire chain.

## Result

``` text
Null0rigin{he_set_two_sorts_where_one_would_do}
```

This became the final key for Stage 38.

------------------------------------------------------------------------

# 10. Stage 38 --- quietus

## Carrier

``` text
quietus.png
press-notes.txt
```

The archive was deliberately kept as a ZIP so that the original PNG
could be extracted without accidental image conversion.

We extracted it directly:

``` bash
unzip -l quietus.zip
unzip quietus.zip
```

The untouched carrier hash matched the supplied manifest:

``` text
78e18d4e14439a6a6fb5d6bce83d2ac855b19ff51393d5c631c6fb8b4ae687b6
```

## The obvious answer was a trap

Inspecting metadata/strings exposed:

``` text
Null0rigin{the_compositor_went_home_at_six}
```

It was tempting to submit immediately.

The press notes explicitly warned against this type of mistake.

The notes described a seventh share and said it was not part of the
construction.

Therefore the obvious embedded string was treated as a decoy.

## Reading the forme

The notes explained the actual mechanism through compositor terminology.

A compositor can obtain a sort in two ways:

``` text
1. reach back and reuse an already-set sort
2. take a fresh sort from the case
```

Both result in identical visible text.

The important information is therefore not the final text but the
compositor's choice.

The notes further constrained the process:

``` text
THREE CHARACTERS MAKE A LIGATURE.
```

and:

``` text
he always reached for the nearest one.
```

Thus, whenever the next three-character sequence already existed in the
active stick, the compositor had a choice:

``` text
reuse the nearest previous occurrence
```

or:

``` text
set a fresh copy
```

That binary choice is the hidden channel.

## Recovering the actual message

We therefore ignored the obvious metadata share and analyzed the actual
image/forme layer.

The image was extremely low contrast. Direct viewing made the hidden
text almost invisible.

The effective workflow was:

``` text
original quietus.png
        ↓
inspect raw pixel/luminance values
        ↓
identify low-contrast secondary layer
        ↓
increase contrast / threshold the relevant layer
        ↓
recover the hidden message
```

The resulting message was:

``` text
Null0rigin{the_channel_closes_from_the_inside}
```

The compositor narrative and the recovered text were consistent with the
stage's final construction: the visible page was not the message; the
hidden information was in the choices made while creating it.

## Result

``` text
Null0rigin{the_channel_closes_from_the_inside}
```

That completed the entire STEGO CHAIN.

------------------------------------------------------------------------

# 11. What Made the Chain Difficult

The challenge was not difficult because any single primitive was
exceptionally advanced.

It was difficult because each stage changed the representation of the
hidden information.

Across the eight stages we encountered:

``` text
Stage 31  → pixel bit plane + PNG structure
Stage 32  → paired images + key-derived bit-plane selection
Stage 33  → halftone cell arrangements
Stage 34  → spread-spectrum audio
Stage 35  → GIF palette ordering
Stage 36  → per-frame cryptographic seals
Stage 37  → PNG filter types + two-stage cryptographic decoding
Stage 38  → compositor/forme choices + low-contrast carrier
```

The challenge continuously forced us to stop asking:

> "Where is the text?"

and instead ask:

> "What property of the carrier could have been changed without changing
> what a normal observer sees?"

That was the central design principle behind the chain.

------------------------------------------------------------------------

# 12. Team Struggles and Lessons Learned

## 12.1 We initially trusted readable strings

Several stages deliberately exposed strings that looked like flags.

Examples included:

``` text
NullOrigin{the_tide_returns_what_it_took}
Null0rigin{recovered_from_the_shingle}
Null0rigin{the_screen_was_turned_fifteen_degrees}
Null0rigin{eleven_frames_and_none_of_them_true}
Null0rigin{the_compositor_went_home_at_six}
```

Some came from metadata.

Some came from appended files.

Some were explicitly described by the notes as things that should not be
entered.

The correct approach became:

``` text
candidate text
    ↓
does it obey the stage mechanism?
    ↓
does it validate?
    ↓
only then accept it
```

## 12.2 Carrier integrity became a first-class concern

The Stage 36 carrier was particularly painful.

The supplied manifest expected:

``` text
ab61266239b3c574ace2bef3ef395276225a20c1b22f81baac3f9b250825ddad
```

but a working copy had a different hash.

A modified image can still look perfectly normal while destroying
steganographic information.

The correct operational procedure is therefore:

``` bash
sha256sum carrier
```

before doing anything destructive.

For fragile formats:

``` text
original/
    untouched carrier

work/
    analysis copies
```

The original should never be opened in software that may silently
rewrite it.

## 12.3 Stage 37 taught us to read the cryptographic wording literally

The biggest conceptual mistake in Stage 37 was treating the six mordants
as a direct XOR key.

The notes were more precise than that.

The correct interpretation was:

``` text
Stage 6 flag
      ↓ SHA-256
key stream #1

six mordants
      ↓ concatenate
48 bytes
      ↓ SHA-256
key stream #2
```

The second SHA-256 operation was the missing step.

The final CRC validation immediately confirmed the corrected pipeline.

## 12.4 Known plaintext was extremely useful

Stage 34 provided a line test with a known result:

``` text
NULL0RIGIN LINE TEST 1 OF 1
```

This is exactly the kind of known plaintext that should be exploited
when reverse-engineering a signal-processing scheme.

Instead of guessing the long recording's parameters blindly, we used the
short carrier to identify the correct despreading/filtering behavior
first.

## 12.5 The notes were not flavor text

The challenge documents were effectively the protocol specification.

Examples:

``` text
FOUR TO THE CELL
```

gave the Stage 33 cell size.

``` text
The machine lays the cells as the ox ploughs.
```

gave the serpentine traversal.

``` text
They were shot IN PAIRS.
```

gave the Stage 36 pair structure.

``` text
Every frame carries its own SEAL.
```

gave the validation mechanism.

``` text
A mordant is not one thing. Ours is SIX.
```

gave the number of secondary key components.

``` text
THREE CHARACTERS MAKE A LIGATURE.
```

gave the Stage 38 unit of comparison.

The recurring lesson was:

> Treat narrative wording as technical documentation.

------------------------------------------------------------------------

# 13. Reusable Cryptographic Helpers

The following helper implements the challenge's key schedule:

``` python
import hashlib

def derive_key(previous_flag: str) -> bytes:
    return hashlib.sha256(
        previous_flag.encode("ascii")
    ).digest()

def derive_stream(key: bytes, length: int) -> bytes:
    out = bytearray()

    counter = 0
    while len(out) < length:
        out.extend(
            hashlib.sha256(
                key + counter.to_bytes(4, "big")
            ).digest()
        )
        counter += 1

    return bytes(out[:length])

def xor_bytes(data: bytes, stream: bytes) -> bytes:
    return bytes(a ^ b for a, b in zip(data, stream))
```

For a normal stream-based stage:

``` python
key = derive_key(previous_flag)
stream = derive_stream(key, len(ciphertext))
plaintext = xor_bytes(ciphertext, stream)
```

For a number-based stage:

``` python
key = derive_key(previous_flag)
number = int.from_bytes(key[:4], "big")
```

------------------------------------------------------------------------

# 14. Important Extraction Patterns

## LSB extraction

``` python
bits = pixels & 1

data = bytearray()

for i in range(0, len(bits) - 7, 8):
    byte = 0
    for b in bits[i:i+8]:
        byte = (byte << 1) | int(b)
    data.append(byte)
```

## PNG scanline filter extraction

For an 8-bit grayscale PNG:

``` text
each scanline =
    1 filter byte
    + width pixel bytes
```

Therefore:

``` python
filters = raw_scanline_data[0::width + 1]
```

can expose the filter-choice channel directly.

## 4×4 halftone analysis

``` python
for y in range(0, height, 4):
    for x in range(0, width, 4):
        cell = image[y:y+4, x:x+4]
        weight = cell.sum()
```

The number of distinct cell patterns for each weight reveals whether
that weight carries a binary choice.

## GIF palette analysis

Do not render the GIF and re-save it.

Instead:

``` text
read original GIF
    ↓
extract frame palettes
    ↓
compare palette ordering
    ↓
recover pair swaps
```

The rendered image can remain unchanged while the palette itself carries
the message.

------------------------------------------------------------------------

# 15. Final Verification Table

The completed chain was:

``` text
31-driftwood
    ↓
Null0rigin{pit_head_plate_number_four}

32-safelight
    ↓
Null0rigin{what_the_print_kept_from_the_negative}

33-moire
    ↓
Null0rigin{a_carrier_laid_across_the_whole_room}

34-undertone
    ↓
Null0rigin{the_order_of_the_colours_is_the_message}

35-pentimento
    ↓
Null0rigin{keep_the_grain_and_burn_the_chaff}

36-winnow
    ↓
Null0rigin{nothing_takes_until_it_is_fixed}

37-mordant
    ↓
Null0rigin{he_set_two_sorts_where_one_would_do}

38-quietus
    ↓
Null0rigin{the_channel_closes_from_the_inside}
```

The strongest independent validation occurred at Stage 37, where the
recovered works entry had a CRC/check value of:

``` text
0x29eb2d30
```

and the calculated CRC32 of the recovered payload was exactly:

``` text
0x29eb2d30
```

This eliminated ambiguity about the final Stage 37 payload.

------------------------------------------------------------------------

# 16. Final Takeaways

The STEGO CHAIN was ultimately a lesson in information surviving
transformation.

The hidden messages were not always encoded as obvious bits. Instead,
the challenge repeatedly used properties that ordinary viewers ignored:

``` text
pixels
metadata
bit planes
image differences
halftone geometry
audio correlation
palette order
frame authenticity
PNG filter bytes
cryptographic seals
low-level compositor choices
```

The most important workflow for solving the chain was:

1.  Read the accompanying document completely.
2.  Hash the previous flag exactly.
3.  Verify the carrier hash.
4.  Preserve the original.
5.  Identify what can change without visibly changing the carrier.
6.  Extract the hidden choice rather than searching only for text.
7.  Use the card's cryptographic rule literally.
8.  Validate the result using the works-entry stamp/check whenever
    possible.
9.  Never accept an attractive string without independent validation.

The final flag of the chain was:

``` text
Null0rigin{the_channel_closes_from_the_inside}
```

------------------------------------------------------------------------

## Source/Artifact Notes

The write-up was reconstructed from the challenge's supplied README,
operator card, stage notes, original carriers, SHA-256 manifest, and the
analysis performed during the solve.

Important original-carrier hashes included:

``` text
31-driftwood/driftwood.png
87bebfa171b724cdf5ab3243fe480a18841a4b849e34b06e7a2b755ae01518e4

32-safelight/negative.png
804f5f096d7545bc436e51ecb69c5e0c47ff7d361139e72c23546664837909a7

32-safelight/print.png
614c41b0e2bdb81cab8f27eedb65899771746855b3fa214f629cdfe7fe63dbd2

33-moire/moire.png
1ce87c251ecae1b1df37b091c328e1b551635d389527edc42fff109868ec7c9e

34-undertone/undertone.wav
571fb8f324849ac7e1a50dcbd6b71f581437aa60b9a4b34b6c47f31cd70e1b12

35-pentimento/pentimento.gif
5b820e63c4b440b82d25ccdcb62fe5a65e0b2fb2e47d603655701968ccd02939

36-winnow/sheet.png
ab61266239b3c574ace2bef3ef395276225a20c1b22f81baac3f9b250825ddad

37-mordant/mordant.png
f67a45518dbb0f222b594b4414f451ddf1650b2948913a42aaa92cebd8a618db

38-quietus/quietus.png
78e18d4e14439a6a6fb5d6bce83d2ac855b19ff51393d5c631c6fb8b4ae687b6
```

The exact hashes matter because several of these carriers are
deliberately sensitive to re-saving or re-encoding.
