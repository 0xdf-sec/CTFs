# picoCTF: Sleuthkit Intro - Writeup

## Challenge Description

> Download the disk image and use `mmls` on it to find the size of the Linux partition.  
> Connect to the remote checker service to check your answer and get the flag.  
> [Download disk image](https://artifacts.picoctf.net/c/164/disk.img.gz)

---

## Solution Steps

### 1. Extract the Disk Image

```bash
gunzip disk.img.gz
```

### 2. Inspect the Image

```bash
file disk.img
```
**Output:**  
`DOS/MBR boot sector; partition 1 : ID=0x83, active, start-CHS (0x0,32,33), end-CHS (0xc,190,50), startsector 2048, 202752 sectors`

### 3. Find the Partition Size

You can use `mmls` or just read the output above.

- **Linux partition size:** 202752 sectors

### 4. Connect to the Checker Service

```bash
nc saturn.picoctf.net 62454
```
**Prompt:**  
`What is the size of the Linux partition in the given disk image? Length in sectors:`

**Input:**  
`202752`

**Output:**  
`Great work! picoCTF{mm15_f7w!}`

---

## Flag

```
picoCTF{mm15_f7w!}
```

---

## Summary

- Use `gunzip` to extract the disk image.
- Use `file` or `mmls` to determine Linux partition size in sectors.
- Submit the answer to the checker service to receive the flag.

**Flag:**  
`picoCTF{mm15_f7w!}`
