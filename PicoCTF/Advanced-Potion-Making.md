# picoCTF: Advanced Potion Making - Writeup

## Challenge Description

Ron just found his own copy of advanced potion making, but it’s been corrupted by some kind of spell. Help him recover it!

**File Provided:**  
[advanced-potion-making](https://artifacts.picoctf.net/picoMini+by+redpwn/Forensics/advanced-potion-making/advanced-potion-making)

---

## Steps to Solution

### 1. Initial Analysis

```bash
file advanced-potion-making
```
**Output:**  
`advanced-potion-making: data`

No strings or readable content found.

### 2. Hexdump and File Signature

```bash
hexdump -C advanced-potion-making | head
```

**First bytes:**  
`89 50 42 11 0d 0a 1a 0a 00 12 13 14 49 48 44 52 ...`

Compare with [file signature list](https://en.wikipedia.org/wiki/List_of_file_signatures):

- PNG files start with: `89 50 4E 47 0D 0A 1A 0A`  
- This file starts with: `89 50 42 11 ...` (so, PNG but with `42 11` instead of `4E 47`).

### 3. Repairing the Header

Open in **hexeditor** and change the bytes at offsets 2 and 3:

- Change `42 11` → `4E 47`

So, the first eight bytes become:  
`89 50 4E 47 0D 0A 1A 0A`

Save as `fixed.png`.

### 4. Viewing the Image

Open `fixed.png`. It shows a solid red image.

### 5. Image Manipulation

- Tried grayscale, invert, and other filters.
- **Black and White filter revealed the flag.**

### 6. Solution

**Flag revealed:**  
`picoCTF{w1z4rdry}`

---

## Summary

- The corruption was in the PNG magic bytes.
- Repairing the header made the file openable as a PNG.
- The flag was hidden in the image, only visible after making it black and white.

**Flag:**  
``picoCTF{w1z4rdry}``

## References

- [File signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)
