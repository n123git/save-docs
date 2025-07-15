---
parent: Save Docs!
layout: default
nav_order: 4
title: Header Files
---

This page documents Header files (also known as `head.yw`) for V2.0 saves. Note that each game has 3 save files (`game*.yw`) but only ONE header (`head.yw`) which contains the data for all save files.

The `head.yw` has 3 "player blocks" - with each corresponding to a different save file. Some things however, are the same for all save files and are therefore not in a player block. For example the seed located in the head file at `0x0C`. Despite head files using a SectionID system there is only one Section: Section ID 248 (`F8`) meaning that a direct offset can be used - which for simplicity is used in this doc.

Here are some of the known elements:

## Fixed Elements (1 per head)


| Offset | Length  | Description                                                                                                    |
| ------ | ------- | -------------------------------------------------------------------------------------------------------------- |
| `0x0C` | 0x04    | (Uint32) - A seed ran through a Xorshift-based PRNG to derive an AES encryption key, used to decrypt V2 saves. |

<br/>
 > #### **⚠️ changing the encryption key can render your save file unrecoverable with current technology (AES-128).**

## Player Block Positions

| Block # | Start Addr | End Addr | Block Length        |
| ------- | ---------- | -------- | ------------------- |
| 1       | `0x5390`   | `0x5417` | `0x88` (136) bytes  |
| 2       | `0x5418`   | `0x549F` | `0x88` (136) bytes  |
| 3       | `0x54A0`   | `0x5527` | `0x88` (136) bytes  |

## Save File Dependent (3 per head)
Note that the `Offset` in this table assumes the player block is at `0x00` instead of providing all 3 possible positions. A list of player name positions can be found above for reference.

| Offset | Length  | Notes                                                                                          |
| ------ | ------- | ---------------------------------------------------------------------------------------------------- |
| `0x00` | 0x08    | Player name (UTF-8 or cp932 depending on region), fixed length 8 bytes (padded with `00` if shorter) |
| `0x18` | 0x01    | Gender. **DO NOT EDIT THIS UNTIL FURTHER RESEARCH HAS BEEN CONCLUDED!**                              |
| `0x19` | 0x01    | Metadata - does various things. Note that for now, editing this isn't recommended. (`0xFF`) makes the game think you imported your data from BS/FS on a PS game copy. |
| `0x1B` | 0x01    | Unknown (1 byte)                                                                                     |
| `0x1C` | 0x24    | Unknown (10 bytes)                                                                                   |
| `0x25` | 0x01    | **Watch Rank** (`0x05 = S`, `0x04 = A`, `0x03 = B`, etc.)                                            |
| `0x28` | 0x04    | The **BaseID** (not **ParamID** like usual) of the 1st Yo-kai in the party?                          |
| `0x2C` | 0x04    | The **BaseID** (not **ParamID** like usual) of the 2nd Yo-kai in the party?                          |
| `...`  | ...     | ...                                                                                                  |
| `0x40` | 0x04    | Play Time (in seconds)                                                                               |
| `0x68` | 0x02    | Last Save Date Year (3DS clock)                                                                      |
| `0x6A` | 0x01    | Last Save Date Day (3DS clock)                                                                       |
| `0x6B` | 0x01    | Last Save Date Month (3DS clock)                                                                     |
| `0x6C` | 0x01    | Last Save Date Hour (3DS clock)                                                                      |
| `0x6E` | 0x01    | Last Save Date Minute (3DS clock)                                                                    |

## Notes
The `head.yw` is extremely tolerant of illegal values, it allows illegal names (disallowed unicode characters appear as a black-box), an hour value of 25 and most changes. And changing certain data incorrectly such as Gender can render your save file unusable - although note that this **IS** recoverable. **Most** changes to the `head.yw` only affect the preview you see *before* you enter the save file, NOT the actual save file (This is very important, dont forget :P) and often reset to their true state when the game is saved. Exceptions to this rule include the save file names, gender and decryption seed which actually affect behaviour and apply *permenantly*.

TO;DO document save file play time, save file location, save file version

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
