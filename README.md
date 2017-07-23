# Branca Specification

 Authenticated Encrypted API Tokens (IETF ChaCha20-Poly1305)

## What?

Branca allows you to generate and verify encrypted API tokens.
This specification defines the external format and encryption scheme of the
token to help interoperability userland between implementations. Branca is closely
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

Timestamp is 32 bits ie. standard big endian 4 byte UNIX timestamp. By having a
timestamp instead of expiration time enables the consuming side to decide how
long tokens are valid. You cannot accidentaly create tokens which are valid for
the next 10 years.

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

## Implementations

Currently known implementations in wild.

| Language | Author | Crypto library used |
| -------- | ------ | -------------- |
| [JavaScript](https://github.com/tuupola/branca-js) | [tuupola](https://github.com/tuupola) | [calvinmetcalf/chacha20poly1305](https://github.com/calvinmetcalf/chacha20poly1305)|
| [PHP](https://github.com/tuupola/branca-php) | [tuupola](https://github.com/tuupola) | [paragonie/sodium_compat](https://github.com/paragonie/sodium_compat) |

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email tuupola@appelsiini.net instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.