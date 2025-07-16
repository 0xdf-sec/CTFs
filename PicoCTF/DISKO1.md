# picoCTF: Disko 1 - Writeup

## Challenge Description

> The file given to us is disko-1.dd.gz that we need to unzip using gunzip.

---

## Solution Steps

### 1. Extract the Disk Image

```bash
gunzip disko-1.dd.gz
```

### 2. Try File Recovery

- Attempted with **testdisk**, but the disk is empty and no files can be recovered.

### 3. Search for the Flag with Strings

Since the disk image is empty, try searching for readable strings:

```bash
strings disko-1.dd | grep pico
```

**Output:**
```
:/icons/appicon
# $Id: piconv,v 2.8 2016/08/04 03:15:58 dankogai Exp $
...
picoCTF{1t5_ju5t_4_5tr1n9_c63b02ef}
```

---

## Flag

```
picoCTF{1t5_ju5t_4_5tr1n9_c63b02ef}
```

---

## Summary

- Use `gunzip` to extract the disk image.
- No files to recover, but the flag can be found using `strings` and `grep`.

**Flag:**  
`picoCTF{1t5_ju5t_4_5tr1n9_c63b02ef}`
