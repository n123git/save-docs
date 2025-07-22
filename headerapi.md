---
parent: API Docs!
layout: default
nav_order: 3
title: Header API
---


## Misc
* `utf8` - A global var that determines whether to use `cp932` (JP ver.) or `UTF-8` (International ver.). True by default. 
* `panic()` - A function used to signal warnings - this is usually used by the API to signal errors, more can be found in the Error section.
* `api` - A global (and pre-defined) instance of `HeaderAPI`.
* `fileBuffer` - decrypted save data used by the UI - stored as a `Uint8Array`.
* `playerOffsets` - a global array containing the start pos' of each player block (`[0x5390, 0x5418, 0x54A0]`).
* `encoder.encode()` - a global alias for `Text.Encode(str)`.
* `decoder.decode()` - a global alias for `Text.Decode(str)`.
* `async isDecrypted(fileBuffer)` - Checks if a `head.yw`/`head.ywd` is decrypted - returns true if decrypted, if encrypted or heavily corrupted returns false.
* `async fullDecrypt()` - Decrypts a `head.yw` from `fileBuffer` (mutates it) (`head.yw` -> `head.ywd`). This also works on Yo-kai Watch 1 save files - although unintentional.
* `async fullEncrypt()` - Encrypts a `head.ywd` from `fileBuffer` (mutates it) (`head.ywd` -> `head.yw`). This also works on Yo-kai Watch 1 save files - although unintentional.
* `readUInt32(offset)` - Takes an offset of type `Number` and returns a `Number` corresponding that point in `fileBuffer` read as a uint32 (unsigned 32-bit integer).
* `readUInt16(offset)` - Takes an offset of type `Number` and returns a `Number` corresponding that point in `fileBuffer` read as a uint16 (unsigned 16-bit integer).
* `writeUInt32(offset, value)` - Takes an offset of type `Number` and a value, it writes that value as a `uint32` (unsigned 32-bit integer) and the offset in `fileBuffer`.
* `writeUInt16(offset, value)` - Takes an offset of type `Number` and a value, it writes that value as a `uint16` (unsigned 16-bit integer) and the offset in `fileBuffer`.
* `async download()` - Downloads `fileBuffer` as `head.ywd`.
* `async downloadEncrypted()` - Encrypts `fileBuffer`, downloads it as a `head.yw` and then decrypts it using `fullDecrypt` and `fullEncrypt` respectively.
* `formatPlayTime(seconds)` - Takes seconds in the form of `Number` and returns a string in format `HH:MM:SS` (seconds, minutes and hours are 0 padded when needed to look good).
* `getUTF8ByteLength(str)` - Returns the amount of bytes a string takes up. Note: DESPITE THE NAME STATING UTF8 IT USES CP932 WHEN `utf8 == false`!


## Formats
* `playerIndex` - A number from 0-2 where 0 is the 1st save file/player, 1 is the 2nd and 2 is the 3rd.

## HeaderAPI
* Constructor: `HeaderAPI(filebuf = fileBuffer)`
  * `filebuf` is where the save data is stored in the API.
* `async refreshUI()` - Refreshes/reloads the UI based on the API's save data.
* `async grabChanges()` - Saves the UI changes and sets the API's save data to the UI save data.
* `exportSave()` - Exports the API's save data as a `Uint8Array`.
* `getPlayers()` - Exports a JSON snapshot of the API's save data in this format:
  * `[{ name, rank, playTime, year, month, day, hour, minute, party } /* player 1 */, { name, rank, playTime, year, month, day, hour, minute, party } /* player 2 */, { name, rank, playTime, year, month, day, hour, minute, party } /* player 3 */]`
* `setPlayerName(index, newName)` - Takes a `playerIndex` and `name` and sets that player's name to the input.

## Text Class
* `static Encode(str, utf8 = window.utf8)` - Encodes a string -> `Uint8Array`, if `utf8` is true it'll use UTF-8 encoding otherwise cp932/Windows 31-J (UTF-8 is needed for INTL versions while cp932 for JP).
* `static Decode(bytes, utf8 = window.utf8)` - Decodes a `Uint8Array` -> string, if `utf8` is true it'll use UTF-8 encoding otherwise cp932/Windows 31-J (UTF-8 is needed for INTL versions while cp932 for JP).
* `constructor() { throw new Error("Do NOT use the constructor.")}` - Warning.


## Errors
