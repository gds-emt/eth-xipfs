# eth-xipfs

[IPFS](https://ipfs.io)-augmented [Ethereum](https://www.ethereum.org) function calls enabling lower gas consumption & data encryption.

Also known as **xIPFS** in short for eXtended IPFS function call.

## Abstract

Ethereum is a public blockchain-based distributed computing platform, feature smart contract functionality. It provides a decentralized virtual machine, the Ethereum Virtual Machine (EVM) that can execute peer-to-peer smart contracts while transacting in Ethereum's native cryptocurrency – Ether (ETH).

## Motivations

The very nature of peer-to-peer smart contract means that it is public and open, with full visibility on not only its execution instructions, but also function calls, arguments and parameters. There may, however, be use cases, where privacy of data may be of concern, for example, not wanting to leak your phone number when sending notification from Ethereum SMS service.

Sending of encrypted data, including but not limited to cipher and ciphertext, on Ethereum blockchain can get expensive in gas costs for on-chain storage, an alternative off-chain storage would help alleviate the usage cost. IPFS is a great candidate for the following reasons:

1. Its content-based addressing scheme guarantees data integrity without the need to separately store hash on Ethereum blockchain.
1. Like Ethereum, IPFS is decentralized. Encryption and data hosting can be done independent of service providers.
1. It is free, with the requirement of having an IPFS node.

## Specification

xIPFS includes Ethereum function calls augmented by IPFS whether or not encryption is used.

### Encryption scheme

xIPFS utilizes a similar encryption design as Pretty Good Privacy (PGP) involving both symmetric-key cryptography and public-key (RSA) cryptography.

#### Encryption procedures (client-side)

1. Prepare payload of function arguments in JSON array without the function name or argument names, for instance:

    ```json
      ["param1", 2344, "param3"]
    ```

    For the case without encryption, this JSON array should be directly hosted on IPFS with its hash submitted to xIPFS-supported function.

1. Select a supported symmetric-key cipher of choice, say aes-256-cbc. Randomly, or psuedo-randomly generate the key, and initialization vector (IV) (if required by cipher of choice).

1. Encrypt the payload from step 1 with the selected symmetric-key cipher using the key and IV derived from step 2.

1. Encrypt the generated key, and IV, **individually**, with the published RSA public key of smart contract.

1. With the results from steps 3 and 4, store the following JSON object on IPFS and notes its IPFS hash.

    ```json
    {
      "data": "symmetrically encrypted data from step 3",
      "key": "public-key encrypted key from step 4",
      "iv": "public-key encrypted iv from step 4",
      "cipher": "cipher of choice, e.g.: aes-256-cbc"
    }
    ```

#### Decryption procedures (server-side)

1. Obtain the full JSON object from IPFS with `data`, `key`, `iv` and `cipher`.

1. Derive the symmetric key and IV by decrypting them using a private key of the published public key.

1. Decrypt the actual payload – function arguments by performing symmetric-key decryption using the specified cipher, decrypted key and decrypted IV. This should give you the very payload (JSON array) from step 1 of encryption procedure.

### Smart contract

#### Key generation

Service provider is to generate RSA public-private key pair. This can be done using standard `openssl` command, as so:

```bash
openssl genrsa -out ./privkey.pem 2048;
openssl rsa -in ./privkey.pem -pubout -out ./pubkey.pem;
```

A minimum of 2048-bit is recommended.

#### Public key

Public key is published at smart contract at the storage variable `string xIPFSPublicKey` of the main contract, stripping out end lines, header and footer of PEM.

#### xIPFS functions

Functions that support xIPFS should have accompanying non-xIPFS functions.

For example, first consider the following non-xIPFS function

```
function fn(uint param1, address param2, string param3, uiint256 param4)
```

A compatible xIPFS function can be introduced by appending `x` prefix to the function name and capitalizing the next letter of the function name, and accepting only a single argument `(string ipfs_hash)`. For the same example, a compatible xIPFS function would be:

```
function xFn(string ipfs_hash)
```

`ipfs_hash` would be the hash of IPFS where the payload can be found, either encrypted or not. The final content of it would be a JSON array representation of the original non-xIPFS function in the exact same order and without the parameter name or key.
