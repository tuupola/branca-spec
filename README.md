# Branca Token

Authenticated and encrypted API tokens using modern crypto.

## What?

Branca is a secure, easy to use token format which makes it hard to shoot yourself in the foot. It uses IETF XChaCha20-Poly1305 AEAD symmetric encryption to create encrypted and tamperproof tokens. The payload itself is an arbitrary sequence of bytes. You can use for example a JSON object, plain text string or even binary data serialized by [MessagePack](http://msgpack.org/) or [Protocol Buffers](https://developers.google.com/protocol-buffers/).

Although not a goal, it is possible to use [Branca as an alternative to JWT](https://appelsiini.net/2017/branca-alternative-to-jwt/). Also see [getting started](https://branca.io/) instructions.

This specification defines the external format and encryption scheme of the token to help developers create their own implementations. Branca is closely based on the [Fernet specification](https://github.com/fernet/spec/blob/master/Spec.md).

## Design Goals

1. Secure
2. Easy to implement
3. Small token size

## Token Format

A Branca token consists of a header, ciphertext and an authentication tag. The header consists of a version, timestamp and nonce. Putting them all together we get following structure:

```
Version (1B) || Timestamp (4B) || Nonce (24B) || Ciphertext (*B) || Tag (16B)
```

String representation of the above binary token must use base62 encoding with the following character set.


```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

### Version

Version is 8 bits ie. one byte. Currently, the only version is `0xBA`. This is a magic byte which you can use to quickly identify a given token. Version number guarantees the token format and encryption algorithm.

### Timestamp

Timestamp is 32 bits ie. unsigned big-endian 4 byte UNIX timestamp. By having a timestamp instead of expiration time enables the consuming side to decide how long tokens are valid. You cannot accidentally create tokens which are valid for the next 10 years.

Storing the timestamp as an unsigned integer allows us to avoid the 2038 problem. Unsigned integer overflow will happen in the year 2106. Possible values are 0 - 4294967295.

### Key

This is a symmetric key of 32 arbitrary bytes. The key can be randomly generated using a CSPRNG or created during a key-exchange for example.

It is the responsibility of the user of Branca, to ensure the strength and confidentiality of the secret key used. See [libsodium](https://libsodium.gitbook.io/doc/quickstart#what-is-the-difference-between-a-secret-key-and-a-password) for more information on the difference between a password and a secret key.

### Nonce

Nonce is 192 bits ie. 24 bytes. It should be cryptographically secure random bytes. A nonce should never be used more than once with the same secret key between different payloads. It should be generated automatically by the implementing library. Allowing end users to provide their own nonce is a foot gun.

### Payload

Payload is an arbitrary sequence of bytes of any length.

### Ciphertext

Payload is encrypted and authenticated using [IETF XChaCha20-Poly1305](https://download.libsodium.org/doc/secret-key_cryptography/xchacha20-poly1305_construction.html). Note that this is [Authenticated Encryption with Additional Data (AEAD)](https://tools.ietf.org/html/rfc7539#section-2.8) where the header part of the token is the additional data. This means the data in the header (version, timestamp and nonce) is not encrypted, it is only authenticated. In laymans terms, header can be seen but it cannot be tampered with.

### Tag

The authentication tag is 128 bits ie. 16 bytes. This is the
[Poly1305](https://en.wikipedia.org/wiki/Poly1305) message authentication code. It is used to make sure that the payload, as well as the non-encrypted header have not been tampered with.

## Working With Tokens

Instructions below assume your crypto library supports combined mode. In combined mode, the authentication tag and the encrypted message are stored together. If your crypto library does not provide combined mode, then the `tag` is last 16 bytes of the `ciphertext|tag` combination.

### Generating a Token

Given a 256 bit ie. 32 byte secret `key` and an arbitrary `payload`, generate a token with the following steps in order:

1. Generate a 24 byte cryptographically secure random `nonce`.
2. If no `timestamp` is provided, use the current unixtime.
3. Construct the `header` by concatenating `version`, `timestamp` and `nonce`.
4. Encrypt the payload with IETF XChaCha20-Poly1305 AEAD with the `key`. Use `header` as the additional data for the AEAD.
5. Concatenate the `header` and the returned `ciphertext|tag` combination from step 4.
6. Base62 encode the entire token.

### Verifying a Token

Given a 256 bit ie. 32 byte secret `key` and a `token` to verify that the `token` is valid and recover the original unencrypted `payload`, perform the following steps, in order.

1. Base62 decode the token.
2. Make sure the first byte of the decoded token is `0xBA`.
3. Extract the `header` ie. the first 29 bytes from the decoded token.
4. Extract the `nonce` ie. the last 24 bytes from the `header`.
5. Extract the `timestamp` ie. bytes 2 to 5 from the `header`.
6. Extract `ciphertext|tag` combination ie. everything starting from byte 30.
7. Decrypt and verify the `ciphertext|tag` combination with IETF XChaCha20-Poly1305 AEAD using the secret `key` and `nonce`. Use `header` as the additional data for AEAD.

### Working With the Timestamp

Optionally the implementing library may use the `timestamp` for additional verification. For example the library could provide a non-negative `ttl` parameter which is used to check if token is expired by adding the `ttl` to the `timestamp` and comparing the result with the current unixtime. Any optional timestamp check must happen as the last step **after** decrypting and verifying the token. Further, the implementing library must ensure adding `ttl` to `timestamp` does not overflow 4294967295.

## Libraries

Currently known implementations in the wild.


| Language | Repository | License | Crypto library used |
| -------- | ---------- | ------- | ------------------- |
| Clojure | [miikka/clj-branca](https://sr.ht/~miikka/clj-branca/) | EPL-2.0 | [terl/lazysodium-java](https://github.com/terl/lazysodium-java) |
| .NET | [AmanAgnihotri/Branca](https://github.com/AmanAgnihotri/Branca) | LGPL-3.0-or-later | [NaCl.Core](https://github.com/idaviddesmet/NaCl.Core) |
| .NET | [scottbrady91/IdentityModel](https://github.com/scottbrady91/IdentityModel) | Apache-2.0 | [Bouncy Castle](https://www.bouncycastle.org/csharp/index.html) |
| Elixir | [tuupola/branca-elixir](https://github.com/tuupola/branca-elixir) | MIT | [ArteMisc/libsalty](https://github.com/ArteMisc/libsalty) |
| Erlang | [1ma/branca-erl](https://github.com/1ma/branca-erl) | MIT | [jedisct1/libsodium](https://github.com/jedisct1/libsodium) |
| Go | [essentialkaos/branca](https://github.com/essentialkaos/branca) | Apache-2.0 | [golang/crypto](https://github.com/golang/crypto)
| Go | [hako/branca](https://github.com/hako/branca) | MIT | [golang/crypto](https://github.com/golang/crypto)
| Go | [juranki/branca](https://github.com/juranki/branca) | MIT | [golang/crypto](https://github.com/golang/crypto)
| Java | [Kowalski-IO/branca](https://github.com/Kowalski-IO/branca) | MIT | [Bouncy Castle](https://www.bouncycastle.org/java.html) |
| Java | [bjoernw/jbranca](https://github.com/bjoernw/jbranca) | Apache-2.0 | [Bouncy Castle](https://www.bouncycastle.org/java.html) |
| JavaScript | [tuupola/branca-js](https://github.com/tuupola/branca-js) | MIT | [jedisct1/libsodium.js](https://github.com/jedisct1/libsodium.js) |
| Kotlin | [petersamokhin/kbranca](https://github.com/petersamokhin/kbranca) | Apache-2.0 | [Bouncy Castle](https://www.bouncycastle.org/java.html) |
| PHP | [tuupola/branca-php](https://github.com/tuupola/branca-php) | MIT | [jedisct1/libsodium](https://github.com/jedisct1/libsodium) |
| Python | [tuupola/branca-python](https://github.com/tuupola/branca-python) | MIT | [jedisct1/libsodium](https://github.com/jedisct1/libsodium) |
| Ruby | [thadeu/branca-ruby](https://github.com/thadeu/branca-ruby) | MIT | [RubyCrypto/rbnacl](https://github.com/RubyCrypto/rbnacl)
| Ruby | [crossoverhealth/branca](https://github.com/crossoverhealth/branca) | MIT | [RubyCrypto/rbnacl](https://github.com/RubyCrypto/rbnacl)
| Rust | [return/branca](https://github.com/return/branca) | MIT | [brycx/orion](https://github.com/brycx/orion)

## Acceptance Test Vectors

All test vectors can be found [here](https://github.com/tuupola/branca-spec/blob/master/test_vectors.json).

## Similar Projects

* [PASETO](https://github.com/paragonie/paseto) ie. Platform-Agnostic Security Tokens.
* [Fernet](https://github.com/fernet) which provides AES 128 in CBC mode tokens.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
