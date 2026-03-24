# JustTransfer Whitepaper

This page describes the security design of JustTransfer, which aims to provide an end-to-end secure account and link transfer service for large files.

## Table of Contents

- [Threat Model](#threat-model)
- [Security Level](#security-level)
- [Cryptographic Primitives](#cryptographic-primitives)
- [Link Transfer vs Account Transfer](#link-transfer-vs-account-transfer)
- [Link Transfer](#link-transfer)
- [Account Transfer](#account-transfer)
- [Annex](#annex)

---

## Threat Model

The threat model of JustTransfer includes the following assumptions:

- Active Adversary: The adversary can actively intercept, modify, and inject messages between the client and the server. They may also attempt to impersonate either party to gain unauthorized access.
- Server honest-but-curious: The server is assumed to be honest in following the protocol but may try to learn sensitive information from the data it processes. It will not intentionally deviate from the protocol but may attempt to analyze the data for insights.
- Client-side Security: The client is responsible for securely handling sensitive information, such as passwords and cryptographic keys. The client is assumed to be secure and not compromised by malware or other attacks.

---

## Security Level

The following key sizes and parameters are used in JustTransfer:

- Symmetric Key Size: 256 bits
- Asymmetric Key Size: 512 bits for elliptic curve cryptography (ECC)
- Hash Function Output Size: 256 bits

---

## Cryptographic Primitives

JustTransfer utilizes the following cryptographic primitives:

- `OPAQUE`: An asymmetric password-authenticated key exchange protocol that allows secure authentication without revealing the password to the server.
- `Argon2`: A memory-hard password hashing function used to derive keys from passwords.

The following cryptographic primitives are used in Link Transfer:

- `AEGIS-256`: An AEAD cipher used for encrypting and authenticating filenames and file metadata in link transfer.
  - Key size: 256 bits
  - Nonce size: 256 bits
- `XChacha20-Poly1305`: A stream cipher combined with a MAC for authenticated encryption, used to encrypt file chunks in link transfer.
  - Key size: 256 bits
  - Header size (nonce): 192 bits

The following cryptographic primitives are used in Account Transfer:

- `XSalsa20-Poly1305`: A stream cipher combined with a MAC for authenticated encryption, used to encrypt the private keys of an account.
  - Key size: 256 bits
  - Nonce size: 192 bits
- `X25519`, `XSalsa20` and `Poly1305`: A suite of cryptographic algorithms used to provide hybrid encryption, used to encrypt filename and file chunks in account transfer.
  - Public key size: 256 bits
  - Private key size: 256 bits
  - Nonce size: 192 bits
- `Ed25519ph`: A digital signature scheme used for signing messages and verifying signatures to ensure authenticity and integrity.
  - Public key size: 256 bits
  - Private key size: 512 bits

---

### Link Transfer vs Account Transfer

The following table summarizes the differences between link transfer and account transfer in JustTransfer:

| Feature          | Link Transfer                            | Account Transfer                                                  |
| ---------------- | ---------------------------------------- | ----------------------------------------------------------------- |
| Authentication   | One password for each transfer           | One password for the account, which can manage multiple transfers |
| Repudiation      | Yes                                      | No (transfers are signed by the sender)                           |
| Forward Secrecy  | Yes                                      | Yes (on key rotation)                                             |
| Backward Secrecy | Yes                                      | Yes (on key rotation)                                             |
| Limits           | Limited (size, lifetime, download times) | Higher limits (depends on the account)                            |

---

## Link Transfer

The link transfer allows a user to share a file with other users by creating a link. As link transfer does not require the user to have an account, the transfers are repudiable, and the security relies on the secrecy of the password used in the OPAQUE registration.

### Link Transfer Creation

To create a link transfer, the client performs the following steps:

1. The client performs an OPAQUE registration with the server.
2. The client derives the `key_metadata` using the first 256 bits of the `export_key` obtained from the OPAQUE registration.
3. The client derives the `key_file` using the last 256 bits of the `export_key` obtained from the OPAQUE registration.
4. The client encrypts the filename using AEGIS-256 with `key_metadata` and `nonce_filename` (random), and put the `max_download`, `lifetime`, `creation_time`, `file_size`, `chunk_size` and `header` in authenticated data to ensure the integrity of the metadata.
5. The client finish the OPAQUE registration with the server, and sends the following data along with the OPAQUE registration finish result:
   - ID of the transfer, which is provided by the server with the server registration start result
   - The encrypted filename
   - The nonce of the encrypted filename, which is generated randomly
   - Header, which is randomly generated and used as the nonce for the chunk encryption
   - Max download times of the transfer
   - Lifetime of the transfer
   - Transfer creation time
   - The file size of the transfer

6. The server stores the data and returns the upload-URLs to the client, which the client can use to upload the file in chunks.
7. The client encrypts each chunk of the file using XChacha20-Poly1305 with `key_file` and `header` as the nonce (random).
8. The client uploads the encrypted chunks to the upload-URLs.
9. The client terminates the transfer by sending the `etags` of the chunks to the server, which are used to finish the transfer upload.
10. The client displays the link to the user, using the transfer ID.

> Note: As the `header` is included in the authenticated data of the filename encryption, any modification or swapping of chunks will be detected as the integrity verification of the chunks will fail.

### Link Transfer Access

To access a link transfer, the client performs the following steps:

1. The client performs an OPAQUE login with the server, using the transfer ID as the username.
2. The client derives the `key_metadata` using the first 256 bits of the `export_key` obtained from the OPAQUE login.
3. The client derives the `key_file` using the last 256 bits of the `export_key` obtained from the OPAQUE login.
4. The client requests the transfer metadata from the server, which includes the following data:
   - ID of the transfer
   - The encrypted filename
   - The nonce of the encrypted filename
   - The file ID of the transfer
   - The header of the transfer
   - The max download times of the transfer
   - The lifetime of the transfer
   - The transfer creation time
   - The current download times of the transfer
   - The file size of the transfer
   - The chunk size of the transfer

5. The client decrypts the filename using AEGIS-256 with `key_metadata` and `nonce_filename`. It then verifies the integrity of the metadata using the authenticated data.
6. The client requests the download URLs from the server, which returns pre-signed URLs for downloading the file in chunks.
7. The client downloads the encrypted chunks using the download URLs, and decrypts each chunk using XChacha20-Poly1305 with `key_file` and `header` as the nonce.
8. The client verifies the integrity of each chunk using the MAC provided by XChacha20-Poly1305.
9. The client reconstructs the file from the decrypted chunks and provides it to the user.

### Key Management

The keys used in link transfers are obtained as follows:

1. The `export_key` is obtained from a successful OPAQUE registration or login, which is a 512-bit key.
2. `key_metadata` is obtained using the first 256 bits of the `export_key`
3. `key_file` is obtained using the last 256 bits of the `export_key`

As the `export_key` is derived from the password (which never leaves the client device), the server can never access the keys, which ensures the confidentiality and integrity of the files in link transfer.

### Nonce Management

The following nonces are used in link transfer:

- `nonce_filename`: A 192-bit nonce generated randomly for encrypting the filename.
- `header`: A 192-bit nonce generated randomly for encrypting each chunk of the file.

As a new key is generated for each new link transfer, there is no risk of nonce reuse across different transfers. For the chunk encryption within the same transfer, the `header` is used as the nonce, and rekeying is automatically performed by Libsodium if the counter exceeds the limit.

### Data Storage

The following data are stored **unencrypted** in the server for each link transfer:

- ID of the transfer
- The nonce of the encrypted filename
- Header of the transfer
- Max download times of the transfer
- Lifetime of the transfer
- Transfer creation time
- The file size of the transfer
- The chunk size of the transfer

The following data are stored **encrypted** in the server for each link transfer:

- The encrypted filename
- The file chunks

---

## Account Transfer

The account transfer allows a user to send files to other users. The transfer is non-repudiable as the sender signs the transfer. The security of the transfer relies on the security of the account, which avoids requiring one password per transfer and allows users to manage their transfers and keys in one place.

### Account Creation

To create an account, the client performs the following steps:

1. The client performs an OPAQUE registration with the server.
2. The client derives the `key_enc` using the first 256 bits of the `export_key` obtained from the OPAQUE registration.
3. The client generates two key pairs for encryption and signing, which are used for encrypting the file and signing the transfer in account transfer.
4. The client encrypts the private keys of the key pairs (`enc_private_key` and `sign_private_key`) using `key_enc`, with `enc_nonce_private_key` and `sign_nonce_private_key` as the nonces (random), respectively using XSalsa20-Poly1305. This generates the `enc_cipher_private_key` and `sign_cipher_private_key`.
5. The client sends the following data to the server at the end of the OPAQUE registration:
   - Username
   - Email
   - `enc_cipher_private_key`, `enc_nonce_private_key` and `enc_public_key`, which is a key pair used for encrypting operations in account transfer.
   - `sign_cipher_private_key`, `sign_nonce_private_key` and `sign_public_key`, which is a key pair used for signing operations in account transfer.

### Account Login

The account login process is the same as the OPAQUE login process, which produces the `export_key` to the client. At the end of the OPAQUE login, the server also sends the following data to the client:

- Role of the user (e.g., user, premium, etc.)
- An array of keys, where each key contains the following data:
  - ID of the key
  - `enc_cipher_private_key`, `enc_nonce_private_key` and `enc_public_key`, which is a key pair used for encrypting operations in account transfer.
  - `sign_cipher_private_key`, `sign_nonce_private_key` and `sign_public_key`, which is a key pair used for signing operations in account transfer.
  - If the key is active or not
  - The time of the key creation
  - The time of the key revocation (if the key is revoked)
- Cookie which is used for authentication of the future requests to the server.

The client can then decrypt the private keys using the `export_key` and the nonces and use the key pairs for account transfer operations.

### Key Management

The keys used in account transfers are obtained as follows:

1. The `export_key` is obtained from a successful OPAQUE registration or login, which is a 512-bit key.
2. `key_enc` is obtained using the first 256 bits of the `export_key` and is used to encrypt the private key.
3. The encryption and signing key pairs are generated by the client, and stored in the server in encrypted form for the private keys.

As the private keys are encrypted using the `key_enc` derived from the password (which never leaves the client device), the server can never access the private keys, which ensures the confidentiality and integrity of the files in account transfer.

### Nonce Management

The following nonces are used in account transfer:

- `nonce_priv_enc`: A 192-bit nonce generated randomly for encrypting the encryption private key.
- `nonce_priv_sign`: A 192-bit nonce generated randomly for encrypting the signing private key.
- `nonce_filename`: A 192-bit nonce generated randomly for encrypting the filename.
- `nonce_chunk`: A 192-bit nonce generated randomly for each chunk of the file.

The maximum number of chunks (and transfers) that can be securely encrypted with the same key and randomly generated nonce can be calculated using the birthday paradox formula:

$$
1 - e^{-\frac{n^2}{2d}} < 2^{-32}
$$

Where `n` is the number of chunks and `d` is the number of possible nonces (which is $2^{192}$ for a 192-bit nonce). Solving for `n` gives us:

$$
n \approx 2^{80}
$$

This means that up to $2^{80}$ chunks can be securely encrypted with the same key and randomly generated nonce before the probability of a nonce collision becomes significant (greater than $2^{-32}$).

$$
chunkSize = 10 \text{ MB}
$$

$$
sizeLimit = 2^{80} \times chunkSize \approx 1.2 \times 10^{22} \text{ GB}
$$

As this limit is extremely large and practically unreachable, we can safely assume that nonce reuse will not occur in account transfer under normal usage.

### Data Storage

For each account, the server stores the following data:

- Username
- Email
- An array of keys, where each key contains the following data:
  - ID of the key
  - `enc_cipher_private_key`, `enc_nonce_private_key` and `enc_public_key`, which is a key pair used for encrypting operations in account transfer.
  - `sign_cipher_private_key`, `sign_nonce_private_key` and `sign_public_key`, which is a key pair used for signing operations in account transfer.
  - If the key is active or not
  - The time of the key creation
  - The time of the key revocation (if the key is revoked)

For each account transfer, the server stores the following data **unencrypted**:

- ID of the transfer
- Sender's username
- Sender's key ID, which is used to reference the sender's public signing key
- Recipient's username
- Recipient's key ID, which is used to reference the recipient's keys
- The nonce of the encrypted filename
- The file ID of the transfer
- The max download times of the transfer
- The lifetime of the transfer
- The transfer creation time
- The current download times of the transfer
- The file size of the transfer
- The chunk size of the transfer

The server also stores the following data **encrypted** for each account transfer:

- The encrypted filename
- The file chunks

### Account Transfer Creation

To create an account transfer, the client performs the following steps:

1. The client requests the server for the recipient's public keys using the recipient's username.
2. The client encrypts the filename using the following steps:
   - Generate a random nonce `nonce_filename` for encrypting the filename.
   - Encrypt the filename using the `X25519-XSalsa20-Poly1305` cipher suite and the following keys:
     - Sender's private encryption key
     - Recipient's public encryption key

3. The client signs the following data using the sender's private signing key to ensure the authenticity and integrity of the transfer:
   - Encrypted filename
   - Nonce of the encrypted filename
   - File ID of the transfer
   - Sender's username
   - Recipient's username
   - Max download times of the transfer
   - Lifetime of the transfer
   - Creation time of the transfer
   - File size of the transfer
   - Chunk size of the transfer
4. The client sends the following data to the server to create the transfer:
   - Sender's key ID, which is used to reference the sender's public signing key
   - Recipient's key ID, which is used to reference the recipient's keys
   - Encrypted filename and the nonce of the encrypted filename
   - Max download times of the transfer
   - Lifetime of the transfer
   - Transfer creation time
   - The file size of the transfer

5. The server stores the data and returns the upload-URLs to the client.
6. The client splits the file into chunks and performs the following steps for each chunk:
   - Generate a random nonce `nonce_chunk` for encrypting the chunk.
   - Encrypt the chunk using the `X25519-XSalsa20-Poly1305` cipher suite and the following keys:
     - Sender's private encryption key
     - Recipient's public encryption key
   - Append the `nonce_chunk` to the encrypted chunk at the beginning, which will be used for decryption later.
   - Update the signature of the transfer using the sender's private signing key and the content of the current chunk.

7. The client uploads the encrypted chunks to the upload-URLs.
8. The client terminates the transfer by sending the `etags` and the signature of the transfer to the server, which are used to finalize the transfer upload.

### Account Transfer Access

To access an account transfer, the client performs the following steps:

1. The client requests its received transfers from the server, and the server returns the metadata of the transfers, which includes the following data:

- ID of the transfer
- Sender's username
- Sender's key ID
- Recipient's username
- Recipient's key ID
- The encrypted filename
- The nonce of the encrypted filename
- The file ID of the transfer
- The max download times of the transfer
- The lifetime of the transfer
- The transfer creation time
- The signature of the transfer
- The current download times of the transfer
- The file size of the transfer
- The chunk size of the transfer

2. The client initializes the signature verification of the transfer using the metadata of the transfer.
3. The client decrypts and verify the integrity and authenticity of the filename using the `X25519-XSalsa20-Poly1305` cipher suite and the following keys:
   - Recipient's private encryption key
   - Sender's public encryption key
4. The client requests a download URL from the server, which returns a pre-signed URL for downloading the file.
5. The client downloads the file in chunks using the download URL, and each chunk is processed as follows:
   - The client extracts the `nonce_chunk` from the beginning of the encrypted chunk.
   - The client decrypts the chunk using the `X25519-XSalsa20-Poly1305` cipher suite and the following keys:
     - Recipient's private encryption key
     - Sender's public encryption key
   - The client verifies the integrity and authenticity of the chunk using the MAC provided by the `X25519-XSalsa20-Poly1305` cipher suite.
   - The client updates the signature verification of the transfer using the content of the current chunk.
6. The client verifies the final signature of the transfer after processing all chunks to ensure the authenticity and integrity of the entire transfer.
7. The client reconstructs the file from the decrypted chunks and provides it to the user.

### Account Change Password

To change the password of an account, the client performs the following steps:

1. The client initiates an OPAQUE registration with the server using the new password. The private keys are encrypted using the new `key_enc` derived from the new password (and new `export_key`), and the client sends the following data to the server at the end of the OPAQUE registration:

- An array of keys, where each key contains the following data:
  - ID of the key
  - `enc_cipher_private_key`, `enc_nonce_private_key` and `enc_public_key`, which is a key pair used for encrypting operations in account transfer.
  - `sign_cipher_private_key`, `sign_nonce_private_key` and `sign_public_key`, which is a key pair used for signing operations in account transfer.

2. The server accepts updating the password file and the keys if the OPAQUE registration is successful, and the user is authenticated (valid cookie). The server then updates the password file and the keys with the new data.

### Account Key Rotation

To rotate the keys of an account, the client performs the following steps:

1. The client generates two new key pairs for encryption and signing. The client encrypts both private keys using the `key_enc`. The new keys and nonces are randomly generated.
2. The client sends the new keys to the server, which adds the new keys to the account and marks the old keys as revoked. The server also stores the time of the key revocation.
3. The server deletes the old keys if they are revoked and not used in any other transfer.

### Account Recovery

To recover an account, the client performs the following steps:

1. The client submits an account recovery request to the server with the email address associated with the account.
2. The server sends a recovery email to the email address, which contains a recovery link with a unique token.
3. The client clicks on the recovery link, which directs them to a password reset page where they can enter a new password.
4. The client initiates an OPAQUE registration with the server using the new password.
5. The server accepts to reset the password if the OPAQUE registration is successful, and the reset token is valid. The server then updates the password file and revokes the reset token to prevent reuse.
6. The server deletes all other keys and transfers associated with the account, as it is assumed that the password is lost, so the client will not be able to decrypt them anymore.

### Account Deletion

To delete an account, the client performs the following steps:

1. The client sends an account deletion request to the server.
2. The server deletes all data associated with the account, including the user, keys, sent and received transfers, and any other related data.

---

## Annex

### OPAQUE Register

The OPAQUE registration process consists of the following steps:

1. The client initiates the OPAQUE registration by sending the username and the result of `OPAQUE_{ClientRegistration}` to the server.

$$
client\_registration\_start\_result = OPAQUE_{ClientRegistration}(password)
$$

2. The server computes the OPAQUE registration start result using the username and the client's registration start result, and sends it back to the client.

$$
server\_registration\_start\_result  = OPAQUE_{ServerRegistration}(username, client\_registration\_start\_result)
$$

3. The client computes the OPAQUE registration finish result using the password and the server's registration start result, which produces the export key.

$$
client\_registration\_finish\_result = OPAQUE_{ClientRegistrationFinish}(password, server\_registration\_start\_result)
$$

$$
export\_key = client\_registration\_finish\_result.export\_key
$$

4. The client sends the OPAQUE registration finish result to the server. The server computes the OPAQUE registration finish result, which produces the `password_file` that will be stored with username for future authentication. The client can also send other data to the server at this step, which depends on the type of registration (account registration or link registration).

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

### OPAQUE Login

The OPAQUE login process consists of the following steps:

1. The client initiates the OPAQUE login by sending the username and the result of `OPAQUE_{ClientLogin}` to the server.

$$
client\_login\_start\_result = OPAQUE_{ClientLogin}(password)
$$

2. The server computes the OPAQUE login start result using the username, the client's login start result, and the `password_file` stored for the username and sends it back to the client.

$$
server\_login\_start\_result  = OPAQUE_{ServerLogin}(username, client\_login\_start\_result, password\_file)
$$

3. The client computes the OPAQUE login finish result using the password and the server's login start result, which produces the export key.

$$
client\_login\_finish\_result = OPAQUE_{ClientLoginFinish}(password, server\_login\_start\_result)
$$

$$
export\_key = client\_login\_finish\_result.export\_key\_key
$$

4. The client sends the OPAQUE login finish result to the server. The server computes the OPAQUE login finish result and returns various data to the client, which depends on the type of login (account login or link login).

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
