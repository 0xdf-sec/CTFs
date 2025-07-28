**Category:** Forensics  
**Difficulty:** Hard  
**Files Provided:** 10 files (HTML, TXT, PDF, CSV, PNG images)

## Initial File Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ ls -la
total 32,456
drwxr-xr-x 2 0xdf-sec 0xdf-sec     4096 Jul 28 23:16 .
drwxr-xr-x 3 0xdf-sec 0xdf-sec     4096 Jul 28 23:16 ..
-rw-r--r-- 1 0xdf-sec 0xdf-sec    14934 Jul 26 14:06 chatterpiller_donoteat.html
-rw-r--r-- 1 0xdf-sec 0xdf-sec     1549 Jul 26 14:05 Personal_Research_Notes_-_A._Liddell.txt
-rw-r--r-- 1 0xdf-sec 0xdf-sec   291585 Jul 26 14:06 PWNDERLAND_HEALTH_AUTHORITY.pdf
-rw-r--r-- 1 0xdf-sec 0xdf-sec     6062 Jul 26 14:06 PWNDERLAND_MYCLOGICAL_LAB_ANALYSIS.csv
-rw-r--r-- 1 0xdf-sec 0xdf-sec  6130816 Jul 26 14:06 specimen_001.png
-rw-r--r-- 1 0xdf-sec 0xdf-sec  6893781 Jul 26 14:06 specimen_002.png
-rw-r--r-- 1 0xdf-sec 0xdf-sec  5691415 Jul 26 14:06 specimen_003.png
-rw-r--r-- 1 0xdf-sec 0xdf-sec  6403226 Jul 26 14:06 specimen_004.png
-rw-r--r-- 1 0xdf-sec 0xdf-sec  5615346 Jul 26 14:06 specimen_unknown.png
-rw-r--r-- 1 0xdf-sec 0xdf-sec   358275 Jul 26 14:06 THE_PWNDERLAND_MYCOLOGICAL_FIELD_GUIDE.pdf
```

## File Type Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ file *
chatterpiller_donoteat.html:                    HTML document, ASCII text, with very long lines (65536)
Personal_Research_Notes_-_A._Liddell.txt:       ASCII text
PWNDERLAND_HEALTH_AUTHORITY.pdf:                PDF document, version 1.4
PWNDERLAND_MYCLOGICAL_LAB_ANALYSIS.csv:         CSV text
specimen_001.png:                               PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced
specimen_002.png:                               PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced
specimen_003.png:                               PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced
specimen_004.png:                               PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced
specimen_unknown.png:                           PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced
THE_PWNDERLAND_MYCOLOGICAL_FIELD_GUIDE.pdf:     PDF document, version 1.4
```

## Metadata Analysis with ExifTool

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ exiftool *.png
======== specimen_001.png ========
ExifTool Version Number         : 12.40
File Name                       : specimen_001.png
Directory                       : .
File Size                       : 5.8 MiB
File Modification Date/Time     : 2025:07:26 14:06:00-04:00
File Access Date/Time           : 2025:07:28 23:16:55-04:00
File Permissions                : rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1920
Image Height                    : 1080
Bit Depth                       : 8
Color Type                      : RGB with Alpha
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Pixels Per Unit X               : 2835
Pixels Per Unit Y               : 2835
Pixel Units                     : meters

[Similar metadata for other PNG files - no hidden data found]
```

## Binwalk Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ binwalk *.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ binwalk -e *.png
[No embedded files found]
```

## Strings Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ strings specimen_001.png | grep -i flag
[No results]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ strings *.pdf | grep -i flag
[No results]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ strings *.txt | grep -i flag
[No results]
```

## Steganography Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ steghide info specimen_001.png
steghide: the file format of the file "specimen_001.png" is not supported.

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ zsteg specimen_001.png
[No hidden data found]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ stegsolve specimen_001.png
[Analyzed through various bit planes - no visible hidden data]
```

## PDF Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ pdfinfo PWNDERLAND_HEALTH_AUTHORITY.pdf
Title:          Pwnderland Health Authority Report
Creator:        Adobe Acrobat Pro
Producer:       Adobe Acrobat Pro
CreationDate:   Fri Jul 26 14:05:32 2025
ModDate:        Fri Jul 26 14:05:32 2025
Tagged:         no
UserProperties: no
Suspects:       no
Form:           none
JavaScript:     no
Pages:          12
Encrypted:      no
Page size:      612 x 792 pts (letter)
Page rot:       0
File size:      291585 bytes
Optimized:      no
PDF version:    1.4

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ pdfgrep -i "flag" *.pdf
[No results]
```

## CSV Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ head -20 PWNDERLAND_MYCLOGICAL_LAB_ANALYSIS.csv
Specimen_ID,Scientific_Name,Common_Name,Toxicity_Level,Location_Found,Analysis_Date,Notes
001,Amanita phalloides,Death Cap,Extremely High,Wonderland Woods,2025-07-20,Highly toxic - avoid consumption
002,Psilocybe cubensis,Golden Teacher,Low,Mad Hatter's Garden,2025-07-21,Psychoactive properties noted
003,Agaricus bisporus,Button Mushroom,None,Queen's Garden,2025-07-22,Safe for consumption
004,Gyromitra esculenta,False Morel,High,Cheshire Forest,2025-07-23,Contains gyromitrin toxin
[Data continues...]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ grep -i flag PWNDERLAND_MYCLOGICAL_LAB_ANALYSIS.csv
[No results]
```

## HTML File Analysis - The Breakthrough

After exhausting traditional forensics approaches, I decided to examine the HTML file more carefully:

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/chatterpiller]
└─$ firefox chatterpiller_donoteat.html
```

Opening the HTML file revealed a chat conversation interface. The conversation appeared to be between researchers discussing mycological specimens and safety protocols.

## Key Discovery

While reading through the chat conversation, I noticed one particular message that stood out:

- Most messages had standard formatting with white/gray backgrounds
- One specific reply was **highlighted in red** and contained an unusual emoji sequence
- The red highlighting and emoji placement seemed intentional and out of place

The suspicious message contained what appeared to be a decorative emoji sequence, but given the forensics context, I suspected it might be encoded data.

## Emoji Decoding

I copied the emoji sequence from the red-highlighted message:

```
💀󠄱󠄼󠄡󠄳󠄣󠄳󠅄󠄶󠅫󠅔󠄣󠄤󠅤󠅘󠅏󠄤󠅞󠅔󠅏󠅢󠄣󠅒󠄡󠅢󠅤󠅘󠅏󠄡󠅞󠅏󠅠󠅧󠅞󠅔󠄣󠅢󠅜󠄤󠅞󠅔󠅭
```

Using an online emoji decoder/cipher tool, I decoded the emoji sequence:
https://emoji-encoder.vercel.app/?mode=decode

The decoded result revealed the flag:

**FLAG:** `AL1C3CTF{d34th_4nd_r3b1rth_1n_pwnd3rl4nd}`
