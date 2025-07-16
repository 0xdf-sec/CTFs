# picoCTF: Sleuthkit Apprentice - Writeup

## Challenge Description

> Download this disk image and find the flag.  
> [disk.flag.img.gz](https://artifacts.picoctf.net/c/136/disk.flag.img.gz)

---

## Solution Steps

### 1. Extract the Disk Image

```bash
gunzip disk.flag.img.gz
```

### 2. Check Disk Image Info

```bash
file disk.flag.img
```
**Output:**  
DOS/MBR boot sector; multiple partitions, including Linux (0x83) and Linux Swap.

### 3. List Disk Partitions

```bash
mmls disk.flag.img
```
**Output:**
```
Slot      Start        End          Length       Description
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000206847   0000204800   Linux (0x83)
003:  000:001   0000206848   0000360447   0000153600   Linux Swap / Solaris x86 (0x82)
004:  000:002   0000360448   0000614399   0000253952   Linux (0x83)
```

### 4. Search for Flag File in Partitions

- **First Linux partition** (offset 2048): No flag file.
- **Second Linux partition** (offset 360448): Search for flag files.

```bash
fls -o 360448 -r disk.flag.img | grep flag
```
**Output:**
```
++ r/r * 2082(realloc): flag.txt
++ r/r 2371:    flag.uni.txt
```
_Found `flag.uni.txt` with inode 2371._

### 5. Extract Flag

```bash
icat -o 360448 disk.flag.img 2371
```
**Output:**
```
picoCTF{by73_5urf3r_3497ae6b}
```

---

## Flag

```
picoCTF{by73_5urf3r_3497ae6b}
```

---

## Summary

- Use `gunzip` to extract the disk image.
- Use `mmls` to find partition offsets.
- Use `fls` to search for flag files in each partition.
- Use `icat` to extract the flag content.

**Flag:**  
`picoCTF{by73_5urf3r_3497ae6b}`
