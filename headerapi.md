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

## Formats
* `playerIndex` - A number from 0-2 where 0 is the 1st save file/player, 1 is the 2nd and 2 is the 3rd.
* 
## HeaderAPI
* Constructor: `HeaderAPI(filebuf = fileBuffer)`
  * `filebuf` is where the save data is stored in the API.
* `async refreshUI()` - Refreshes/reloads the UI based on the API's save data.
* `async grabChanges()` - Saves the UI changes and sets the API's save data to the UI save data.
* `exportSave()` - Exports the API's save data as a `Uint8Array`.
* `getPlayers()` - Exports a JSON snapshot of the API's save data in this format:
  * `[{ name, rank, playTime, year, month, day, hour, minute, party } /* player 1 */, { name, rank, playTime, year, month, day, hour, minute, party } /* player 2 */, { name, rank, playTime, year, month, day, hour, minute, party } /* player 3 */]`
* `setPlayerName(index, newName)` - Takes a `playerIndex` and `name` and sets that player's name to the input.
  
## Errors
