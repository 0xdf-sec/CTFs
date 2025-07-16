# picoCTF: Disk, disk, sleuth! - Writeup

## Challenge Description

> Use `srch_strings` from the sleuthkit and some terminal-fu to find a flag in this disk image:  
> [dds1-alpine.flag.img.gz](https://mercury.picoctf.net/static/a734f18939e0aaea9d27bc7a243a0ed0/dds1-alpine.flag.img.gz)

---

## Steps to Solution

### 1. Extract the Disk Image

```bash
gunzip dds1-alpine.flag.img.gz
```

### 2. File Type Analysis

```bash
file dds1-alpine.flag.img
```
**Output:**  
`DOS/MBR boot sector; partition 1 : ID=0x83, active, start-CHS (0x0,32,33), end-CHS (0x10,81,1), startsector 2048, 260096 sectors`

### 3. Find the Flag Using Strings

The challenge hints at using `srch_strings` (from Sleuthkit), but `strings` works as well:

```bash
strings dds1-alpine.flag.img | grep pico
```

**Output:**
```
ffffffff81399ccf t pirq_pico_get
ffffffff81399cee t pirq_pico_set
ffffffff820adb46 t pico_router_probe
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_a69a712c}
```

The flag is clearly visible!

---

## Flag

```
picoCTF{f0r3ns1c4t0r_n30phyt3_a69a712c}
```

---

## Summary

- Use `gunzip` to extract the image.
- Use `strings` (or `srch_strings` from Sleuthkit) to search for readable text in the disk image.
- Grep for `pico` to find the flag.

## Command Reference

```bash
gunzip dds1-alpine.flag.img.gz
file dds1-alpine.flag.img
strings dds1-alpine.flag.img | grep pico
```

**Flag:**  
`picoCTF{f0r3ns1c4t0r_n30phyt3_a69a712c}`
