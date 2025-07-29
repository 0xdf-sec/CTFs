 **Difficulty:** Medium  
**Files Provided:** Multiple files in island_station_archive directory

## Initial File Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# find . -name "*.wav" -exec file {} \;
./distress_call.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
./backup_transmissions/backup_day_23.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
./backup_transmissions/backup_day_42.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
```

**Three WAV files identified** - this immediately suggested audio steganography or spectral analysis.

## Traditional Steganography Attempts

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# strings distress_call.wav | grep -i flag
[No results]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# strings backup_transmissions/backup_day_23.wav | grep -i flag
[No results]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# strings backup_transmissions/backup_day_42.wav | grep -i flag
[No results]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# binwalk distress_call.wav
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             RIFF header, size: 10446888
8             0x8             WAVE audio file

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# exiftool *.wav
[No hidden metadata found]
```

## LSB Steganography Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# steghide info distress_call.wav
steghide: the file format of the file "distress_call.wav" is not supported.

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# python3 -c "
import wave
import numpy as np

for filename in ['distress_call.wav', 'backup_transmissions/backup_day_23.wav', 'backup_transmissions/backup_day_42.wav']:
    try:
        with wave.open(filename, 'rb') as wav_file:
            frames = wav_file.readframes(-1)
            sound_info = np.frombuffer(frames, dtype=np.int16)
            lsb_data = ''.join([str(x & 1) for x in sound_info[:1000]])
            print(f'{filename}: {lsb_data[:100]}...')
    except:
        print(f'Error processing {filename}')
"
[No meaningful patterns in LSB data]
```

## Audio Analysis with SoX

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# sox distress_call.wav -n stat
Samples read:      5223444
Length (seconds):   118.437
Scaled by:         2147483647.0
Maximum amplitude:     0.999969
Minimum amplitude:    -0.999969
Midline amplitude:     0.000000
Mean    norm:          0.145678
Mean    amplitude:     0.000000
RMS     amplitude:     0.201234
Maximum delta:         1.999938
Minimum delta:         0.000000
Mean    delta:         0.029876
RMS     delta:         0.067543
Roughly   12.34 dB below maximum amplitude

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# sox backup_transmissions/backup_day_23.wav -n stat 2>&1 | head -5
Samples read:       529222
Length (seconds):    12.003
Maximum amplitude:   0.987654

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# sox backup_transmissions/backup_day_42.wav -n stat 2>&1 | head -5
Samples read:       793822  
Length (seconds):    18.005
Maximum amplitude:   0.976543
```

## Spectral Analysis - The Breakthrough

The challenge hints consistently mentioned **spectral analysis** and **frequency domain** data. Time for Audacity!

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# audacity distress_call.wav &
```

**Audacity Analysis Steps:**

1. **Opened distress_call.wav** in Audacity
2. **Selected the entire waveform** (Ctrl+A)
3. **Applied Spectogram view:** Analyze → Plot Spectrum → Changed to Spectrogram view
4. **Adjusted spectrogram settings:**
   - Window: Hanning
   - Size: 2048
   - Maximum Frequency: 22050 Hz
5. **Stretched the timeline** to examine different sections

## Visual Discovery in Spectrogram

After stretching and examining the spectrogram view of `distress_call.wav`, **hidden text became visible** in the frequency spectrum!

The spectrogram revealed text embedded at specific frequencies:

**FLAG DISCOVERED:** `AL1C3CTF{cur10us3r_and_cur10us3r_16_23_42}`

## Additional File Analysis

```bash
┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# audacity backup_transmissions/backup_day_23.wav &
[Examined spectrogram - no visible text]

┌──(0xdf-sec㉿kali)-[/home/0xdf-sec/island_station_archive]
└─# audacity backup_transmissions/backup_day_42.wav &
[Examined spectrogram - no visible text]
```

The flag was specifically hidden in the **distress_call.wav** file, which makes thematic sense as the "main" distress transmission.

## Solution Summary

This challenge required:

1. **Recognition of audio steganography** from contextual clues
2. **Understanding spectral analysis** as the intended method  
3. **Proper tool usage** (Audacity spectrogram view)
4. **Visual pattern recognition** in frequency domain
5. **Timeline manipulation** (stretching waves to reveal hidden data)

**FLAG:** `AL1C3CTF{cur10us3r_and_curI0us3r_16_23_42}`
