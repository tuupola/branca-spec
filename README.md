# Branca Specification

Authenticated Encrypted API Tokens (IETF ChaCha20-Poly1305).

## What?

Branca allows you to generate and verify encrypted API tokens.
This specification defines the external format and encryption scheme of the
token to help interoperability between userland implementations. Branca is closely
based on [Fernet specification](https://github.com/fernet/spec/blob/master/Spec.md).

## Design Goals

1. Secure
2. Easy to implement
3. Small token size

We need a secure easy to use token format which makes it hard to
shoot yourself in the foot.

## Token Format

Branca token consists of header, ciphertext and an authentication tag. Header
consists of version, timestamp and nonce. Putting them all together we get
following structure.

```
Version || Timestamp || Nonce || Ciphertext || Tag
```

String representation of the above binary token must use base62 encoding with
the following character set.


```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxy
```

### Version

Version is 8 bits ie. one byte. Currently the only version is `0xBA`. This is a
magic byte which you can use to quickly identify a given token. Version number
guarantees the token format and encryption algorithm.

### Timestamp

Timestamp is 32 bits ie. unsigned big endian 4 byte UNIX timestamp. By having a
timestamp instead of expiration time enables the consuming side to decide how
long tokens are valid. You cannot accidentaly create tokens which are valid for
the next 10 years.

Storing timestamp as unsigned integer allows us to avoid 2038 problem. Unsigned
integer overflow will happen in year 2106.

### Nonce

Nonce is 96 bits ie. 12 bytes. These should be cryptographically secure random
bytes and never reused between tokens.

### Ciphertext

Payload is encrypted and authenticated using [IETF ChaCha20-Poly1305](https://tools.ietf.org/html/rfc7539).
Note that this is [Authenticated Encryption with Additional Data (AEAD)](https://tools.ietf.org/html/rfc7539#section-2.8) where the
he header part of the token is the additional data. This means the data in the
header (version, timestamp and nonce) is not encrypted, it is only
authenticated. In laymans terms, header can be seen but it cannot be tampered.

### Tag

The authentication tag is 128 bits ie. 16 bytes. This is the
[Poly1305](https://en.wikipedia.org/wiki/Poly1305) message authentication
code. It is used to make sure that the message, as well as the
non-encrypted header has not been tampered with.

NOTE! Some crypto libraries such as Libsodium offer [combined mode](https://download.libsodium.org/doc/secret-key_cryptography/ietf_chacha20-poly1305_construction.html) where the authentication tag and the encrypted message are
stored together. You do not need to care about separating the tag from ciphertext.
This is usually what you want to use.

## Generating a Token

Given a 256 bit ie. 32  byte secret key and an arbitrary message, generate a branca token with the following steps, in order:

1. Generate a cryptocraphically secure `nonce`.
2. If user has not provided `timestamp` record the current unixtime.
3. Construct the `header` by concatenating `version`, `timestamp` and `nonce`.
4. Encrypt the user given message with IETF ChaCha20-Poly1305 AEAD with user
   provided secret `key`. Use `header` as the additional data for AEAD.
5. Concatenate `header` and returned `ciphertext` and `tag` from step 4 together.
6. Base62 encode the entire token.

## Verifying a Token

Given a 256 bit ie. 32  byte secret key a token, to verify that the token is valid and recover the original unencrypted message, perform the following steps, in order:

1. Base62 decode the token.
2. Ensure the first byte of the token is `0xBA`.
3. Extract the `header` ie. the first 17 bytes from the binary token.
4. Extract the `nonce` ie. the last 12 bytes from the `header`.
5. Decrypt and verify the with IETF ChaCha20-Poly1305 AEAD with user
   provided secret `key` and `nonce` from previous step. Use `header` as the additional data for AEAD.
6. If the user has specified a maximum age (or "time-to-live") for the token, ensure the recorded timestamp is not too far in the past.

## Libraries

Currently known implementations in the wild.

| Language | Author | Crypto library used |
| -------- | ------ | -------------- |
| [JavaScript](https://github.com/tuupola/branca-js) | [tuupola](https://github.com/tuupola) | [calvinmetcalf/chacha20poly1305](https://github.com/calvinmetcalf/chacha20poly1305)|
| [PHP](https://github.com/tuupola/branca-php) | [tuupola](https://github.com/tuupola) | [paragonie/sodium_compat](https://github.com/paragonie/sodium_compat) |

## Acceptance Test Vectors

WIP. In the meanwhile see [JavaScript](https://github.com/tuupola/branca-js/blob/master/test.js) and [PHP](https://github.com/tuupola/branca-php/blob/master/tests/BrancaTest.php) example tests.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.