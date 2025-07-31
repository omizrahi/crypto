# crypto

## Purpose

This library provides a C++ implementation of common cryptographic functionality, designed for integration into the TRII C++ project. Its main goal is to separate cryptographic concerns from telemetry decoding and frame reconstruction logic, offering a minimal API for cryptographic operations.

## Layout

- `include/crypto/` — Public API headers
- `src/cipher` — Implementation of cipher-related algorithms and operations, including encryption and decryption.
- `src/key` — Implementation of key generation, validation, and management for cryptographic processes.
- `test/` — Unit tests

## Usage

The simplest way to construct and use a `Cipher` object is via the factory functions defined in `crypto/cipherfactory.h`.

### Creating a Key

Use `makeKey` to generate a key from a hexadecimal string:

```cpp
#include <crypto/key.h>

auto keyResult = Crypto::makeKey<128>("e03bb85b42563d6aed5f45cca57eb36d");

if (!keyResult.hasValue())
{
   std::cerr << "Failed to generate key: " << keyResult.error() << '\n';
}
```

- The template parameter (`<128>`) specifies the key size in bits.

- The function returns a `Result<Key<128>>`.

- If `hasValue()` is true, the key was successfully created and can be accessed via `keyResult.value()`.

A failed keyResult can occur because of a hex string size mismatch or invalid characters being provided in the hex string.

### Creating a Cipher

Once a key is available, create a Cipher. A Cipher can be created for encryption or decryption through the relevant factory function:

##### Encryption Cipher Creation
```cpp
#include <crypto/cipherfactory.h>

Crypto::Cipher cipher = Crypto::makeEncryptionCipher<AES>(keyResult.value());
```

##### Decryption Cipher Creation
```cpp
#include <crypto/cipherfactory.h>

Crypto::Cipher cipher = Crypto::makeDecryptionCipher<AES>(keyResult.value());
```

This sets up a cipher for encryption or decryption using AES in CFB1 mode by default.

### Processing a Payload

To encrypt or decrypt data in place, call process() on the cipher with a Payload. A Payload is defined as a block of ciphertext (for decryption) or plaintext (for encryption) in byte representation.

```cpp
// Create a Payload containing encrypted or plain bytes
Crypto::Payload payload{/* ... */};

// Process (encrypt or decrypt) the payload in place
cipher.process(payload);
```

Note that cipher.process() can throw a CipherSessionInitialisationError if the CipherSession cannot be initialised correctly (ie: if the Payload is not large enough to create an Initialisation Vector). Therefore, it is recommended to catch the exception as so:

```cpp
try
{
   cipher.process(payload);
}
catch (const CipherSessionInitialisationError& ex)
{
   std::cerr << "Failed to initialise the Cipher: " << ex.what() << '\n';
   // Handle the intialisation error appropriately.
}
```


### Stream Processing
For streaming scenarios, repeated calls to process() will continue the encryption or decryption using internal feedback from the cipher session. This allows each payload to be treated as part of a continuous ciphertext or plaintext stream:

```cpp
// Simulated source of incoming payloads (e.g., from a network)
PayloadSource source{};

// Continuously process each payload as it becomes available
while (source.hasMorePayloads())
{
   auto& payload = source.getPayload();
   cipher.process(payload); 
}
```

The payload is now ready for downstream operations such as decoding, storage, or transmission. Note that the first 128 bits of the first payload are extracted and used to initialise the cipher's Initialisation Vector. 

### Null Cipher

The `NullCipher` is a cipher implementation that performs no encryption or decryption; it processes data by leaving it unchanged.

In applications where encryption might be conditionally applied, using a `NullCipher` avoids the need to scatter conditional statements throughout the code.

A NullCipher can be created via the `makeNullCipher` factory function defined in `crypto/cipherfactory.h`

```cpp
#include <crypto/cipherfactory.h>

Crypto::Cipher cipher = Crypto::makeNullCipher();
```

### Storing Ciphers in Containers

`Cipher` objects can be stored in standard containers such as `std::vector` or `std::unordered_map` as so:

```cpp
auto encryptKey = makeKey("E03BB85B42563D6AED5F45CCA57EB36D");
auto decryptKey = makeKey("1234567890ABCDEF1234567890ABCDEF");

std::unordered_map<std::string, Cipher> cipherMap {};
cipherMap.emplace("Encryptor", makeEncryptionCipher<AES>(encryptKey.value()));
cipherMap.emplace("Decryptor", makeDecryptionCipher<AES>(decryptKey.value()));
cipherMap.emplace("Passthrough", makeNullCipher());
```

A generic run function can then be written to dispatch on the correct Cipher given the map's cipherType key as so:

```cpp
void runCipher(std::string_view cipherType, Payload& payload)
{
  cipherMap.at(cipherType.data()).process(payload);
}

runCipher("Decryptor", payload);
```

### Advanced Usage

While the recommended approach is to use the factory functions to construct Cipher instances, users can manually construct and compose the underlying components.

#### Create a BlockCipher Manually

```cpp
#include <crypto/key.h>
#include <crypto/blockcipher.h>

Crypto::Key<128> key = /* your key */;

using BlockCipherType = Crypto::BlockCipher<AES, 128>;
BlockCipherType blockCipher{key};
```

This directly instantiates an AES block cipher using the provided key. A Block of type BlockCipherType::BlockType can directly be processed through the blockCipher as so if required. The Block is process in-place.

```cpp

blockCipher.process(block);
```

#### Create a StreamSession

```cpp
#include <crypto/ciphersession.h>

CipherSession<BlockCipherType, CFB1, Decrypt> cipherSession{};
```
This creates a CFB1 session configured for decryption. For an encryption session, substitute Encrypt for Decrypt as the third template parameter.

#### Process a payload with the CipherSession

```cpp
Crypto::Payload payload{/* ... */};

cipherSession(blockCipher, payload);
```

This will encrypt or decrypt the payload in place utilising the blockCipher. The CipherSession can be reused with multiple payloads enabling a stream like encryption or decryption session.
