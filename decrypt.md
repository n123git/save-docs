---
parent: Save Docs!
layout: default
nav_order: 3
title: Decryption/Encryption
---

# Game Files (game*.yw)

First you must understand the structure of `game*.yw` files:

* Nonce (12 bytes) — at offset `0x00`.
* MAC (16 bytes) — from `0x0C` to `0x1C` (used internally during decryption).
* Ciphertext — from offset `0x10` onward (includes MAC at the front).
* AESkey (16 bytes) - the value and location depends on the save file version - more on this further in the doc.
* Save files are encrypted using `AES-CCM` and a proprietary [*symmetric*](https://en.wikipedia.org/wiki/Symmetric-key_algorithm) (symmetric in that decryption = encryption) cipher which this doc will refer to as `YWCipher` - inspired by Togenyan's naming schema.

## v0.0 Save Files
The YW2 Demo dosen't save progress. This is a joke, ignore this (I refuse to remove it).

## v1.0 Save Files
### Detection
* In the international versions (non-JP), this format affects save files last saved in v1.0. They can be read by v2.0 game copies.
  * In JP versions, it describes any version under 2.0, as the version history is different.
  * Note that all copies of _Psychic Specters_ or _Shin'uchi_ are v2.0. A save file will have v2.0 marked on it in-game if it is.
* This can be programatically detected by checking if the fixed key AES-CCM decryption fails due to an incorrect authentication tag.

### Method
* These are first decrypted via AES-CCM (not GCM or CTR) and the AESkey is fixed in v1.0 saves: (the UTF-8 representation of "5+NI8WVq09V7LI5w").
  * Note that since the ciphertext is formatted [mac][data], you may have to rearrange it depending on your crypto lib.
* Then, you extract (store and remove) the CRC and the key from last 8 bytes from the input - where the last 8 bytes are formatted [CRC 32-bit][KEY 32-bit].
* Then you (this is REALLY important and will save you from a **major** headache) verify the data matches the CRC before running it through `0x1000` (4096) rounds of `YWCipher` and reappending the CRC and key for encryption.


## v2.0 Save Files
### Detection
* This format affects all non-v1 save files.
### Method
* Identical to that of v1 saves *but*:
  * The AESkey is no longer fixed, it is instead loaded from the (encrypted) `head.yw`.

## YWCipher
**Inputs:**

* `seed`: an integer used to initialize a deterministic pseudo-random number generator.
* `rounds`: a positive integer indicating how many shuffling iterations to perform.

#### Constructor init

* Initialise `primeList` - a list of fixed, ordered list of all prime numbers from 3-1621.
* Create a `Uint8Array` - referred to as table (with a length of 256), and initialize it with an identity mapping (table[i] = i for i from 0 to 255).
* Fill `table` so that `table[i] = i` for all `i` in `[0, 255]`
* Create a `Xorshift` instance, seeded by the given `seed`.

#### Create the substitution table

Repeat the following for each iteration `j` in `[0, rounds-1]`:

1. Generate a 16-bit random number `r` from the PRNG (`0 <= r < 65536`).

2. Extract two indices from `r`:

   * `i1 = r & 0xFF` (lower 8 bits of `r`)
   * `i2 = (r >> 8) & 0xFF` (upper 8 bits of `r`)

3. If `i1` != `i2`, proceed with the swap; otherwise, do nothing for this iteration.

4. Let `val1 = table[i1]` and `val2 = table[i2]`. These are values stored at positions `i1` and `i2`.

5. Swap the elements located at indices equal to the values found in `table` at positions `i1` and `i2`. Meaning, the swap targets `table[val1]` and `table[val2]`, where `val1 = table[i1]` and `val2 = table[i2]`:
<!-- This is ANNOYINGLY complicated to clarify lol -->
   * Temporarily store `table[val1]`
   * Set `table[val1] = table[val2]`
   * Set `table[val2]` to the temporarily stored value

* After completing all iterations, the `table` array should represent a permutation of all values `[0, 255]`, scrambled according to the process described above.
* This table is ready to be used as the cipher’s substitution mapping.

##### Important Notes

* The swap in Step  targets positions **defined by the values stored at the extracted indices**, *not* the actual indices.

##### Apply
This method performs both **encryption and decryption**, since the cipher is **symmetric** (i.e., XOR-based).

##### Parameters

* **`data`**: A `Uint8Array` of bytes to encrypt or decrypt.

##### Returns

* A new `Uint8Array` with the transformed (encrypted or decrypted) data.

###### Process

1. Initialize `ka = 0`.
2. For each index `idx` in the input:

   * If `(idx & 0xFF) === 0` (i.e., every 256 bytes), update `ka`:

     ```js
     ka = this.primes[this.table[(idx & 0xFF00) >>> 8]];
     ```

     This selects a prime number based on the high byte of the index and the scrambled table and ensures that every 256-byte "block" of data uses a new key component (`ka`), derived from the high byte of the index.
   * Compute a pseudo-random index:

     ```js
     kb = this.table[(ka * (idx + 1)) & 0xFF];
     ```
   * Apply XOR transformation:

     ```js
     out[idx] = data[idx] ^ kb;
     ```

### Xorshift PRNG <!-- such a shame that code blocks look AWFUL in headers also why is this highlighted?? oh nice it isnt rendered just syntax highlighting being weird -->

**Xorshift** is a [*deterministic*](https://en.wikipedia.org/wiki/Deterministic_system) 128‑bit [xorshift](https://en.wikipedia.org/wiki/Xorshift) PRNG class that uses a 128‑bit internal state and updates it through bitwise ops. It creates a repeatable sequence of `uint32`s and optionally supports bounded output.

### Notes

* Maintains a four‑part 128‑bit internal state.
* Uses shift and XOR operations to generate the next value.
* Deterministic: the same seed always leads to the same sequence.
* Optionally supports bounding output using modulus arithmetic.
* **Note:** this `Xorshift` uses a `0x6C078965`‑based multiplier and fixed starting words - even for good o'l `0`.

### Initialization

```js
new Xorshift(seed)
```

When a seed is provided, the generator initializes its internal state as follows:

```js
initialize(seed):
  // default state words when seed === 0
  state[0] = 0x6C078966
  state[1] = 0xDD5254A5
  state[2] = 0xB9523B81
  state[3] = 0x03DF95B3

  if (seed === 0) return // if its 0 nothing else needs to happen lol

  // otherwise, mix seed into state[0..2]
  const mult = 0x6C078965

  for (let i = 0; i < 3; i++) {
    seed ^= seed >>> 30
    seed = Math.imul(seed, mult) >>> 0 // bound to uint32
    seed = (seed + (i + 1)) >>> 0
    state[i] = seed
  }
  // state[3] remains 0x03DF95B3
```

* If the seed is zero, you get the fixed default state.
* Otherwise, each of the first three state words is derived from the seed using a 30‑bit right shift, an `imul` (integer multiplication) by `0x6C078965`, and an increment.
* The fourth state word is always `0x03DF95B3`.

The same seed always leads to the same internal state.

### next(divisor = 0)

Generates the next value in the sequence.

#### Behavior:

1. A temporary variable is created by shifting and XORing part of the current state.
2. The state is rotated forward: each part takes the value of the next.
3. The final part of the state is updated using additional XOR and shift operations.
4. The result is either:

   * The new state value (as a `uint32`), or
   * That value modulo `divisor`, if a `divisor > 0` is supplied.

This advances the generator's state and makes sure every call leads to a *new deterministic value*.

### initialize(seed)

Re‑seeds the generator, resetting the internal state in the same way as during initial construction. This allows restarting the sequence or switching to a new one deterministically.

### Internal State

The internal state consists of four 32‑bit values that together represent the generator’s full 128‑bit memory. Every call to `next()` modifies this state, and the generator's output depends entirely on it.

### Example Behavior

If seeded with the same number:

```js
A = new Xorshift(12345)
B = new Xorshift(12345)
A.next() === B.next()  // true
```

If called repeatedly:

```js
R = new Xorshift(42)
R.next()  // → some 32-bit number
R.next()  // → another number, always the same given the same seed
```

If using a divisor:

```js
R = new Xorshift(42)
R.next(10)  // always returns a value between 0 and 9
```


## Examples
### Togenyan (C++)
Note: I am *NOT* togenyan, in the credits page you should find a link to his github and the appropriate license.
* AES-CCM:<br/>

```cpp
static const int TAG_SIZE = 16;

QByteArray *CCMCipher::decrypt(const QByteArray &in)
{
    /*
     * in  : { MAC (16 byte), ciphertext (x byte) }
     * out : { plaintext (x byte) }
     */

    // { MAC, ciphertext } -> { ciphertext, MAC}
    std::string ciphertext(in.data() + TAG_SIZE, in.size() - TAG_SIZE);
    ciphertext.append(in.data(), TAG_SIZE);

    std::string out;
    try
    {
        CryptoPP::CCM< CryptoPP::AES, TAG_SIZE>::Decryption d;
        d.SetKeyWithIV((unsigned char*)this->key.data(), this->key.size(),
                       (unsigned char*)this->nonce.data(), this->nonce.size());
        d.SpecifyDataLengths(0, ciphertext.size() - TAG_SIZE, 0);

        CryptoPP::AuthenticatedDecryptionFilter df(
                    d, new CryptoPP::StringSink(out)
                    );
        CryptoPP::StringSource(ciphertext, true, new CryptoPP::Redirector(df));
    }
    catch (CryptoPP::Exception &e)
    {
        return 0;
    }
    QByteArray *result = new QByteArray(out.c_str(), in.size() - TAG_SIZE);
    return result;
}
```

* Version Detection:<br/>

```cpp
setAeskey("5+NI8WVq09V7LI5w"); // test with the hardcoded key used in v1.0 saves
if ((status = loadFile(file)) != Error::SUCCESS) { // If that fails, assume Ganso / Honke ver 2.x OR Shin'uchi

   if ((status = loadKeyFromHeadFile(file)) == Error::SUCCESS) {
     status = loadFile(file);
    }
}

if (status != Error::SUCCESS) { // if it fails
   QMessageBox::critical(this, tr("ERROR"), QString(tr("ERROR (%1)")).arg(status)); // have a tantrum
   setAeskey(prevKey); // restore the key
   return; // exit
}
```

* General Decryption Process (slightly readjusted from the original):<br/>

```cpp
SaveManager::loadFile(QString path)
{
    // assume file exists

    if (!file.open(QIODevice::ReadOnly)) { // Error handling, ignore this
        return Error::FILE_CANNOT_OPEN;
    }

    // bodydata = file

    // key
    QByteArray nonce;

    // decrypt first layer (AES CCM)
    nonce = bodydata.left(0x0C); // get the nonce
    CCMCipher myCCM(this->aeskey, nonce); // the aeskey depends, read the previous example for more info.
    QByteArray *decryptedFirst = myCCM.decrypt(bodydata.right(bodydata.size() - 0x10));
    if (!decryptedFirst) {
        return Error::DECRYPTION_CCM_FAILED;
    }

    // decrypt second layer (YWCipher)
    QByteArray ywkeyBytes = decryptedFirst->right(4);
    QByteArray *decryptedSecond = SaveManager::processYW(*decryptedFirst, false);
    delete decryptedFirst;
    if (!decryptedSecond) {
        return Error::DECRYPTION_YW_FAILED;
    }

    // strip CRC + key
    decryptedSecond->resize(decryptedSecond->size() - 8);

    // split into sections
    Error::ErrorCode status = this->parseSavedata(*decryptedSecond);

    delete decryptedSecond;
    if (status != Error::SUCCESS) {
        return status;
    }

    // loaded successfully
    this->filepath = path;
    this->nonce = nonce;
    this->ywcipherKey = ywkeyBytes;
    this->isLoaded = true;

    return Error::SUCCESS;
}
```

### n123git (me, JS)
* AES-CCM (from my save editor but without unneeded bloat):<br/>

```js

/* Note:
*  This uses SJCL (Standard Javascript Crypto Library) if it didnt - this example would be longer than my brain could comprehend
* (took embarrasingly long to find a web-compatible crypto lib that supported CCM mode).
*/

    // Patch SJCL to add toBits and fromBits
    sjcl.codec.bytes = {
      // Converts an array of bytes (0-255) into an array of uint32s (bits)
      toBits: function(bytes) {
        var out = [], i, tmp = 0;
        for (i = 0; i < bytes.length; i++) {
          // Shift tmp left by 8 bits (a byte lol) and add the current byte
          tmp = (tmp << 8) | bytes[i];
          // Every 4 bytes (32 bits), push tmp to output and reset tmp
          if ((i & 3) === 3) {  // i % 4 === 3
            out.push(tmp);
            tmp = 0;
          }
        }
        // If the bytes length is != to a multiple of 4, pad the remaining bits and push it lol
        if ((bytes.length & 3) !== 0) {
          // Shift tmp to the left to fill the remaining bits with zeros before pushing
          out.push(tmp << (8 * (4 - (bytes.length & 3))));
        }
        return out;
      },
    
      // Converts an array of uint32s (bits) back into an array of bytes (0-255)
      fromBits: function(bits) {
        var bytes = [], i, j;
        // For each 32-bit integer (uint32)
        for (i = 0; i < bits.length; i++) {
          // Extract each byte from it, starting from the most significant byte
          for (j = 3; j >= 0; j--) {
            bytes.push((bits[i] >>> (8 * j)) & 0xff);
          }
        }
        // Remove any leftover zero bytes (padding) at the end of the array
        while (bytes.length > 0 && bytes[bytes.length - 1] === 0) {
          bytes.pop();
        }
        return bytes;
      }
    };

    function aesCcmDecrypt(ciphertext, key, nonce) { 
      try {
        const keyBits = sjcl.codec.hex.toBits(key);
        const nonceBits = sjcl.codec.bytes.toBits(Array.from(nonce));

        // Extract MAC (first 16 bytes) and ciphertext (rest)
        const mac = Array.from(ciphertext.slice(0, 16));
        const ct = Array.from(ciphertext.slice(16));

        const macBits = sjcl.codec.bytes.toBits(mac);
        const ctBits = sjcl.codec.bytes.toBits(ct);
        const combinedBits = ctBits.concat(macBits);

        const decryptedBits = sjcl.mode.ccm.decrypt(new sjcl.cipher.aes(keyBits), combinedBits, nonceBits, [], 128);
        const decryptedBytes = sjcl.codec.bytes.fromBits(decryptedBits);
        return new Uint8Array(decryptedBytes);
      } catch (error) {
        console.error('AES-CCM decryption error:', error);
        return null;
      }
    }

```

* Version Detection (psuedocode this time):<br/>

```js
function masterdecrypt(savefile, head) {
    try {
      decrypt(savefile, "v1") // inconsistent spacing FTW
    } catch(error) { // if v1 decryption fails
      try {
        const key = grabkey(head) // grab key
        decrypt(savefile, "v2", key) // try again
      } catch(error) { // if it still fails
        throw new Error("v1+v2 DECRYPTION FAILED") // complain because something went horribly wrong
      }
    }
}
```

---

# Header Files (head.yw)
These are decrypted in the same way as YW1 saves. Meaning that they are decrypted as if they were a v1.0 save, but without the AES encryption at ALL, just `YWCipher`. Here is an example from Togenyan and NobodyF34R's YW1 Save Editor:

```cpp
Error::ErrorCode SaveManager::loadFile(QString path)
{
    QFile file(path);

    QDir dir(QFileInfo(path).absolutePath());

    if (!file.open(QIODevice::ReadOnly)) {
        return Error::FILE_CANNOT_OPEN;
    }
   
    QByteArray bodydata = file.readAll();
    file.close();

    // the above isn't important.

    // decrypt second layer (YWCipher)
    QByteArray ywkeyBytes = bodydata.right(4); // get the 4 bytes, not bits
    QByteArray *decryptedSecond = SaveManager::processYW(bodydata, false); // decrypt via processYW
    if (!decryptedSecond) {
        return Error::DECRYPTION_YW_FAILED;
    }

    // strip CRC + key
    decryptedSecond->resize(decryptedSecond->size() - 8); // remove the CRC+key

    // split into sections
    Error::ErrorCode status = this->parseSavedata(*decryptedSecond); // ignore this, this is unimportant for decryption, as it is for SectionID parsing, see my general.md for more info

    delete decryptedSecond;
    if (status != Error::SUCCESS) {
        return status;
    }

    // loaded successfully
    this->filepath = path;
    this->ywcipherKey = ywkeyBytes;
    this->isLoaded = true;
    if (bodydata.size() == 47556) { // Switch Saves have a length of 47556 BYTES, 3DS saves do not.
        this->isModern = true;
    } else {
        this->isModern = false;
    }

    return Error::SUCCESS;
}
```
And for `YWCipher`, the exact specifics can be found here (again from Togenyan):
```cpp
﻿#pragma execution_character_set("utf-8")

#include "ywcipher.h"

const qint32 YWCipher::oddPrimes[] = {
    3,    5,    7,   11,   13,   17,   19,   23,   29,   31,   37,   41,   43,   47,   53,   59,
   61,   67,   71,   73,   79,   83,   89,   97,  101,  103,  107,  109,  113,  127,  131,  137,
  139,  149,  151,  157,  163,  167,  173,  179,  181,  191,  193,  197,  199,  211,  223,  227,
  229,  233,  239,  241,  251,  257,  263,  269,  271,  277,  281,  283,  293,  307,  311,  313,
  317,  331,  337,  347,  349,  353,  359,  367,  373,  379,  383,  389,  397,  401,  409,  419,
  421,  431,  433,  439,  443,  449,  457,  461,  463,  467,  479,  487,  491,  499,  503,  509,
  521,  523,  541,  547,  557,  563,  569,  571,  577,  587,  593,  599,  601,  607,  613,  617,
  619,  631,  641,  643,  647,  653,  659,  661,  673,  677,  683,  691,  701,  709,  719,  727,
  733,  739,  743,  751,  757,  761,  769,  773,  787,  797,  809,  811,  821,  823,  827,  829,
  839,  853,  857,  859,  863,  877,  881,  883,  887,  907,  911,  919,  929,  937,  941,  947,
  953,  967,  971,  977,  983,  991,  997, 1009, 1013, 1019, 1021, 1031, 1033, 1039, 1049, 1051,
 1061, 1063, 1069, 1087, 1091, 1093, 1097, 1103, 1109, 1117, 1123, 1129, 1151, 1153, 1163, 1171,
 1181, 1187, 1193, 1201, 1213, 1217, 1223, 1229, 1231, 1237, 1249, 1259, 1277, 1279, 1283, 1289,
 1291, 1297, 1301, 1303, 1307, 1319, 1321, 1327, 1361, 1367, 1373, 1381, 1399, 1409, 1423, 1427,
 1429, 1433, 1439, 1447, 1451, 1453, 1459, 1471, 1481, 1483, 1487, 1489, 1493, 1499, 1511, 1523,
 1531, 1543, 1549, 1553, 1559, 1567, 1571, 1579, 1583, 1597, 1601, 1607, 1609, 1613, 1619, 1621
};

YWCipher::YWCipher(quint32 seed, int count) :
    Xorshift(seed)
{
    for (int i = 0; i < 0x100; i++) {
        this->table.append(i);
    }

    for (int i = 0; i < count; i++) {
        int r = this->next(0x10000);
        int r1 = r & 0xFF, r2 = (r >> 8) & 0xFF;
        if (r1 != r2) {
            r1 = this->table.at(r1);
            r2 = this->table.at(r2);
            this->table.swap(r1, r2);
        }
    }
}

QByteArray* YWCipher::encrypt(const QByteArray &in)
{
    int ka, kb;

    QByteArray *out = new QByteArray();
    for (QByteArray::const_iterator i = in.constBegin(); i != in.constEnd(); ++i) {
        int idx = i - in.constBegin();
        if (idx % 0x100 == 0) {
            ka = this->oddPrimes[this->table[(idx & 0xFF00) >> 8]];
        }
        kb = this->table[ka * (idx + 1) & 0xFF];
        out->append((*i) ^kb);
    }
    return out;
}

QByteArray* YWCipher::decrypt(const QByteArray &in) // YWCipher is symetric decrypt = encrypt (this didnt originally confuse me when I read through togenyans code - why would you ask?)
{
    return encrypt(in);
}
```
