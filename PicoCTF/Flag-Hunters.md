#### Description

Lyrics jump from verses to the refrain kind of like a subroutine call. There's a hidden refrain this program doesn't print by default. Can you get it to print it? There might be something in it for you.The program's source code can be downloaded [here](https://challenge-files.picoctf.net/c_verbal_sleep/60bb6b075d79b2e8a2107520060ceb97145c158f47651772fcf53dcdb3df63b1/lyric-reader.py).

Connect to the program with netcat:`$ nc verbal-sleep.picoctf.net 50812`

## Challenge Analysis

**The Problem:**
- The script contains a `secret_intro` that includes the flag
- But the reader starts from `[VERSE1]`, skipping the intro with the flag
- We need to manipulate the execution to read from the beginning (line 0)

**Key Vulnerability:**
The script takes unsanitized user input at the "Crowd:" prompt:
```python
elif re.match(r"CROWD.*", line):
    crowd = input('Crowd: ')
    song_lines[lip] = 'Crowd: ' + crowd
    lip += 1
```

## Solution Commands

If you're solving this challenge, here's what you need to do:

```bash
# Run the script
python3 lyric-reader.py

# When prompted with "Crowd:", enter:
some_string;RETURN 0
```

**Why this works:**
1. The input `some_string;RETURN 0` gets processed by the script
2. The script splits on `;` and processes each part
3. `RETURN 0` is interpreted as a command to jump to line 0
4. Line 0 contains the `secret_intro` with the flag

**Alternative payloads you could try:**
```bash
# At the Crowd prompt:
;RETURN 0
dummy;RETURN 0  
anything;RETURN 0
```

## Key Learning Points

1. **Input Sanitization**: The script doesn't validate user input, allowing command injection
2. **Control Flow Manipulation**: Using the `RETURN` command to jump to different parts of the song
3. **String Processing**: The `;` delimiter allows multiple commands in one input

The flag format appears to be:  `picoCTF{70637h3r_f0r3v3r_62666df2}`
