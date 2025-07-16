---
parent: Save Docs!
layout: default
nav_order: 1
title: General (Read First)
---

This page outlines the basic structure of *decrypted* save files.


There are two file types, both in little-endian format for all datapoints unless otherwise specified:
 * `game*.yw` files. These hold the main save data - where the `*` corresponds to the save slot (i.e., `game1.yw` is the first slot, `game2.yw` is the second, etc.). These store nearly everything and use a `SectionID` format.

 * `head.yw` files, these contain the player's name, encryption keys for V2 saves, preview data (all the data seen before you click on a save file) etc. These are encrypted identically to `YW1` save files - see the decrypt page for more. Despite these also using a `SectionID` format - it only has one top-level `Section` leading direct offsets to still work.

> ⚠️ *Modifying the preview data is mostly pointless*, as the game will overwrite most of it when saving - certain exceptions do exist such as the player's name however, which *can* be edited persistently (although this *actually* changes the player's name) - See the Header Files page for more info.<br/>

# **SectionID Format**

Save data in `game*.yw` files is organized into **nested sections**, each defined by a pair of headers:

- **Header 1 (`h1`)**:  
  A 2-byte `int16` enum to mark section boundaries:
  - `0xFFFE` = Section Start
  - `0xFEFF` = Section End

- **Header 2 (`h2`)**:  
  A 4-byte value with the structure below:
  - **Bits 0–7**: Section ID (`uint8`)
  - **Bits 8–31**: Section size in bytes (excluding the 8 bytes used by `h1` and `h2`)



## **Section ID (ID)**

This is a `Uint8` (0-255) used to uniquely identify the section, kind of like a `UUID`. For instance, `0x06` is the SectionID for Key Items, which is constant, and NEVER varies between save files, despite the exact offset being inconsistent, and therefore cannot directly be used. Each section however *always* contains the same type of data at the same offset *relative* to the start of the section (`FE`).
The general structure would look something like this:

```
Root (ID: 1, size: 64)
├── Child (ID: 5, size: 32)
│   └── Grandchild (ID: 9, size: 8)
└── Child (ID: 6, size: 12)
```

Where each Section would be formatted like this:

| Offset | Bytes                                                   | Meaning                                |
| ------ | ------------------------------------------------------- | -------------------------------------- |
| 0x00   | `FE FF 00 00`                                           | `h1` = Section Start Marker (`0xFFFE`) |
| 0x04   | `F1 34 15 01`                                           | `h2` = Section Header Data             |
|        | → Reversed: `01 15 34 F1` (little-endian to big-endian) |                                        |
|        | → Full value: `0x011534F1`                              |                                        |
|        | → ID = **`0xF1`** (last byte)                           |                                        |
|        | → Size = **`0x011534`** (upper 3 bytes = 70964 bytes)   |                                        |
| 0x08   | (data)                                                  | Payload of section ID 241              |
| ...    | ...                                                     | Data. Can be nested.                   |
| 0x1153C| `FF FE`                                                 | End Marker (0xFEFF)                    |


## Misc Notes

* A decrypted `game*.ywd` will have data before/after the first top-level section for re-encryption.

### **Common Data Types**

- **Integers**: Typically stored as 32-bit signed values - although obviously depends on the datapoint itself.
  - **IDs**: these are stored as a standard CRC-32 Checksum. An example of this is Location; which is stored as the CRC-32 of it's file name. A list of all the file names can be found [here](https://tcrf.net/Notes:Yo-kai_Watch_2). Other examples include Items and Yo-kai.
- **Text**: Encoding depends on region (see below).
- **Bitmasks**: Compact representation of boolean states (example below):

00000000000000000000 → No trophies<br/>
11111111111111111111 → All trophies<br/>
10000000000000000001 → First and last trophy only

Used for data like:
- Trophies
- Opened treasure chests
- Unlocked win poses<br/>
etc



## Reigonal Differences:
* JP copies use cp932 (known as Code Page 932 or Windows 31-J) for text - which is an extension of SHIFT_JIS, whereas international save files use UTF-8. Since they both have ASCII compatibility (not extended ASCII) - the problem is usually more evident using the international system for JP saves than the other way around.

## Code Examples
* SectionID parsing code (taken from an old version of my save editor):<br/>

