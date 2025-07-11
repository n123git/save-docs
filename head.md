---
parent: Save Docs!
layout: default
nav_order: 4
title: Header Files
---

This page documents Header files (also known as `head.yw`) for V2.0 saves. Note that each game has 3 save files (`game*.yw`) but only ONE header (`head.yw`) which contains the data for all save files.

<sup>Disclaimer: **Most** changes to the `head.yw` only affect the preview you see before you enter the save file, NOT the actual save file (This is very important, dont forget :P). Exceptions to this rule include the save file names, and decryption seed.</sup><br/>
The `head.yw` has 3 "player blocks" - with each corresponding to a different save file. Some things however, are the same for all save files and are therefore not in a player block. For example the seed located in the head file at `0x0C`. Despite head files using a SectionID system there is only one Section: Section ID 248 (`F8`) meaning that a direct offset can be used - which for simplicity is used in this doc.

Here are some of the known elements:

## Fixed Elements (1 per head)

| Offset | Length  | Description                                                                          |
| ------ | ------- | ------------------------------------------------------------------------------------ |
| `0x0C` | 0x04    | (Uint32) - A seed ran through a Xorshift-based PRNG to derive an AES encryption key, used to decrypt V2 saves. |


## Player Block Positions

| Block # | Start Addr | End Addr | Size      |
| ------- | ---------- | -------- | --------- |
| 1       | `0x5390`   | `0x5417` | 136 bytes |
| 2       | `0x5418`   | `0x549F` | 136 bytes |
| 3       | `0x54A0`   | `0x5527` | 136 bytes |

## Save File Dependant (3 per head)
Note that the `Offset` in this table assumes the player block is at `0x00` instead of providing all 3 possible positions. A list of player name positions can be found above for reference.

| Offset | Length  | Description                                                                          |
| ------ | ------- | ------------------------------------------------------------------------------------ |
| `0x00` | 0x08    | Player name (UTF-8 or cp932 depending on region), fixed length 8 bytes (padded with `00` if shorter)|
| `0x09` | 0x1A    | Unknown (18 bytes)                                                                   |
| `0x1B` | 0x01    | Unknown (1 byte)                                                                     |
| `0x1C` | 0x24    | Unknown (10 bytes)                                                                   |
| `0x25` | 0x01    | **Watch Rank** (`0x05 = S`, `0x04 = A`, `0x03 = B`, etc.)                            |
| `0x26` | 0x43    | Unknown (67 bytes) — **still part of the 0x60 block total from 0x08–0x68**           |
| `0x67` | 0x02    | Year (`UInt16`)                                                                      |
| `0x6B` | 0x01    | Day (`LEB128` or `UInt8`)                                                            |
| `0x6C` | 0x01    | Month (`LEB128` or `UInt8`)                                                          |
| `0x6D` | 0x01    | Hour (`LEB128` or `UInt8`)                                                           |
| `0x6E` | 0x01    | Minute (`LEB128` or `UInt8`)                                                         |
| `0x6F` | 0x01`?` | (Optional) Seconds? (`LEB128` or `Uint8`, Not displayed but may be internally used)  |

## Notes
The `head.yw` is extremely tolerant of illegal values, it allows illegal names (disallowed unicode characters appear as a black-box), an hour value of 25 and any possible change BUT please note that changing thre encryption keys at `0x0C` can render your save file unusable without brute force recovery.

TO;DO document save file play time, save file location, save file party, save file gender, save file version

<!--
legacy:
1-8 = Name (Length: 0x8 or 8) in UTF8-LE
9-36 = Unknown
37 = Save File Rank 05 = S, 04 = A etc
38 - 104 Unknown (Length: 0x60 or 96)
Then Int/Uint16 Year followed by LEB/ULEB128 Day, LEB/ULEB128 OR INT8/UINT8 Month (Identical in this case), then LEB/ULEB128 OR INT8/UINT8 Hour (Again, identical in this case), then LEB/ULEB128 OR INT8/UINT8 Minute (Again, identical in this case), then (this isn't shown, but is internally used) then LEB/ULEB128 OR INT8/UINT8 Seconds?

53B8-53D2

53C0-53D0

-->
