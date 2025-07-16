# picoCTF: Information - Writeup

## Challenge Description

> Files can always be changed in a secret way. Can you find the flag?  
> [cat.jpg](https://mercury.picoctf.net/static/e5825f58ef798fdd1af3f6013592a971/cat.jpg)

---

## Solution Steps

### 1. Download and Analyze the Image

Download `cat.jpg` and run ExifTool to check metadata:

```bash
exiftool cat.jpg
```

### 2. Inspect Metadata

Look for suspicious or custom fields.  
The output contains:

```
License                         : cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9
```

### 3. Decode the Flag

The value is Base64 encoded. Decode it:

```bash
echo "cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9" | base64 -d
```

**Output:**  
`picoCTF{the_m3tadata_1s_modified}`

---

## Flag

```
picoCTF{the_m3tadata_1s_modified}
```

---

## Summary

- Use `exiftool` to inspect image metadata.
- Find the flag in the `License` field, Base64 encoded.
- Decode the flag to get the answer.

**Flag:**  
`picoCTF{the_m3tadata_1s_modified}`
