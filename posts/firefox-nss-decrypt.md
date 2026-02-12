---
layout: default
title: "Deep Dive into Firefox Password Decryption via NSS3"
description: "Mozilla Firefox credentials dumping"
keywords: reverse engineering, windows api, malware, infostealer, c, windows kernel 
---

## Deep Dive into Firefox Password Decryption via NSS3

#### [?  ⁄  ?  ⁄  ?]

<br>

![logo](../assets/images/firefox-nss-decrypt/Gemini_Generated_Image_d0ejjod0ejjod0ej.png)

<br>

### Introduction

Mozilla Firefox stores user credentials securely in `logins.json`. These credentials are encrypted using **Network Security Services (NSS)**, ensuring that passwords cannot be easily extracted from disk. NSS provides a set of APIs for cryptographic operations, including encryption and decryption of sensitive data.

In this article, I focus on the technical aspects of decrypting these credentials programmatically using C. Specifically, we examine the `PK11SDR_Decrypt` function, which performs the decryption, and the data structures involved.

<br>

### SECItem: Representing Data Buffers

The primary data structure used for encrypted and decrypted data in NSS is the `SECItem` struct. It represents an arbitrary buffer with its type and length:

```c
typedef enum {
    siBuffer = 0
} SECItemType;

typedef struct SECItemStr {
    SECItemType type;
    unsigned char* data;
    unsigned int len;
} SECItem;
```

- `type` specifies the kind of data (e.g., `siBuffer`).  
- `data` points to the actual byte buffer.  
- `len` is the length of the buffer.  

`SECItem` is used both as input to encryption functions and output from decryption functions.

<br>

### PK11SDR_Decrypt: Decrypting Firefox Credentials

The `PK11SDR_Decrypt` function is responsible for decrypting credentials stored by Firefox. Its C signature is:

```c
SECStatus PK11SDR_Decrypt(
    SECItem *data,     // Input ciphertext
    SECItem *result,   // Output plaintext
    void *cx           // Optional context, usually NULL
);
```

#### How it works

1. The input `SECItem` contains the Base64-decoded encrypted data.  
2. NSS internally locates the key associated with the user’s profile.  
3. A symmetric cipher context is created to decrypt the data.  
4. Decrypted bytes are written to the output `SECItem`.  

The function returns a `SECStatus` value:

```c
typedef enum {
    SECFailure = -1,
    SECSuccess = 0
} SECStatus;
```

- `SECSuccess` indicates the operation succeeded.  
- `SECFailure` indicates an error occurred.

<br>

### Initializing NSS

Before any decryption, NSS must be initialized. This involves dynamically loading the required libraries and initializing the library with the Firefox profile path:

```c
typedef SECStatus(*NssInit)(const char* configdir);
typedef SECStatus(*Pk11SdrDecrypt)(SECItem* data, SECItem* result, void* cx);
typedef SECStatus(*NssShutdown)(void);

.
.
.

HMODULE hNss3 = LoadLibrary("nss3.dll");
NssInit NSS_Init = (NssInit)GetProcAddress(hNss3, "NSS_Init");
Pk11SdrDecrypt PK11SDR_Decrypt = (Pk11SdrDecrypt)GetProcAddress(hNss3, "PK11SDR_Decrypt");
NssShutdown NSS_Shutdown = (NssShutdown)GetProcAddress(hNss3, "NSS_Shutdown");
```

- `NSS_Init` prepares NSS to operate on the profile directory, where key material is stored.  
- `NSS_Shutdown` cleans up the library after operations are complete.

<br>

### Preparing Data for Decryption

Credentials in `logins.json` are Base64-encoded. They must be decoded before passing to `PK11SDR_Decrypt`:

```c
unsigned char* decoded;
size_t decoded_len;
base64_decode(encrypted_b64, &decoded, &decoded_len);

SECItem in;
in.type = siBuffer;
in.data = decoded;
in.len = decoded_len;
```

After decryption, the output can be accessed as:

```c
SECItem out;
if (PK11SDR_Decrypt(&in, &out, NULL) == SECSuccess) {
    printf("Decrypted: %.*s\n", (int)out.len, out.data);
}
```

<br>

### Conclusion

By understanding the `SECItem` structure and the `PK11SDR_Decrypt` function, developers can interact with Firefox’s encrypted credentials programmatically in a safe and controlled manner. Proper initialization and shutdown of NSS are critical to avoid memory leaks or inconsistent library states. This knowledge is foundational for research into browser security and automated password management systems.

<br>

[[../]](./../index.md)

<br>