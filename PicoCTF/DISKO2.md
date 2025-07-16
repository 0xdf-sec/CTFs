# picoCTF: DISKO 2 - Writeup

## Challenge Description

> Can you find the flag in this disk image? The right one is Linux! One wrong step and its all gone!  
> Disk image: [disko-2.dd.gz](https://artifacts.picoctf.net/c/541/disko-2.dd.gz)

---

## Solution Steps

### 1. Extract the Disk Image

```bash
gunzip disko-2.dd.gz
```

### 2. Check Disk Image Info

```bash
file disko-2.dd
```
**Output:**  
`DOS/MBR boot sector; partition 1 : ID=0x83 ...`

### 3. Identify Partitions

```bash
mmls disko-2.dd
```
**Output:**
```
Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000053247   0000051200   Linux (0x83)
003:  000:001   0000053248   0000118783   0000065536   Win95 FAT32 (0x0b)
004:  -------   0000118784   0000204799   0000086016   Unallocated
```
_Two main partitions: Linux (startsector 2048), FAT32 (startsector 53248)._

### 4. Analyze with Autopsy

- Open Autopsy, create a new case.
- Add `disko-2.dd` as data source.
- Select the **Linux partition** for analysis.
- Use keyword search (`picoCTF`) to find the flag.

### 5. Flag Found

```
picoCTF{4_P4Rt_1t_i5_055dd175}
```

---

## Summary

- Use `gunzip` to extract the disk image.
- Use `file` and `mmls` to identify partitions.
- Analyze Linux partition with Autopsy and search for the flag.

**Flag:**  
`picoCTF{4_P4Rt_1t_i5_055dd175}`
