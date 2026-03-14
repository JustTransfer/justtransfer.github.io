# JustTransfer Whitepaper

This page describes the security design of JustTransfer, which aims to provide an end-to-end secure account and link transfer service for large files.

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
- `X25519`, `XSalsa20` and `Poly1305`: A suite of cryptographic algorithms used to provide hybrid encryption. `X25519` is used for key exchange, `XSalsa20` for symmetric encryption, and `Poly1305` for the MAC.
- `Ed25519ph`: A digital signature scheme used for signing messages and verifying signatures to ensure authenticity and integrity.
- `AEGIS-256`: An AEAD cipher used for encrypting and authenticating filenames and file metadata. TODO ??
- `XSalsa20-Poly1305`: A stream cipher combined with a MAC for authenticated encryption of keys and messages.

TODO explain what used for what

## OPAQUE Register

The OPAQUE registration process

$$
Client:
client\_registration\_start\_result = OPAQUE_{ClientRegistration}(password)
$$

$$
Server:
server\_registration\_start\_result  = OPAQUE_{ServerRegistration}(username, client\_registration\_start\_result)
$$

$$
Client:
client\_registration\_finish\_result = OPAQUE_{ClientRegistrationFinish}(password, server\_registration\_start\_result)
$$

$$
Client:
key = client\_registration\_finish\_result.export\_key
$$

The client terminates the OPAQUE registration by sending the result of `OPAQUE_{ClientRegistrationFinish}` to the server and additional data, which depends on the mode used (link transfer or account transfer).

$$
Server:
password\_file  = OPAQUE_{ServerRegistrationFinish}(client\_registration\_finish\_result)
$$

The server stores the `password_file` associated with the username for future authentication.

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

The process of OPAQUE login is as follows:

$$
Client:
client\_login\_start\_result = OPAQUE_{ClientLogin}(password)
$$

- The server retrieves the `password_file` associated with the username and uses it to compute the OPAQUE login start result, which is sent back to the client.

$$
Server:
server\_login\_start\_result  = OPAQUE_{ServerLogin}(username, client\_login\_start\_result, password\_file)
$$

$$
Client:
client\_login\_finish\_result = OPAQUE_{ClientLoginFinish}(password, server\_login\_start\_result)
$$

$$
Client:
key = client\_login\_finish\_result.export\_key\_key
$$

$$
Server:
server\_login\_finish\_result = OPAQUE_{ServerLoginFinish}(client\_login\_finish\_result)
$$

TODO

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

TODO

### Key Management

### Nonce Management

TODO

## Account Related Actions

TODO

### Key Management

TODO

### Nonce Management

TODO

### Account Creation

- Le client génère 2 clés asymétriques de 512 bits

$$
priv1 = random[0..512]
$$

$$
pub1 = priv1*G
$$

$$
priv2 = random[0..512]
$$

$$
pub2 = priv2*G
$$

- Le client chiffre priv1 et priv2 avec `SymEnc` en utilisant sa clé `key`, ce qui donne :

$$
IV1 = random[0..192]
$$

$$
cpriv1||tag1 = SymEnc_{key}(IV1, priv1)
$$

$$
IV2 = random[0..192]
$$

$$
cpriv2||tag2 = SymEnc_{key}(IV2, priv2)
$$

TODO

### Account Login

- Le serveur renvoie (cpriv1, pub1, cpriv2, pub2) au client à la fin de la connexion OPAQUE si la connexion a réussi.
- Le client et le serveur on donc un secret partagé non-déterministe `key_communication` à la fin de l'échange OPAQUE.
- Le client déchiffre cpriv1 et cpriv2 avec SymDec :

$$
priv1 = SymDec_{key}(cpriv1||tag1||IV1)
$$

$$
priv2 = SymDec_{key}(cpriv2||tag2||IV2)
$$

- Le client contrôle que tag1 et tag2 sont correct
- Le client contrôle cette égalité pour s'assurer que la clé publique n'a pas été modifiée :

$$
priv1 * G = pub1
$$

- Le client possède donc :
  - key
  - key_communication
  - priv1
  - pub1
  - priv2
  - pub2

TODO

### Account Transfer

TODO

### Account Change Password

TODO

### Account Recovery

TODO

### Account Deletion

TODO
