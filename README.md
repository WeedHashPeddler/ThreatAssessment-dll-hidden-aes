# ThreatAssessment.dll — Microsoft Phishing Protection's Hidden OpenSSL

Windows' own **phishing protection** silently bundles a full crypto engine. The
DLL that evaluates suspicious files for phishing threats ships OpenSSL with a
complete **AES** implementation (AES-256). 

---

## What is it?

`ThreatAssessment.dll` is a Microsoft-signed DLL (Microsoft Phishing Protection) in `System32`. It
contains OpenSSL with a full **AES** implementation (AES-256, table-based +
AES-NI assembly).

The catch: it exports only 4 functions, none of them crypto. The AES is
compiled in but **not exported**, so `GetProcAddress` can't find it — that's
why I located it by offset and call `base + offset` directly.

## The method

| Function              | Offset (RVA) |
|-----------------------|--------------|
| `AES_set_encrypt_key` | `0x24f5a0`   |
| `AES_set_decrypt_key` | `0x24f320`   |
| `AES_encrypt`         | `0x24ed10`   |
| `AES_decrypt`         | `0x24e720`   |

```c
HMODULE h = LoadLibraryA("ThreatAssessment.dll");
AES_setkey_fn set_enc = (AES_setkey_fn)((unsigned char*)h + 0x24f5a0);
AES_fn       aes_enc = (AES_fn)((unsigned char*)h + 0x24ed10);

AES_KEY k;
set_enc(key, 256, &k);        // DLL: expand the 256-bit key
aes_enc(data, out, &k);       // DLL: encrypt a 16-byte block
```

## Usage

Place `ThreatAssessment.dll` in the same folder as the executables.

```bash
encrypt.exe file.txt           # random 256-bit key -> file.txt.enc + .key
decrypt.exe file.txt.enc <key> # -> file.txt.dec
```

## Verification

```
[+] custom key: AES_256_CUSTOM_KEY_1234567890_!!
[+] calc.exe (49152 bytes) -> padded 49168 -> encrypted -> decrypted
[+] RESULT: AES-256 ENCRYPT+DECRYPT BYTE-IDENTICAL (verified)
```

Also byte-compatible with standard AES (FIPS-197 vector matched).

## Files

```
ThreatAssessment.dll  — the Microsoft-signed crypto DLL (required, same folder)
encrypt.c             — encryptor (random secure key, AES-256)
decrypt.c             — decryptor (AES-256)
encrypt.exe / decrypt.exe — compiled
```
