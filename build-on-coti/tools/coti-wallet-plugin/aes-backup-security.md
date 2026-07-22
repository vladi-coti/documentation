# AES backup security model

Signature-derived AES key backup is a **compatibility fallback**, not the preferred way to protect COTI privacy keys.

## Supported storage path

The **only supported and maintained** encrypted-backup storage path is **browser `localStorage`**, enabled with:

```ts
configureCotiPlugin({
  onboardingServices: { mode: 'localStorage' },
});
```

Keys are stored as `coti-wallet-plugin:aes-backup:<chainId>:<address>` and contain only the encrypted blob (never the plaintext AES key).

Remote / custom backup backends are **not supported or maintained**. Do not plan security work or production integrations around remote AES backup APIs.

## Architectural priority

1. **Wallet-native protected storage** — MetaMask Snap storage (or equivalent wallet-managed sealed storage).
2. **Dedicated deterministic encryption keypair** — separated from the transaction-signing key, when the wallet exposes such a capability.
3. **Signature-derived encrypted backup in localStorage** — this mechanism; only for wallets that have been verified to reproduce identical EIP-712 signing material.

Do not treat (3) as a universal recovery product. The plugin does not introduce HPKE or a dedicated encryption derivation scheme unless wallet support already exists.

## What possession of a backup means

An encrypted AES backup blob alone is **not** enough to recover the COTI AES key.

Possession of **both**:

* the encrypted backup (`iv` + `ciphertext` + bound metadata), and
* a matching EIP-712 signature over the backup wrap message

**is** sufficient to derive the wrapping key (via HKDF-SHA256) and decrypt the AES key.

The wrap signature is therefore a **reusable decryption credential** for that backup context, not a one-time proof.

For localStorage this means: any script on the same origin that can read `localStorage` can obtain the blob; decrypting it still requires tricking the user into producing the wrap signature (or already having that signature).

## Domain separation is not origin binding

The v2 EIP-712 domain includes a protocol salt and version. That separates this protocol from unrelated typed-data messages.

It does **not**:

* bind the request to a specific dApp origin;
* stop another site from recreating the same public EIP-712 payload and asking the user to sign it;
* make recovery phishing-resistant by itself.

If a malicious (or compromised) dApp obtains the encrypted blob **and** tricks the user into signing the wrap message, it can decrypt the AES key.

### User-facing signing warning

Wallet prompts and the EIP-712 `purpose` field use wording equivalent to:

> This signature unlocks your encrypted COTI privacy key backup. Only sign from an official or explicitly trusted COTI application.

Treat every wrap signature as a sensitive unlock action.

## What recovery is *not*

Recovery is **not** guaranteed merely because the user controls the same address later.

Restore only works when:

* the wallet reproduces **identical effective signing material** for the same EIP-712 domain + message;
* the backup format is supported (v2);
* address and chain bindings match;
* AES-GCM authentication succeeds.

Same address with a different signer implementation, MPC policy, smart-account signature scheme, or nondeterministic ECDSA can make an existing backup permanently unrestorable.

localStorage backups are also **same-browser / same-origin**. Clearing site data, another browser, or another device will not restore the blob.

## Determinism check (default on)

Before any `save` / `replace` of a newly created backup, the plugin:

1. encrypts with signature #1;
2. requests signature #2 independently;
3. decrypts the in-memory blob with the key derived from signature #2;
4. **only then** persists the blob to localStorage.

Escape hatch (unsafe, default `false`):

```ts
configureCotiPlugin({
  unsafeSkipBackupDeterminismCheck: true,
});
```

Prefer never enabling this in production.

Stable failure code when the second signature cannot decrypt:

`AES_BACKUP_WALLET_NOT_SUPPORTED` (`CotiErrorCode.AES_BACKUP_WALLET_NOT_SUPPORTED`)

Other persist outcomes are distinguished as:

| Outcome | Meaning |
| --- | --- |
| `cancelled` + `USER_REJECTED` | User rejected a required signature |
| `failed` + `AES_BACKUP_WALLET_NOT_SUPPORTED` | Wallet produced non-reproducible signing material |
| `failed` + `AES_BACKUP_CRYPTO_VALIDATION_FAILED` | Encrypt/self-test / validation failure before storage |
| `failed` + `AES_BACKUP_STORAGE_FAILED` | localStorage (or other storage) write failed after a valid blob was produced |
| `failed` + `NO_PROVIDER` | Wallet provider unavailable |

## Wallet support policy

### Officially supported (expected to work)

EOA wallets that:

* implement **deterministic ECDSA** for EIP-712 (`eth_signTypedData_v4`);
* keep stable signature serialization for the same digest;
* use a fixed signing key for the account (no rotation mid-backup lifecycle).

In practice this includes common software EOAs such as MetaMask (extension) when signing with a standard imported/generated account key.

### Not officially supported / treat as unsupported

Do **not** rely on signature-derived backup for:

* randomized ECDSA implementations;
* MPC wallets;
* multisig wallets;
* ERC-1271 smart accounts;
* passkey wallets;
* rotating-key accounts;
* hardware wallets whose firmware may change signature behavior;
* wallets that change signature serialization across versions.

The default determinism check detects non-reproducible signers **before** saving. Prefer Snap (or equivalent) for those wallets when available.

Avoid marketing copy such as “recoverable from any wallet using the same address.”

## `keyEpoch` (reserved for a future format)

Backup format **v2** does **not** expose `keyEpoch` on the public `EncryptedAesBackup` type.

An unbound or partially bound epoch field must not ship in the public format. If key rotation metadata is required later, introduce a new format version that includes the epoch in:

* the EIP-712 signed message;
* HKDF info;
* AES-GCM AAD;
* restore validation;

without reinterpretating existing v2 blobs under new semantics.

## Remote storage

Remote AES backup APIs are **out of scope**. They are not supported or maintained. Security fixes for encrypted backup apply to the localStorage path and wrap cryptography only. See [Secure remote AES backup storage](aes-backup-remote-storage.md) for the historical note.

## Related docs

* [AES Key Onboarding](aes-key-onboarding.md)
* [Configuration](configuration.md)
