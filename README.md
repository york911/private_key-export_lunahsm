# private_key-export_lunahsm

This document is a step-by-step guide for exporting a Private RSA key from the Luna HSM using an external AES key. The instructions provided are intended to ensure a secure and efficient key export process.

## Use case

Securely share an RSA Private Key created on the Luna HSM with a third party.

## Solution

The third-party will generate an AES 256 key and we will use the Public Key to wrap the AES key, then share it securely and unwrapped inside the HSM.

Once unwrap the AES key inside the HSM, we can use that AES key to wrap the Private Key outside the HSM.

Only the third party who created the AES key would be able to unwrap the Private Key in clear.

## Process:

- Key export partition setup

  - login as PO
  - partition changepolicy -policy 0 -value 0
  - partition changepolicy -policy 1 -value 1

- Openssl AES key creation (third-party side)
  ```bash
  openssl rand -out aeskey.bin 32
  ```
- Luna client commands CMU RSA creation
  ```bash
  ./cmu generatekeypair -keyType=RSA -modulusBits=2048 -labelPublic=publickey -labelPrivate=privatekey -encrypt=1 -decrypt=1 -sign=1 -verify=1 -wrap=1 -unwrap=1 -extractable=1 -mech=aux
  ```
- Public Key Export - CMU
  ```bash
  sudo ./cmu export -key -handle=412 -outputfile=publickeyout.key
  ```
- Wrapping AES key with PublicKey (third-party side)
  ```bash
  openssl pkeyutl -encrypt -in aeskey.bin -out aes-key-wrapped.bin -inkey publickeyout.key -pubin -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256 -pkeyopt rsa_mgf1_md:sha256
  ```
- CKDEMO to unwrap AES key with Private Key
  ( 1) Open Session
  ( 3) Login
  Crypto Officer [1]
  (98) Options
  17 - OAEP Hash Params
  0 - Finished
  (61) Unwrap key
  [26]RSA_OAEP
  [3]SHA256
  [0 for none]
  AES[9]
  1 for the attribute
  Handle ID for the Private Key
  32 bites of the AES key
  aes-key-wrapped.bin
- Wrap Private Key with AES key
  ( 1) Open Session
  ( 3) Login
  Crypto Officer [1]
  (60) Wrap key
  [31]AES-KWP
  1 = yes for external IV
  A65959A6 for the IV in hex format of the AES Key
  Handle ID AES key
  Handle ID Private key
- Unwrapped.key with the AES key (openssl 3.x)
  ```bash
  openssl enc -d -id-aes256-wrap-pad -K "$(xxd -p < aeskey.bin | tr -d '\n')" -iv  A65959A6 -in wrapped.key -out HSM_private_clear.key
  openssl pkcs8 -in HSM_private_clear.key -inform DER -topk8 -out HSM_private_clear.pem -nocrypt

  ```
- Test Private key successfully shared.
  - We will use a public key on the HSM side to encrypt a test file
  - CKDEMO
    ( 1) Open Session
    ( 3) Login
    Crypto Officer [1]
    (40) Encrypt file
    [34] RSA-PKCS
    Handle ID Public Key
  - we will share the ENCRYPT file to customer and they will use the unwrapped private to decrypt that ENCRYPT file
  ```bash
  openssl pkeyutl -decrypt -inkey HSM_private_clear.pem -in ENCRYPT.BIN -out decryptedfile.txt
  ```
