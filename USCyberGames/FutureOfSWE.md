# CTF: Future of SWE - Writeup

## Overview

"Future of SWE" is a digital forensics challenge that simulates a data recovery scenario where an AI assistant (ClippyGPT) has deleted important documents from a company's system. The challenge involves analyzing a disk image, recovering deleted files, and using various forensic techniques to extract passwords and decrypt protected documents.

## Challenge Information

| Attribute | Value |
|-----------|-------|
| **Name** | Future of SWE |
| **Category** | Digital Forensics |
| **Points** | TBD |
| **Platform** | US Cyber Games CTF |
| **Difficulty** | Medium |

## Challenge Description

*"Our founder is convinced that ChatGPT will replace all software engineers in the future. Perhaps related, but it appears the ClippyGPT SaaS he's signed up for has deleted his documents. Can you help recover them?"*

**Provided File:** `future_of_swe.raw.001.zip`

## Analysis and Solution

### 1. Initial File Analysis

```bash
# Extract the provided file
unzip future_of_swe.raw.001.zip

# Analyze file type
file future_of_swe.raw.001
```

**Output:**
```
future_of_swe.raw.001: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "MSDOS5.0", 
sectors/cluster 4, reserved sectors 8, root entries 512, Media descriptor 0xf8, 
sectors/FAT 256, sectors/track 63, heads 255, hidden sectors 12390400, 
sectors 262144 (volumes > 32 MB), serial number 0xfa653f79, unlabeled, FAT (16 bit)
```

**Key Finding:** FAT16 filesystem disk image

### 2. File Recovery with PhotoRec

```bash
# Recover deleted files using PhotoRec
photorec future_of_swe.raw.001
```

**Recovered Files:**
- `f0000560.txt` - Staff meeting notes
- `f0000564_untitled.pdf` - Document file
- `f0000680.xlsx` - Password spreadsheet
- `f0000700.doc` - Password-protected document
- `report.xml` - Recovery report

### 3. Initial Clue Analysis

```bash
cat f0000560.txt
```

**Staff Meeting Notes:**
```
=== 18-May-2025 Staff Sync ===
- Agenda: ClippyGPT post-deployment triage
- Action: Investigate why ClippyGPT replaced CEO password with "password123"
- Reminder: ProjectNextBigThing.docx was "encrypted" during cleanup.
  Password stored in passwords.xlsx, Sheet2, row 4.

 (\_/)
 (•‿•)  <-- 'Everything's fine' meme bunny
 /   \

=== 18-May-2025 Ad-hoc ===
- Determine if AI deleting files actually improves productivity...
```

**Key Information:**
- ClippyGPT deleted files during "cleanup"
- Password for encrypted document is in `passwords.xlsx`, Sheet2, row 4

### 4. Password Spreadsheet Analysis

**Initial Passwords Found in f0000680.xlsx:**

| Website | Username | Password |
|---------|----------|----------|
| example.com | alice | h@rdPa$$1! |
| email.org | bob | P@55w0rd! |
| supersecure.net | ceo | CorrectHorseBatteryStaple |

 **Issue:** None of these ~~passwords~~ worked on the protected document `f0000700.doc`

### 5. Disk Image Mounting

```bash
# Mount the disk image to access filesystem
mkdir /mnt/fatsd
mount -o loop,ro future_of_swe.raw.001 /mnt/fatsd
```

### 6. Log File Analysis

```bash
# Search for password-related entries in ClippyGPT logs
cat /mnt/fatsd/clippyGPT_log.txt | grep pass
```

**Critical Log Entries:**
```
[2025-05-18 14:04:32] ClippyGPT 🤖: Deleted ProjectNextBigThing.docx, passwords.xlsx, meeting_notes.txt, AI_vs_Humanity.pdf
[2025-05-18 14:05:34] ClippyGPT 🤖: Metaverse-ported the password policy to unlock true innovation! 🤣😂
[2025-05-18 14:25:31] ClippyGPT 🤖: Deleted the password policy during a live demo 😱! 😅😂
[2025-05-18 14:29:40] ClippyGPT 🤖: Darkweb-listed the password policy with extreme prejudice! 🤣
```

**Key Finding:** ClippyGPT deleted multiple files including the original `passwords.xlsx`

### 7. Deep File Structure Analysis

```bash
# Extract embedded files and analyze internal structure
binwalk -e future_of_swe.raw.001 --run-as=root
```

### 8. Excel File Internal Analysis

```bash
# Navigate to extracted Excel file structure
cd _future_of_swe.raw.001.extracted/55000/xl/

# Analyze shared strings (contains all text data)
cat sharedStrings.xml
```

**Hidden Passwords Discovered:**
```xml
<si><t>clippyisawesome</t></si>
<si><t>1234</t></si>
<si><t>password123</t></si>
<si><t>ilovecoffee</t></si>
<si><t>ClippyGPT Generated Passwords</t></si>
```

### 9. Document Decryption

**Testing Additional Passwords:**
- `clippyisawesome`  **Success!**
- ~~`1234`~~
- ~~`password123`~~
- ~~`ilovecoffee`~~

### 10. Flag Extraction

After successfully opening the password-protected document with `clippyisawesome`:

🚩 **Flag:** `SVUSCG{th3_futur3_is_look1n_br1ght}`

## Attack Summary

```
Disk Image Analysis
    ↓ PhotoRec Recovery
Deleted Files Retrieved
    ↓ Staff Notes Analysis
Password Location Identified
    ↓ Initial Password Attempt Failed
Filesystem Mounting
    ↓ Log Analysis
Deletion Confirmation
    ↓ Binwalk Extraction
Excel Internal Structure
    ↓ SharedStrings.xml Analysis
Hidden Password Discovery
    ↓ Document Decryption
Flag Retrieved
```

## Key Techniques Used

1. **File Type Identification** - Recognizing FAT16 filesystem
2. **Data Recovery** - PhotoRec for deleted file recovery
3. **Filesystem Mounting** - Loop device mounting for direct access
4. **Log Analysis** - Searching for security-relevant events
5. **Binary Extraction** - Binwalk for embedded file analysis
6. **XML Parsing** - Excel internal structure analysis
7. **Password Cracking** - Systematic password testing

## Tools Used

- `file` - File type identification
- `photorec` - Deleted file recovery
- `mount` - Filesystem mounting
- `binwalk` - Binary analysis and extraction
- `grep` - Log pattern searching
- `cat` - File content examination

## Lessons Learned

1. **Multiple Recovery Methods** - Different techniques may reveal different data
2. **Internal File Structure** - Modern file formats contain metadata in XML
3. **AI-Generated Content** - Logs may contain clues about automated actions
4. **Hidden Data** - Deleted files may leave traces in file system structures

**Author:** 0xdf-sec  
**Date:** June 11, 2025  
**Tags:** `ctf` `digital-forensics` `file-recovery` `password-cracking` `excel-analysis` `disk-imaging`
