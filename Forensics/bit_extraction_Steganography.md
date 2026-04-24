# Bit Extraction Steganography – README

## Overview
The goal is to extract hidden data embedded within an image by analyzing pixel-level information.

The provided script performs a **systematic bit-level extraction** across RGB channels to uncover a hidden flag (expected format: `FLAG{...}`).

---

## Approach

Images store pixel data in three color channels:

* Red (R)
* Green (G)
* Blue (B)

Each channel contains 8 bits per pixel. Steganography often hides data in:

* Least Significant Bits (LSBs)
* Specific bit planes
* Different traversal orders (row-wise, column-wise)

This script brute-forces multiple extraction strategies:

* Different channels (R, G, B)
* Bit positions (0–7)
* Bit ordering (MSB-first, LSB-first)
* Transposed data
* Bit shifting
* Bit reversal
* XOR transformations

---

## Script

```python
from PIL import Image
import numpy as np

img = Image.open("challenge.png")
arr = np.array(img)

FLAGS = [b"FLAG{"]

def extract_bytes(channel_data, bit, lsb_first=False, transpose=False):
    bits = ((channel_data >> bit) & 1)

    if transpose:
        bits = bits.T

    bits = bits.flatten()

    out = []
    for i in range(0, len(bits), 8):
        byte = 0
        chunk = bits[i:i+8]

        if lsb_first:
            for j, b in enumerate(chunk):
                byte |= (b << j)
        else:
            for b in chunk:
                byte = (byte << 1) | b

        out.append(byte)

    return bytes(out)

def reverse_bits(byte):
    return int('{:08b}'.format(byte)[::-1], 2)

def check_flag(data, label):
    for flag in FLAGS:
        if flag in data:
            idx = data.find(flag)
            print(f"\n[FOUND] {label}")
            print(data[idx:idx+120])
            return True
    return False

def try_all(data, label):
    if check_flag(data, label + " | direct"):
        return

    rev = bytes(reverse_bits(b) for b in data)
    if check_flag(rev, label + " | bit-reversed"):
        return

    for shift in range(8):
        bits = []

        for byte in data:
            for i in range(8):
                bits.append((byte >> (7 - i)) & 1)

        bits = bits[shift:]

        out = []
        for i in range(0, len(bits), 8):
            byte = 0
            for b in bits[i:i+8]:
                byte = (byte << 1) | b
            out.append(byte)

        shifted = bytes(out)

        if check_flag(shifted, label + f" | shift={shift}"):
            return

    for k in range(256):
        x = bytes(b ^ k for b in data)
        if check_flag(x, label + f" | xor={k}"):
            return


channels = {
    "R": arr[:,:,0],
    "G": arr[:,:,1],
    "B": arr[:,:,2]
}

for cname, channel in channels.items():
    for bit in range(8):
        data = extract_bytes(channel, bit, lsb_first=False, transpose=False)
        try_all(data, f"{cname}-bit{bit} (MSB-first)")

        data = extract_bytes(channel, bit, lsb_first=True, transpose=False)
        try_all(data, f"{cname}-bit{bit} (LSB-first)")

        data = extract_bytes(channel, bit, lsb_first=False, transpose=True)
        try_all(data, f"{cname}-bit{bit} (transposed)")
```

---

## Key Techniques Used

* **Bit-plane extraction**: Isolating individual bits from pixel values
* **Byte reconstruction**: Grouping bits into bytes
* **Traversal variations**:

  * Row-major
  * Column-major (transpose)
* **Bit manipulation**:

  * Bit reversal
  * Bit shifting
* **Obfuscation handling**:

  * XOR brute force (0–255)

---

## Expected Outcome

The script searches for a flag pattern:

```
FLAG{...}
```

Once found, it prints:

* The method used (channel, bit, transformation)
* The extracted flag (and surrounding bytes)

---

## Usage

1. Place `challenge.png` in the same directory.
2. Change the flag name as per your need
3. Install dependencies:

   ```bash
   pip install pillow numpy
   ```
4. Run:

   ```bash
   python solve.py
   ```

---
