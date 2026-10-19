# Restoring Japanese mojibake (文字化け) in the browser

Notes on why Japanese text turns into things like `縺薙ｓ縺ｫ縺｡縺ｯ` or
`æ—¥æœ¬èªž`, and a minimal recipe for restoring it with the browser's
built-in `TextDecoder`.

A working implementation of everything described here runs at
[henkanya.com](https://henkanya.com/) — free, no sign-up, and nothing
you paste is uploaded anywhere.

## Mojibake is a decoding mistake, not corruption

In almost every common case the bytes are intact. The text was simply
**decoded with the wrong character set**: a UTF-8 file opened as Shift_JIS
re-reads every 3-byte Japanese character as 1.5 garbage characters, which
is exactly where `縺薙ｓ縺ｫ縺｡縺ｯ` (= こんにちは) comes from.

So restoration is a search problem: recover the raw bytes, then try
reading them back with every plausible charset, and keep the readings
that look like natural Japanese.

## The (saved-as → read-as) pairs worth trying

In practice, nine pairings cover the overwhelming majority of Japanese
mojibake — who saved the text in which encoding, and what wrongly read it:

| saved as | misread as | typical scene |
| --- | --- | --- |
| UTF-8 | Shift_JIS | Excel / Windows software opening a UTF-8 CSV |
| UTF-8 | Windows-1252 | non-Japanese-aware software or services |
| UTF-8 | EUC-JP | old Unix environments, EUC-JP web pages |
| Shift_JIS | EUC-JP | Shift_JIS files shown on EUC-JP systems |
| EUC-JP | Shift_JIS | old EUC-JP pages/mail viewed on Windows |
| Shift_JIS | Windows-1252 | Western software opening Shift_JIS |
| EUC-JP | Windows-1252 | same, for EUC-JP |
| UTF-16LE | Windows-1252 | UTF-16 files read as 1-byte text |
| UTF-16BE | Windows-1252 | same, byte-swapped |

## A minimal TextDecoder recipe

When the text was misread through a **single-byte** charset
(Windows-1252 / Latin-1), the original bytes are recoverable by just
taking each character's low byte, then re-decoding:

```ts
const decoders = [
  "utf-8", "shift_jis", "euc-jp", "utf-16le", "utf-16be", "windows-1252",
].map((label) => new TextDecoder(label));

function candidates(mojibake: string): string[] {
  const bytes = Uint8Array.from([...mojibake], (ch) => ch.charCodeAt(0) & 0xff);
  return decoders
    .map((d) => { try { return d.decode(bytes); } catch { return ""; } })
    .filter((s) => s.length > 0);
}
```

The trickier direction is **Japanese-charset → Japanese-charset** (e.g.
saved as Shift_JIS, misread as EUC-JP): `TextEncoder` only ever produces
UTF-8, so you cannot re-encode to Shift_JIS in the browser. The workaround
is to build a byte → string lookup table once, by decoding every possible
byte sequence with `TextDecoder`, and re-using it to map characters back
to bytes.

## Picking the right candidate: not all garbage is equal

Brute force returns several readings that "work". Ranking them by
**how natural the Japanese looks** puts the real answer on top: score
candidates by how often their kanji appear in everyday Japanese, treating
JIS level-1 kanji as a positive signal and level-2 kanji (the kind that
show up in wrong readings, like 縺・繧・繝) as a negative one.

## The trailing 縺・ case

Mail software used to truncate messages at fixed byte lengths — sometimes
mid-character. The result is mojibake that ends in a lone `縺・`: the
first byte of a multi-byte character survived, the rest was cut. Those
bytes can be re-completed by trying the byte sequences that continue the
trailing fragment, and picking the character that best fits the context.

## Double-garbled text

Text that was garbled, then saved and garbled again, needs the whole
pair search run a second time on the first round's output. It's rare in
files, common in copy-pasted sentences.
