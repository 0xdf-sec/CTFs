**Category:** Forensics  
**Difficulty:** Hard  
**Files Provided:** alice_diary_page.pdf

## Initial File Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# ls -la
total 68
drwxr-xr-x 2 0xdf-sec 0xdf-sec  4096 Jul 28 23:45 .
drwxr-xr-x 3 0xdf-sec 0xdf-sec  4096 Jul 28 23:45 ..
-rwxr-x--- 1 0xdf-sec 0xdf-sec 65536 Jul 26 13:49 alice_diary_page.pdf

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# file alice_diary_page.pdf
alice_diary_page.pdf: PDF document, version 1.7
```

## PDF Content Analysis

The PDF contained Alice's diary written in German. After translation, it revealed a story about Alice joining a secret society with three tests:

**First Test:** `QWxpY2UgZmllbCBpbiBkYXMgS2FuaW5jaGVubG9jaA==`  
**Second Test:** `UKYOSNEHRZWEGHVRMBVW` (key: PWNDERLAND)  
**Third Test:** `GUVF VF ABG GUR NAFJRE`

## Cryptographic Analysis Attempts

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# echo "QWxpY2UgZmllbCBpbiBkYXMgS2FuaW5jaGVubG9jaA==" | base64 -d
Alice fiel in das Kaninchenloch
```

**First Test - Base64 Decoded:** "Alice fell into the rabbit hole"

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# python3 -c "
import string
cipher = 'UKYOSNEHRZWEGHVRMBVW'
key = 'PWNDERLAND'
result = ''
for i, c in enumerate(cipher):
    if c in string.ascii_uppercase:
        shift = ord(key[i % len(key)]) - ord('A')
        result += chr((ord(c) - ord('A') - shift) % 26 + ord('A'))
    else:
        result += c
print(result)
"
ALICEWASNEVERHEREEVER
```

**Second Test - Vigenère Cipher:** "ALICEWASNEVERHEREEVER"

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# echo "GUVF VF ABG GUR NAFJRE" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
THIS IS NOT THE ANSWER
```

**Third Test - ROT13:** "THIS IS NOT THE ANSWER"

## PDF Metadata Investigation

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# exiftool alice_diary_page.pdf
ExifTool Version Number         : 13.25
File Name                       : alice_diary_page.pdf
Directory                       : .
File Size                       : 65 kB
File Modification Date/Time     : 2025:07:26 13:49:30-04:00
File Access Date/Time           : 2025:07:28 19:36:56-04:00
File Inode Change Date/Time     : 2025:07:26 13:49:30-04:00
File Permissions                : -rwxr-x---
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.7
Linearized                      : No
Warning                         : Invalid xref table
```

The **"Invalid xref table"** warning suggested the PDF might contain embedded data.

## Deep PDF Analysis with Strings

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# strings alice_diary_page.pdf | tail -20
!c6e*
EI3YsR
KN3o
eaaL
.hidden/UT
/pYhux
.hidden/secrets/UT
/pYhux  
.hidden/secrets/manuscripts/UT
/pYhux
.hidden/secrets/manuscripts/the_final_answer.pngUT
/pYhux
```

**Key Discovery:** The strings output revealed hidden directory structure:
- `.hidden/secrets/manuscripts/the_final_answer.png`

This suggested a ZIP archive was embedded within the PDF!

