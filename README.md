# JustTransfer Whitepaper

This page describes the security design of JustTransfer, which aims to provide an end-to-end secure account and link transfer service for large files.

## Table of Contents

TODO

## Threat Model

The threat model of JustTransfer includes the following assumptions:

- Active Adversary: The adversary can actively intercept, modify, and inject messages between the client and the server. They may also attempt to impersonate either party to gain unauthorized access.
- Server honest-but-curious: The server is assumed to be honest in following the protocol but may try to learn sensitive information from the data it processes. It will not intentionally deviate from the protocol but may attempt to analyze the data for insights.
- Client-side Security: The client is responsible for securely handling sensitive information, such as passwords and cryptographic keys. The client is assumed to be secure and not compromised by malware or other attacks.

## Security Level

The following key sizes and parameters are used in JustTransfer:

- Symmetric Key Size: 256 bits
- Asymmetric Key Size: 512 bits for elliptic curve cryptography (ECC)
- Hash Function Output Size: 512 bits

TODO check with current implem

## Cryptographic Primitives

JustTransfer utilizes the following cryptographic primitives:

- `OPAQUE`: An asymmetric password-authenticated key exchange protocol that allows secure authentication without revealing the password to the server.
- `Argon2`: A memory-hard password hashing function used to derive keys from passwords.
- `X25519`, `XSalsa20` and `Poly1305`: A suite of cryptographic algorithms used to provide hybrid encryption. `X25519` is used for key exchange, `XSalsa20` for symmetric encryption, and `Poly1305` for the MAC. This cipher suite is used to encrypt the filename and file in account transfer.
- `Ed25519ph`: A digital signature scheme used for signing messages and verifying signatures to ensure authenticity and integrity.
- `AEGIS-256`: An AEAD cipher used for encrypting and authenticating filenames and file metadata (used to encrypt filename in link transfer).
- `XSalsa20-Poly1305`: A stream cipher combined with a MAC for authenticated encryption of keys and messages (used to encrypt private keys).
- `XChacha20-Poly1305`: A stream cipher combined with a MAC for authenticated encryption of keys and messages (used to encrypt chunk in link transfer).

TODO explain what used for what

## OPAQUE Register

The OPAQUE registration process consists of the following steps:

1. The client initiates the OPAQUE registration by sending the username and the result of `OPAQUE_{ClientRegistration}` to the server.

$$
client\_registration\_start\_result = OPAQUE_{ClientRegistration}(password)
$$

2. The server computes the OPAQUE registration start result using the username and the client's registration start result, and sends it back to the client.

$$
server\_registration\_start\_result  = OPAQUE_{ServerRegistration}(username, client\_registration\_start\_result)
$$

3. The client computes the OPAQUE registration finish result using the password and the server's registration start result, which gives the export key.

$$
client\_registration\_finish\_result = OPAQUE_{ClientRegistrationFinish}(password, server\_registration\_start\_result)
$$

$$
export_key = client\_registration\_finish\_result.export\_key
$$

4. The client sends the OPAQUE registration finish result to the server. The server computes the OPAQUE registration finish result, which gives the `password_file` that will be stored with username for future authentication. The client can also send other data to the server at this step, which depends on the type of registration (account or link).

$$
password\_file  = OPAQUE_{ServerRegistrationFinish}(client\_registration\_finish\_result)
$$

### Message Flow Summary

```
  Client                                     Server
    |                                          |
    | -- username, client_registration_start ->|
    |                                          |
    |<------- server_registration_start -------|
    |                                          |
    | ------ client_registration_finish ------>|
    |                                          |
    |
    ▼
export_key
```

TODO

## OPAQUE Login

The process of OPAQUE login

1. The client initiates the OPAQUE login by sending the username and the result of `OPAQUE_{ClientLogin}` to the server.

$$
client\_login\_start\_result = OPAQUE_{ClientLogin}(password)
$$

2. The server computes the OPAQUE login start result using the username, the client's login start result, and the `password_file` stored for the username, and sends it back to the client.

$$
server\_login\_start\_result  = OPAQUE_{ServerLogin}(username, client\_login\_start\_result, password\_file)
$$

3. The client computes the OPAQUE login finish result using the password and the server's login start result, which gives the export key.

$$
client\_login\_finish\_result = OPAQUE_{ClientLoginFinish}(password, server\_login\_start\_result)
$$

$$
export_key = client\_login\_finish\_result.export\_key\_key
$$

4. The client sends the OPAQUE login finish result to the server. The server computes the OPAQUE login finish result.

$$
server\_login\_finish\_result = OPAQUE_{ServerLoginFinish}(client\_login\_finish\_result)
$$

### Message Flow Summary

```
  Client                              Server
    |                                   |
    | -- username, client_login_start ->|
    |                                   |
    |<------- server_login_start -------|
    |                                   |
    | ------ client_login_finish ------>|
    |                                   |
    |
    ▼
export_key
```

## Link Transfer

The link transfer allows a user to share a file to other users by creating a link. As link transfer does not require the user to have an account, the transfer are repudiable and the security relies on the secrecy of the password used in the OPAQUE registration.

