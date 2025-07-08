---
title: API Docs!
nav_order: 2
layout: default
has_children: true
---

Hi! This API allows you to interact with my save editor, allowing you to make macros like the one below. Check the sidebar for specific documentation.
* [this]() describes the API calls for the yokai editor (available through devtools console once the save is loaded).
* [this!]() describes the API calls for the key items editor  (again, available through devtools console once the save is loaded).
* [thiswow!]() describes the API calls for the general editor  (again, AGAIN, available through devtools console once the save is loaded).

Anyway, here are the examples :)


## Examples

Yo-kai Example #1: Fixes IVs of all Yo-kai:
```js
(function fixYokaiData() { // define an IIFE for no reason (you dont need to do this I just felt like it when I made this example)
    const yokaiList = getAllYokai(); // get all yokai
    console.log(`Fixing ${yokaiList.length} Yo-kai...`);

    let ivTotalTarget = 40; // IV sum

    yokaiList.forEach((yokai, index) => { // iterate through - this is kinda obvious
        // --- Fix IVs --- //
        let IV_HP = yokai.get("IV_HP"); // get all the Ivs of the current yokai
        let IV_Str = yokai.get("IV_Str");
        let IV_Spr = yokai.get("IV_Spr");
        let IV_Def = yokai.get("IV_Def");
        let IV_Spd = yokai.get("IV_Spd");

        let IV_Sum = Math.floor(IV_HP / 2) + IV_Str + IV_Spr + IV_Def + IV_Spd; // should be 40 if it has valid IVs - HP is worth half

        let needsFix = (IV_HP % 2 !== 0) || (IV_Sum !== ivTotalTarget); // checks if its valid HP must be even since the point goal is 40 and nothing else can be decimal except HP so if HP isnt whole the goal cant be reached - meaning HP must be even

        if (needsFix) {
            let ivs = [0, 0, 0, 0, 0];

            for (let i = 0; i < ivTotalTarget; ++i) { // good ol trial and error
                let r = Math.floor(Math.random() * 5);
                ivs[r]++;
            }

            yokai.set("IV_HP", ivs[0] * 2); // Even HP
            yokai.set("IV_Str", ivs[1]);
            yokai.set("IV_Spr", ivs[2]);
            yokai.set("IV_Def", ivs[3]);
            yokai.set("IV_Spd", ivs[4]);
        }
    });

    console.log("All done!");
})();
```

Yo-kai Example #2: The inner workings of the copy-paste button (requires API v1.1)
```js


function copy() {
    const slotIndex = currentSlot();
    if (slotIndex === null) {
        alert("No Yo-kai slot selected to copy");
        return;
    }
    window.copiedYokaiSlot = slotIndex;
    console.log(`Copied Yo-kai from slot ${slotIndex}`);
}

function paste() {
    if (window.copiedYokaiSlot === undefined) {
        alert("No Yo-kai copied");
        return;
    }
    
    const targetSlot = currentSlot();
    if (targetSlot === null) {
        alert("No target slot selected");
        return;
    }
    
    copyYokaiSlot(window.copiedYokaiSlot, targetSlot);
    syncSave();
}

function copyYokaiSlot(sourceIndex, targetIndex) {
    let yokaiList = getAllYokai();
    if (sourceIndex < 0 || sourceIndex >= yokaiList.length) {
        console.error("Invalid source slot:", sourceIndex);
        return;
    }
    if (targetIndex < 0 || targetIndex >= yokaiList.length) {
        console.error("Invalid target slot:", targetIndex);
        return;
    }
    
    // Get raw hex for the source Yo-kai
    let fieldsToCopy = [
        "num1", "num2", "youkaiId", "nickname", "unused1", "specialUnlock",
        "expPoint", "energy", "ownerId", 
        "IV_HP", "IV_Str", "IV_Spr", "IV_Def", "IV_Spd",
        "EV_HP", "EV_Str", "EV_Spr", "EV_Def", "EV_Spd",
        "SC_Str", "SC_Spr", "SC_Def", "SC_Spd",
        "unknown", "level", "special6", "loafAndAi", "specialEquip"
    ];

    fieldsToCopy.forEach(field => {
        try {
            let raw = yokaiList[sourceIndex].getRaw(field);
            yokaiList[targetIndex].setRaw(field, raw);
        } catch (e) {
            console.warn(`Skipping field '${field}' (reason: ${e.message})`);
        }
    });

    console.log(`Copied Yo-kai from slot ${sourceIndex} to slot ${targetIndex}`);
}
```

Key items Example #1 Shuffles Item Order
```js
// Simple Item Randomizer Example
// Swaps item IDs between all non-empty slots to randomize inventory order

function randomizeItems() {
    // Check if save is loaded
    if (!SaveAPI.isLoaded) {
        console.log("Please load a save first!");
        return;
    }
    
    // Get all non-empty items
    const items = SaveAPI.getAllItems().filter(item => item.itemId !== 0);
    
    if (items.length < 2) {
        console.log("Need at least 2 items to randomize!");
        return;
    }
    
    // Extract item IDs and shuffle them
    const itemIds = items.map(item => item.itemId);
    
    // Simple shuffle (Fisher-Yates)
    for (let i = itemIds.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [itemIds[i], itemIds[j]] = [itemIds[j], itemIds[i]];
    }
    
    // Apply shuffled IDs back to original slots (keeping nums)
    for (let i = 0; i < items.length; i++) {
        SaveAPI.setItem(items[i].index, {
            num1: items[i].num1,    // Keep original num1
            num2: items[i].num2,    // Keep original num2
            itemId: itemIds[i]      // Use shuffled ID
        });
    }
    
    // Update the UI
    SaveAPI.updateUI();
    
    console.log(`Randomized ${items.length} items!`);
}

console.log("Item randomizer loaded! Use: randomizeItems()");
```
