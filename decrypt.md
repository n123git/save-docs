---
parent: Save Docs!
layout: default
nav_order: 3
title: Decryption/Encryption
---

# Game Files (game*.yw)

## v0.0 Save Files
The YW2 Demo dosen't save progress. This is a joke, ignore this (I refuse to remove it).

## v1.0 Save Files
In the international versions, this format affects save files last saved in v1.0. They can be read by v2.0 game copies. In the JP versions, it describes any version under 2.0, as the version history is different. Note that all copies of _Psychic Specters_ or _Shin'uchi_ are v2.0 ignoring version. A save file will have v2.0 marked on it in-game if it isn't.
* These are first decrypted via a slighly non-standard AES-CCM (not GCM or plain CTR). The aeskey is fixed in v1.0 saves: in UTF-8 its "5+NI8WVq09V7LI5w". Then it uses a proprietary cipher, which I refer to as "YWCipher" inspired by Togenyan's naming schema. After that the CRC and key can be stripped. Here is a C++ demonstration from Togenyan's editor:

```cpp
#pragma execution_character_set("utf-8")

#include "ccmcipher.h"

static const int TAG_SIZE = 16;

CCMCipher::CCMCipher(const QByteArray &key, const QByteArray &nonce) :
    key(key),
    nonce(nonce)
{

}

QByteArray *CCMCipher::encrypt(const QByteArray &in)
{
    /*
     * in  : { plaintext (x byte) }
     * out : { MAC (16 byte), ciphertext (x byte) }
     */
    std::string plaintext(in.data(), in.size());
    std::string out;
    try
    {
        CryptoPP::CCM< CryptoPP::AES, TAG_SIZE>::Encryption e;
        e.SetKeyWithIV((unsigned char*)this->key.data(), this->key.size(),
                       (unsigned char*)this->nonce.data(), this->nonce.size());
        e.SpecifyDataLengths(0, plaintext.size(), 0);

        /*
         *  StringSource destroys AuthenticatedEncryptionFilter and StringSink
         *  when it is destroyed. so no need to delete them.
         */
        CryptoPP::StringSource(plaintext, true,
                               new CryptoPP::AuthenticatedEncryptionFilter(
                                   e, new CryptoPP::StringSink(out)
                                   )
                               );
    }
    catch (CryptoPP::Exception &e)
    {
        return 0;
    }
    QByteArray *result = new QByteArray(out.c_str(), in.size());
    result->prepend(out.c_str() + in.size(), TAG_SIZE);
    return result;
}

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

## v2.0 Save Files
This format affects save files last saved in v2.0+ (high versions exist due to JP version history). Note that all copies of _Psychic Specters_ or _Shin'uchi_ (regardless of update) are v2.0. A save file will have v2.0 marked on it in-game if it is. The main difference is that the AESkey is no longer fixed, it is instead loaded from the `head.yw`. It is ...... ... . .. .LOREM IPSUM DOLAR SIT AMET {ciphertext, CRC value of ciphertext, encryption key}. First, it extracts the last 8 bytes:
* 4 bytes CRC32 of the ciphertext
* 4 bytes encryption key
Then it removes/strips them as they are no longer important. Verifies the integrity of the ciphertext by checking that its calculated CRC matches the given CRC. It then uses `YWCipher` to decrypt it (using the key) before it appends the original CRC + key (the last 8 bytes of the input) back into the decrypted data. Where it then reads a uint32 (32-bit unsigned integer) from the decrypted file starting at offset `0x0C` (12). Which it uses as a seed for XORshift PRNG to generate 16 bytes continuously until it gets a 128-bit AES key.



Here is a slightly readjusted snippet from Togenyan's save editor, the appropriate license is placed next to this `.md`. This snippet detects which version the save file is, and adjusts it accordingly.
```cpp
if (encrypted) {  // Is it an encrypted save
        this->mgr->setAeskey("5+NI8WVq09V7LI5w"); // test with the hardcoded key used in v1.0 saves
        if ((status = this->mgr->loadFile(file)) != Error::SUCCESS) {
            // If that fails, assume Ganso / Honke ver 2.x OR Shin'uchi
            if ((status = this->mgr->loadKeyFromHeadFile(file)) == Error::SUCCESS) {
                status = this->mgr->loadFile(file);
            }
        }
    } else { // If decrypted
        status = this->mgr->loadDecryptedFile(file); // just edit it
    }
    if (status != Error::SUCCESS) { // if it fails
        QMessageBox::critical(this, tr("ERROR"), QString(tr("ERROR (%1)")).arg(status)); // have a tantrum
        this->mgr->setAeskey(prevKey); // restore the key
        return; // exit
    }
```

Here is an example depicting the general decryption process (again from Togenyan's save editor, but slightly readjusted. MIT license is goated). 

```cpp
Error::ErrorCode SaveManager::loadFile(QString path)
{
    QFile file(path);

    if (!file.open(QIODevice::ReadOnly)) { // Error handling, ignore this
        return Error::FILE_CANNOT_OPEN;
    }

    QByteArray bodydata = file.readAll(); // get the file.... not really complicated
    file.close();

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
