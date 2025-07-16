# picoCTF: Disk, disk, sleuth! II - Writeup

## Challenge Description

> All we know is the file with the flag is named `down-at-the-bottom.txt`...  
> Disk image: [dds2-alpine.flag.img.gz](https://mercury.picoctf.net/static/6cd5ca45d75250451931cea538fb38c0/dds2-alpine.flag.img.gz)

---

## Solution Steps

### 1. Extract the Disk Image

```bash
gunzip dds2-alpine.flag.img.gz
```

### 2. Check Disk Image Info

```bash
file dds2-alpine.flag.img
```

### 3. Identify Partitions

```bash
mmls dds2-alpine.flag.img
```
**Output:**
```
Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000262143   0000260096   Linux (0x83)
```
_Note: Linux partition starts at sector 2048._

### 4. Search for the Flag File

```bash
fls -r -p -o 2048 dds2-alpine.flag.img | grep down-at-the-bottom.txt
```
**Output:**
```
r/r 18291:      root/down-at-the-bottom.txt
```
_Inode number: 18291_

### 5. Extract the Flag

```bash
icat -o 2048 dds2-alpine.flag.img 18291
```
**Output:**
```
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( p ) ( i ) ( c ) ( o ) ( C ) ( T ) ( F ) ( { ) ( f ) ( 0 ) ( r ) ( 3 ) ( n )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( s ) ( 1 ) ( c ) ( 4 ) ( t ) ( 0 ) ( r ) ( _ ) ( n ) ( 0 ) ( v ) ( 1 ) ( c )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
   _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( 3 ) ( _ ) ( f ) ( f ) ( 2 ) ( 7 ) ( f ) ( 1 ) ( 3 ) ( 9 ) ( } )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
```

### 6. Flag

```
picoCTF{f0r3ns1c4t0r_n0v1c3_ff27f139}
```

---

## Summary

- Use `gunzip` to extract the image.
- Use `mmls` to find partition offset.
- Use `fls` to find the file and inode number.
- Use `icat` to extract the file content (flag).

**Flag:**  
`picoCTF{f0r3ns1c4t0r_n0v1c3_ff27f139}`