## Binary Analysis with Binwalk

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# binwalk -e alice_diary_page.pdf --run-as=root

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
67            0x43            Zlib compressed data, best compression
2020          0x7E4           Zlib compressed data, best compression
4500          0x1194          Zlib compressed data, best compression
6200          0x1838          Zlib compressed data, best compression
10542         0x292E          Zlib compressed data, best compression
16063         0x3EBF          Zlib compressed data, best compression
23311         0x5B0F          Zlib compressed data, best compression
24511         0x5FBF          Zlib compressed data, best compression
30657         0x77C1          Zlib compressed data, best compression
33929         0x8489          Zlib compressed data, best compression
34376         0x8648          Zlib compressed data, best compression
34874         0x883A          Zlib compressed data, best compression
35445         0x8A75          Zlib compressed data, best compression
35750         0x8BA6          Zlib compressed data, best compression
36250         0x8D9A          Zlib compressed data, best compression
36875         0x900B          Zlib compressed data, best compression
37087         0x90DF          Zlib compressed data, best compression
37345         0x91E1          Zlib compressed data, best compression
37557         0x92B5          Zlib compressed data, best compression
37815         0x93B7          Zlib compressed data, best compression
38027         0x948B          Zlib compressed data, best compression
38284         0x958C          Zlib compressed data, best compression
38496         0x9660          Zlib compressed data, best compression
38619         0x96DB          Zlib compressed data, best compression
41158         0xA0C6          Zlib compressed data, best compression
41409         0xA1C1          Zip archive data, at least v1.0 to extract, name: .hidden/
41475         0xA203          Zip archive data, at least v1.0 to extract, name: .hidden/secrets/
41549         0xA24D          Zip archive data, at least v1.0 to extract, name: .hidden/secrets/manuscripts/
41635         0xA2A3          Zip archive data, at least v2.0 to extract, compressed size: 23240, uncompressed size: 24649, name: .hidden/secrets/manuscripts/the_final_answer.png
```

**Breakthrough:** Binwalk successfully identified and extracted the embedded ZIP archive containing the hidden PNG file!

## Extracted File Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# cd _alice_diary_page.pdf.extracted/

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland/_alice_diary_page.pdf.extracted]
└─# ls -la
total 156
drwxr-xr-x 3 0xdf-sec 0xdf-sec   4096 Jul 28 23:46 .
drwxr-xr-x 3 0xdf-sec 0xdf-sec   4096 Jul 28 23:46 ..
-rw-r--r-- 1 0xdf-sec 0xdf-sec  24649 Jul 28 23:46 A2A3
-rw-r--r-- 1 0xdf-sec 0xdf-sec  24649 Jul 28 23:46 A2A3.zip
drwxr-xr-x 3 0xdf-sec 0xdf-sec   4096 Jul 28 23:46 .hidden
[Additional Zlib extracted files...]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland/_alice_diary_page.pdf.extracted]
└─# cd .hidden/secrets/manuscripts/

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland/_alice_diary_page.pdf.extracted/.hidden/secrets/manuscripts]
└─# ls -la
total 32
drwxr-xr-x 2 0xdf-sec 0xdf-sec  4096 Jul 28 23:46 .
drwxr-xr-x 3 0xdf-sec 0xdf-sec  4096 Jul 28 23:46 ..
-rw-r--r-- 1 0xdf-sec 0xdf-sec 24649 Jul 28 23:46 the_final_answer.png

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland/_alice_diary_page.pdf.extracted/.hidden/secrets/manuscripts]
└─# file the_final_answer.png
the_final_answer.png: PNG image data, 800 x 600, 8-bit/color RGB, non-interlaced
```

## Image Analysis - The Final Cipher

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland/_alice_diary_page.pdf.extracted/.hidden/secrets/manuscripts]
└─# feh the_final_answer.png
```

Opening the image revealed a collection of **mysterious symbols and characters** that didn't match any standard alphabet. The symbols appeared to be from a historical cipher system.

## Historical Cipher Identification

Given the diary's references to:
- **"German masters"** 
- **"ancient documents"**
- **"old masters' art of hiding truths"**
- **18th-century Germanic secret society context**

I suspected this might be the **Copiale Cipher** - a famous 18th-century German secret society cipher that was only decrypted in 2011.

## Copiale Cipher Decryption

Using the dCode Copiale Cipher decoder (https://www.dcode.fr/copiale-cipher):

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/AliceInPwnderland]
└─# firefox https://www.dcode.fr/copiale-cipher
```

**Steps:**
1. Uploaded the `the_final_answer.png` image to dCode
2. Selected "Copiale Cipher" from the cipher list
3. Used automatic symbol recognition
4. Applied the historical Copiale decryption algorithm

**Decrypted Result:** `AL1C3CTF{THECANDIDATEANSWERSYES}`

## Solution Summary

This multi-layered forensics challenge required:

1. **PDF Analysis:** Recognition of corrupted xref table
2. **String Analysis:** Discovery of hidden directory structure  
3. **Binary Extraction:** Using binwalk to extract embedded ZIP archive
4. **Historical Cipher Knowledge:** Identifying the Copiale Cipher from contextual clues
5. **Specialized Decryption:** Using dCode's Copiale Cipher decoder

**FLAG:** `AL1C3CTF{THECANDIDATEANSWERSYES}`