```js

(function (global) {
  class Section {
    constructor(id, size, pos) {
      this.id = id;
      this.size = size;
      this.pos = pos;
      this.children = [];
    }
    addChild(child) {
      this.children.push(child);
    }
  }

  function parseSavedata(fullBytes) {
    if (!(fullBytes instanceof Uint8Array)) {
      throw new Error("Input must be a Uint8Array.");
    }

    if (fullBytes.length <= 0x20) {
      console.error("Input too short to skip 0x20 offset.");
      return null;
    }

    const bytes = fullBytes.slice(0x20); // skip the part outside of the first/main SectionID (0x01)
    const view = new DataView(bytes.buffer, bytes.byteOffset, bytes.byteLength);
    const stack = [];
    let root = null;
    let pos = 0;

    function atEnd() { // self-explanatory
      return pos >= bytes.length;
    }

    if (pos + 4 > bytes.length) return null;
    let h1 = view.getUint32(pos, true);
    pos += 4;

    if ((h1 & 0xFFFF) !== 0xFFFE) return null; // if the low 16-bits == 0xFFFE, then continue, also gotta love magic numbers

    if (pos + 4 > bytes.length) return null; // no data
    let h2 = view.getUint32(pos, true);
    pos += 4;

    let id = h2 & 0xFF; // get the low 8-bits of h2 as id
    let size = h2 >>> 8; // shift right by 8 bits — you get the remaining 24 bits as size

    root = new Section(id, size, pos); // init the new sectionID
    stack.push(root);

    while (!atEnd()) {
      if (pos + 4 > bytes.length) break; // too short/no data left
      h1 = view.getUint32(pos, true);
      pos += 4;

      while ((h1 & 0xFFFF) === 0xFFFE) { // same check as earlier
        if (pos + 4 > bytes.length) return null;

        h2 = view.getUint32(pos, true);
        pos += 4;
        id = h2 & 0xFF; // again grab the ID
        size = h2 >>> 8; // and the size

        if (stack.length === 0 || pos + size > bytes.length) return null; // very sad oh noes

        const sec = new Section(id, size, pos); // init a section
        stack[stack.length - 1].addChild(sec); // now we
        stack.push(sec); // add it to stack

        if (pos + 4 > bytes.length) break; // too short/no data left
        h1 = view.getUint32(pos, true);
        pos += 4;
      }

      if ((h1 & 0xFFFF) === 0xFEFF) {
        if (stack.length === 0) return null;
        stack.pop();
      } else {
        if (stack.length === 0) {
          console.warn("Data outside sections, stopping parse but exporting tree anyway.");
          break; // Don't return null — just stop parsing.
        }

        const current = stack[stack.length - 1];
        const dataToSkip = current.size - 4;
        if (pos + dataToSkip > bytes.length) return null;
        pos += dataToSkip;
      }
    }

    if (!root) return null;

    // Final check: if the stack is unbalanced, log a warning, but still return the result
    if (stack.length !== 0) {
      console.warn("Unbalanced sections: some sections were not closed properly.");
    }

    return sectionToObject(root);
  }

  function sectionToObject(section) {
    return {
      id: section.id,
      size: section.size,
      pos: section.pos,
      children: section.children.map(sectionToObject),
    };
  }

function grabOffset(node, targetId) {
  console.log(`Checking node id: ${node.id}`);
  if (node.id === targetId) {
    console.log(`Found id ${targetId}, pos: ${node.pos}`);
    return node.pos;
  }

  for (const child of node.children) {
    const pos = grabOffset(child, targetId);
    if (pos !== null) {
      return pos;
    }
  }

  return null;
}

  global.SaveDataParser = {
    parseSavedata,
  };

global.grabOffset = grabOffset;

global.IDObj = { /* The IDs for different data sections note that these sections dont just hold one piece of data i.e. 0x07 isnt JUST yokai - it's just the first use found for that section*/
  "Yokai": "0x07",
  "Item": "0x04",
  "Equipment": "0x05",
  "Important": "0x06",
  "misc1": "0x01",
  "misc9": "0x09",
  "Soul": "0x13",
  "Medallium": "0x01",
  "misc8": "0x14"
}
global.genOffset = {
  "Yokai": 0x20,
  "Important": 0x20
} /* The general offset constants i.e. for Yokai it starts at the ID 0x07 but the first 0x20 bytes aren't for any Yo-kai in particular, and I havent figured out what they do, as of me writing this (I probably have, and just haven't updated this */
})(window);

```

## Unconfirmed
- After a h2 there is always 32 bytes (`0x20`) to skip past. This is correct for Key Items and Yo-kai.
