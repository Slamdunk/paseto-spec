# Paseto Version 2

## Encrypt

Given a message `m`, key `k`, and optional footer `f`.

1. Before encrypting, first assert that the key being used is intended for use
   with `v2.local` tokens, and has a length of 256 bits (32 bytes). 
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set header `h` to `v2.local.`
3. Generate 24 random bytes from the OS's CSPRNG.
4. Calculate BLAKE2b of the message `m` with the output of step 2 as the key,
   with an output length of 24. This will be our nonce, `n`.
   * This step is to ensure that an RNG failure does not result in a
     nonce-misuse condition that breaks the security of our stream cipher.
5. Pack `h`, `n`, and `f` together (in that order) using 
   [PAE](Common.md#authentication-padding).
   We'll call this `preAuth`.
6. Encrypt the message using XChaCha20-Poly1305, using an AEAD interface such as
   the one provided in libsodium.
   ```
   c = crypto_aead_xchacha20poly1305_encrypt(
       message = m
       aad = preAuth
       nonce = n
       key = k
   );
   ```
7. If `f` is:
   * Empty: return h || base64url(n || c)
   * Non-empty: return h || base64url(n || c) || `.` || base64url(f)
   * ...where || means "concatenate"
   * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Decrypt

Given a message `m`, key `k`, and optional footer `f`.

1. Before decrypting, first assert that the key being used is intended for use
   with `v2.local` tokens, and has a length of 256 bits (32 bytes).
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v2.local.`, otherwise throw an
   exception. This constant will be referred to as `h`.
4. Decode the payload (`m` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
   * `n` to the leftmost 24 bytes
   * `c` to the middle remainder of the payload, excluding `n`.
5. Pack `h`, `n`, and `f` together (in that order) using
   [PAE](Common.md#authentication-padding). 
   We'll call this `preAuth`
6. Decrypt `c` using `XChaCha20-Poly1305`, store the result in `p`.
   ```
   p = crypto_aead_xchacha20poly1305_decrypt(
      ciphertext = c
      aad = preAuth
      nonce = n
      key = k
   );
   ```
7. If decryption failed, throw an exception. Otherwise, return `p`. 

## Sign

Given a message `m`, Ed25519 secret key `sk`, and
optional footer `f` (which defaults to empty string):

1. Before signing, first assert that the key being used is intended for use
   with `v2.public` tokens, and is the secret key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. Set `h` to `v2.public.`
3. Pack `h`, `m`, and `f` together using
   [PAE](Common.md#authentication-padding)
   (pre-authentication encoding). We'll call this `m2`.
4. Sign `m2` using Ed25519 `sk`. We'll call this `sig`.
   ```
   sig = crypto_sign_detached(
       message = m2,
       private_key = sk
   );
   ```
5. If `f` is:
   * Empty: return "`h` || base64url(`m` || `sig`)"
   * Non-empty: return "`h` || base64url(`m` || `sig`) || `.` || base64url(`f`)"
   * ...where || means "concatenate"
   * Note: `base64url()` means Base64url from RFC 4648 without `=` padding.

## Verify

Given a signed message `sm`, public key `pk`, and optional footer `f`
(which defaults to empty string):

1. Before verifying, first assert that the key being used is intended for use
   with `v2.public` tokens, and is the public key of the intended keypair.
   See [Algorithm Lucidity](../02-Implementation-Guide/03-Algorithm-Lucidity.md)
   for more information.
2. If `f` is not empty, implementations **MAY** verify that the value appended
   to the token matches some expected string `f`, provided they do so using a
   constant-time string compare function.
3. Verify that the message begins with `v2.public.`, otherwise throw an exception.
   This constant will be referred to as `h`.
4. Decode the payload (`sm` sans `h`, `f`, and the optional trailing period
   between `m` and `f`) from base64url to raw binary. Set:
   * `s` to the rightmost 64 bytes
   * `m` to the leftmost remainder of the payload, excluding `s`  
5. Pack `h`, `m`, and `f` together using
   [PAE](Common.md#authentication-padding).
   We'll call this `m2`.
6. Use Ed25519 to verify that the signature is valid for the message:
   ```
   valid = crypto_sign_verify_detached(
       signature = s,
       message = m2,
       public_key = pk
   );
   ```
7. If the signature is valid, return `m`. Otherwise, throw an exception.