### Link Transfer Creation

To create a link transfer, the client performs the following steps:

1. The client performs a OPAQUE registration with the server, and sends the following data to the server at the end of the OPAQUE registration:

- ID of the transfer, which is provided by the server with the server registration start result
- The encrypted filename, which is encrypted using TODO
- The nonce of the encrypted filename, which is generated randomly
- Header, which is used to encrypt the file.
- Max download times of the transfer
- Lifetime of the transfer
- Creation time of the transfer
- The file size of the transfer

2. The server stores the data and return the upload URLs to the client, which the client can use to upload the file in chunks.
3. The client uploads the file in chunks to the upload URLs, and each chunk is encrypted using TODO.
4. The client terminates the transfer by sending the `etags` of the chunks to the server, which are used to finish the transfer upload.
5. The client displays the link to the user, using the transfer ID.

### Link Transfer Access

To access a link transfer, the client performs the following steps:

1. The client performs a OPAQUE login with the server, using the ID of the transfer as the username.
2. The client requests the transfer metadata from the server, which includes the following data:

- ID of the transfer
- The encrypted filename
- The nonce of the encrypted filename
- The file ID of the transfer
- The header of the transfer
- The max download times of the transfer
- The lifetime of the transfer
- The creation time of the transfer
- The current download times of the transfer
- The file size of the transfer
- The chunk size of the transfer

3. The client decrypts the filename using the encrypted filename and the nonce.
4. The client requests a download URL from the server, which return a presigned URL for downloading the file.
5. The client downloads the file in chunks using the download URL, and each chunk is decrypted using TODO.

### Key Management

The keys used in link transfer are obtained as follows:

1. The `export_key` is obtained from a successful OPAQUE registration or login, which is a 512-bit key.
2. `key_filename` is obtained using the first 256 bits of the `export_key`
3. `key_file` is obtained using the last 256 bits of the `export_key`

### Nonce Management

The following nonces are used in link transfer:

- `nonce_filename`: A 192-bit nonce generated randomly for encrypting the filename.
- `nonce_chunk`: A 192-bit nonce generated randomly for encrypting each chunk of the file.

TODO nonce calculation and management

## Account Transfer

The account transfer allows a user to send a file to another user. The transfer is non-repudiable as the sender signs the transfer. The security of the transfer relies on the security of the account, which allows to avoid one password per transfer as in link transfer.

### Account Creation

To create an account, the client performs the following steps:

1. The client performs a OPAQUE registration with the server, and sends the following data to the server at the end of the OPAQUE registration:

- Username
- Email
- `cpriv_enc`, `nonce_priv_enc` and `pub_enc`, which are a key pair used for encrypt operations in account transfer. `cpriv_enc` is the encrypted private key using `export_key` with `nonce_priv_enc` as the nonce, and `pub_enc` is the public key. The key pair is randomly generated by the client, and the private key is encrypted to ensure that only the client can access it.
- `cpriv_sign`, `nonce_priv_sign` and `pub_sign`, which are a key pair used for signing operations in account transfer. `cpriv_sign` is the encrypted private key using `export_key` with `nonce_priv_sign` as the nonce, and `pub_sign` is the public key. The key pair is randomly generated by the client, and the private key is encrypted to ensure that only the client can access it.

### Account Login

The account login process is the same as the OPAQUE login process, which gives the `export_key` to the client. At the end of the OPAQUE login, the server also sends the following data to the client:

- Role of the user (e.g., user, premium, etc.)
- An array of keys, where each key contains the following data:
  - ID of the key
  - `enc_cipher_private_key`, `enc_nonce_private_key` and `enc_public_key`, which are a key pair used for encrypt operations in account transfer.
  - `sign_cipher_private_key`, `sign_nonce_private_key` and `sign_public_key`, which are a key pair used for signing operations in account transfer.
  - If the key is active or not
  - The time of the key creation
  - The time of the key revocation (if the key is revoked)

The client can then decrypt the private keys using the `export_key` and the nonces, and use the key pairs for account transfer operations.

### Key Management

The keys used in account transfer are obtained as follows:

1. The `export_key` is obtained from a successful OPAQUE registration or login, which is a 512-bit key.
2. `key_enc` is obtained using the first 256 bits of the `export_key` and is used to encrypt the private key.
3. The encryption and signing key pairs are generated by the client, with the following sizes:

- Private key: 256 bits
- Public key: 512 bits

### Nonce Management

The following nonces are used in account transfer:

- `nonce_priv_enc`: A 192-bit nonce generated randomly for encrypting the encryption private key.
- `nonce_priv_sign`: A 192-bit nonce generated randomly for encrypting the signing private key.
- `nonce_filename`: A 192-bit nonce generated randomly for encrypting the filename.
- `nonce_chunk`: A 192-bit nonce generated randomly for each chunk of the file.

### Account Transfer Creation

TODO

### Account Transfer Access

TODO

### Account Change Password

TODO

### Account Key Rotation

TODO

### Account Recovery

TODO

### Account Deletion

TODO
