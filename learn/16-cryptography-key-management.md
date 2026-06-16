# Module 16 — Cryptography & Key Management

> **Core question:** How do you store secrets you must later use, and rotate the keys that sign your tokens — without ever leaking the private material?
> **Source:** the Mythos backend crawl (`backend/src/utils/crypto/*`, `creator-signing-keys.service.ts`, the MYT-106 migration, ADR-0004).
> **Prerequisites:** [04 — Auth](04-auth.md) (RS256/JWKS), [08 — Security](08-security.md) (secrets, encryption at rest), [02 — Concurrency](02-concurrency.md) (the rotation race).

## Why this module
Module 04 said "sign launch tokens with a per-listing RSA private key and publish the public half via JWKS." This module is the *how*: you have to **store that private key** somewhere it survives a restart but can't be read if the database leaks, **derive the public JWK** to publish, and **rotate** keys without a window where valid tokens stop verifying. Get the framing (IV, auth tag), the key-encryption-key, or the rotation ordering wrong and you either leak keys or break every launch.

### What you'll be able to do
- Encrypt data at rest with authenticated encryption (AES-256-GCM) and frame the ciphertext correctly.
- Protect data keys with a key-encryption-key (envelope encryption).
- Separate public from private key material and publish a correct JWKS document.
- Make encryption pluggable behind a versioned method so you can migrate to KMS later.
- Rotate signing keys atomically with zero verification downtime.

---

## Lesson 16.1 — Authenticated encryption (AES-256-GCM) & ciphertext framing

**Goal:** encrypt a secret so that tampering is *detectable*, and store everything needed to decrypt it.

### Concept
A plain cipher (CBC) hides data but doesn't detect tampering. **AES-GCM is *authenticated* encryption**: it produces an **auth tag** that decryption verifies — flip one bit of ciphertext and `final()` throws. To decrypt later you need three things: the **IV** (a fresh random nonce per encryption), the **auth tag**, and the ciphertext. The clean trick is to **frame them into one buffer**: `[IV | authTag | ciphertext]`.

### Worked example — `backend/src/utils/crypto/signing-keys.ts`
```ts
const IV_LENGTH = 12;        // 96-bit nonce — the GCM standard
const AUTH_TAG_LENGTH = 16;  // 128-bit tag

export function encryptPrivateKey(privateKeyPem: string, kek: Buffer = parseKekFromEnv()): EncryptedPrivateKey {
  if (kek.length !== 32) throw new InternalError('KEK must be 32 bytes for AES-256-GCM', 'CONFIG_ERROR');
  const iv = crypto.randomBytes(IV_LENGTH);                    // FRESH iv every time
  const cipher = crypto.createCipheriv('aes-256-gcm', kek, iv);
  const encrypted = Buffer.concat([cipher.update(privateKeyPem, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  const ciphertext = Buffer.concat([iv, authTag, encrypted]);  // frame: [iv|tag|data]
  return { ciphertext, encryptionMethod: 'aes-gcm' };
}

export function decryptPrivateKey(buf: Buffer, method: EncryptionMethod, kek = parseKekFromEnv()): string {
  // ... method/length guards ...
  const iv = buf.subarray(0, IV_LENGTH);
  const authTag = buf.subarray(IV_LENGTH, IV_LENGTH + AUTH_TAG_LENGTH);
  const encrypted = buf.subarray(IV_LENGTH + AUTH_TAG_LENGTH);
  const decipher = crypto.createDecipheriv('aes-256-gcm', kek, iv);
  decipher.setAuthTag(authTag);                                // verifies integrity on final()
  return Buffer.concat([decipher.update(encrypted), decipher.final()]).toString('utf8');
}
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — reusing a fixed IV destroys GCM's security; storing tag/iv separately invites mismatch
const iv = Buffer.alloc(12);                 // same nonce every time → catastrophic for GCM
// ❌ BAD — CBC has no integrity; an attacker can flip bits undetected
crypto.createCipheriv('aes-256-cbc', key, iv);
```
```ts
// ✅ GOOD — fresh random IV per encryption, authenticated mode, self-describing [iv|tag|data] frame
const iv = crypto.randomBytes(12);
crypto.createCipheriv('aes-256-gcm', kek, iv);
```

### Check yourself
1. Why must the IV be random *and* stored, not secret? (GCM needs a unique nonce per key; it isn't secret, but reusing one breaks confidentiality — so generate fresh and prepend it.)
2. What happens on `decipher.final()` if the ciphertext was tampered with? (It throws — the auth tag won't validate. That's the "authenticated" in AEAD.)

### ✅ Lesson 16.1 checklist
- [ ] 📖 Read & understood — I can explain AES-GCM, the IV, the auth tag, and the `[iv|tag|data]` frame.
- [ ] 💻 Applied — I encrypted+decrypted a string with AES-256-GCM and verified tampering throws.
- [ ] 🔍 Found in Mythos — `backend/src/utils/crypto/signing-keys.ts` `encryptPrivateKey`/`decryptPrivateKey`.

---

## Lesson 16.2 — Envelope encryption with a key-encryption-key (KEK)

**Goal:** protect the key that protects the data.

### Concept
The private keys are encrypted — but with *what* key, and where does *that* live? The answer is a **key-encryption-key (KEK)**: a single master key held **outside the database** (in env / a secrets manager), used to encrypt every per-listing private key. If the database leaks, the ciphertext is useless without the KEK. This is **envelope encryption**: data → encrypted by a data key (here the private key *is* the protected data) → protected by the KEK.

### Worked example — KEK validated at the boundary
```ts
const KEK_HEX_LENGTH = 64;  // 32 bytes as hex
export function parseKekFromEnv(env = process.env): Buffer {
  const raw = env.KEY_ENCRYPTION_KEY;
  if (!raw || raw.length !== KEK_HEX_LENGTH || !/^[0-9a-fA-F]+$/.test(raw)) {
    throw new InternalError(`KEY_ENCRYPTION_KEY must be exactly ${KEK_HEX_LENGTH} hex characters (32 bytes)`, 'CONFIG_ERROR');
  }
  return Buffer.from(raw, 'hex');
}
```
The KEK is **fail-fast validated** (right length, hex) at startup-ish boundaries — a misconfigured key is a loud `CONFIG_ERROR`, not a silent decrypt failure later.

### Check yourself
1. Where must the KEK *not* live? (In the same database as the ciphertext it protects — that defeats the point.)
2. Why validate the KEK length up front instead of letting `createCipheriv` fail? (Fail-fast with a clear `CONFIG_ERROR` beats an obscure crypto error deep in a request — see [Module 06](06-error-handling.md).)

### ✅ Lesson 16.2 checklist
- [ ] 📖 Read & understood — I can explain envelope encryption and why the KEK lives outside the DB.
- [ ] 💻 Applied — I loaded a master key from env and used it to wrap a data secret.
- [ ] 🔍 Found in Mythos — `parseKekFromEnv` in `signing-keys.ts`; `KEY_ENCRYPTION_KEY` in ADR-0004.

---

## Lesson 16.3 — Public/private separation & the JWKS document

**Goal:** publish only the public half, in the standard format verifiers expect.

### Concept
A keypair has a private half (signs; stays encrypted at rest) and a public half (verifies; safe to publish). For RSA the public key is just **`n` (modulus) and `e` (exponent)** — you derive a **JWK** (JSON Web Key, RFC 7517) from the PEM and expose *only* `n`/`e`, never the private key. A **JWKS** is `{ keys: [...] }`; each key carries a `kid` (key id) so a verifier can pick the right one.

### Worked example — derive JWK, build JWKS
```ts
// signing-keys.ts — public material only
export function pemToPublicKeyJwk(publicKeyPem: string): PublicKeyJwk {
  const keyObject = crypto.createPublicKey(publicKeyPem);
  const jwk = keyObject.export({ format: 'jwk' }) as { n: string; e: string };
  return { kty: 'RSA', n: jwk.n, e: jwk.e };   // n + e only — no private fields
}
// jwks.ts — wrap with kid/use/alg into the JWKS shape
export function jwkRecordToJwksKey(publicKeyJwk: PublicKeyJwk, kid: string): JwksKey {
  return { kty: 'RSA', kid: String(kid), use: 'sig', alg: 'RS256', n: publicKeyJwk.n, e: publicKeyJwk.e };
}
export const buildJwksDocument = (keys) => ({ keys });
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — exporting the whole key object can serialize private fields (d, p, q, ...)
res.json(keyObject.export({ format: 'jwk' }));   // leaks the PRIVATE key as JSON!
// ✅ GOOD — derive a public-only JWK and publish that
res.json(buildJwksDocument(keys.map(k => jwkRecordToJwksKey(k.public_key_jwk, k.kid))));
```

### ✅ Lesson 16.3 checklist
- [ ] 📖 Read & understood — I can explain JWK/JWKS, `kid`, and which fields are safe to publish.
- [ ] 💻 Applied — I derived a public JWK from a PEM and built a JWKS document.
- [ ] 🔍 Found in Mythos — `pemToPublicKeyJwk` (`signing-keys.ts`), `jwks.ts`.

---

## Lesson 16.4 — Pluggable encryption with a versioned method (planning for KMS)

**Goal:** make today's encryption swappable for tomorrow's without a data migration scramble.

### Concept
The MVP encrypts with an env-held KEK (`aes-gcm`). Production should use a managed KMS. Rather than hardcode one, the ciphertext carries an **`encryption_method` discriminator** (`'aes-gcm' | 'kms'`) stored alongside it. Decryption branches on the stored method — so old rows keep working when new rows switch to `kms`. The `kms` branch is a deliberate, *loud* `NOT_IMPLEMENTED` stub pointing at the follow-up ticket. This is forward-compatible by design (and an Open/Closed move — [Module 13](13-design-principles.md)).

### Worked example
```ts
export type EncryptionMethod = 'aes-gcm' | 'kms';
export function decryptPrivateKey(buf: Buffer, method: EncryptionMethod, kek = parseKekFromEnv()): string {
  if (method === 'kms')      throw new InternalError('KMS decryption not implemented (MYT-403)', 'NOT_IMPLEMENTED');
  if (method !== 'aes-gcm')  throw new InternalError(`Unknown encryption_method: ${method}`, 'KEY_ENCODING_ERROR');
  // ... aes-gcm path ...
}
```
The DB even enforces the enum: `CHECK (encryption_method IN ('aes-gcm', 'kms'))`.

### Check yourself
1. Why store the method *with each ciphertext* instead of a global config flag? (So you can migrate row-by-row — old `aes-gcm` rows and new `kms` rows coexist; a global flag would orphan the old rows.)
2. Why is a `NOT_IMPLEMENTED` throw better than silently falling back to `aes-gcm`? (Fail loud — a silent fallback would mis-decrypt or weaken security invisibly. See [Module 06](06-error-handling.md).)

### ✅ Lesson 16.4 checklist
- [ ] 📖 Read & understood — I can explain a versioned ciphertext discriminator and why it enables migration.
- [ ] 💻 Applied — I added a `method` field to an encrypted record and branched decryption on it.
- [ ] 🔍 Found in Mythos — `EncryptionMethod` in `signing-keys.ts`; the `encryption_method` CHECK in the MYT-106 migration; ADR-0004.

---

## Lesson 16.5 — Atomic key rotation with zero verification downtime

**Goal:** swap signing keys so that tokens signed by the *old* key still verify during the cutover.

### Concept
Rotation has a race and an availability trap:
- **The race:** "demote the old active key, insert the new active key" must be atomic, or two rotations leave you with zero or two active keys. Do it in **one `SECURITY DEFINER` RPC**, and enforce **at most one active key per listing** with a **partial unique index**.
- **The availability trap:** tokens already minted with the old key are still in flight. If you *delete* the old key, they fail verification. Instead, demote old → `rotating` (still published in JWKS) and publish the new → `active`. The JWKS query returns **both**, so a verifier picks the matching `kid` either way. You revoke the old key only after its tokens have expired.

### Worked example — the rotation RPC + the overlap query
```sql
-- MYT-106 migration: at most ONE active key per listing (partial unique index)
CREATE UNIQUE INDEX creator_signing_keys_one_active_per_listing
  ON creator_signing_keys (listing_id) WHERE status = 'active';

-- atomic rotate: demote current active → rotating, insert new active, in one transaction
CREATE FUNCTION rotate_signing_key(p_listing_id uuid, p_key_id uuid, p_kid text, ...)
RETURNS TABLE(...) LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp AS $$
BEGIN
  UPDATE creator_signing_keys SET status = 'rotating'
   WHERE listing_id = p_listing_id AND status = 'active';
  INSERT INTO creator_signing_keys(..., status) VALUES (..., 'active');
  RETURN QUERY SELECT ... WHERE key_id = p_key_id;
END $$;
```
```ts
// the JWKS feed publishes active AND rotating → old tokens keep verifying during overlap
export const getActiveJwks = (listingId) => queryJwks(['active', 'rotating'], listingId);
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — two app-level statements: a crash between them leaves 0 or 2 active keys; deleting old breaks in-flight tokens
await demoteOldKey(listingId);
await insertNewActiveKey(listingId);   // not atomic; and where's the overlap window?
```
```ts
// ✅ GOOD — one atomic RPC + a partial unique index guarantee exactly one active key,
//           and JWKS publishes ['active','rotating'] so old tokens still verify
await supabase.rpc('rotate_signing_key', { p_listing_id, p_key_id, p_kid, p_public_key_jwk, ... });
```

### Check yourself
1. Why publish `rotating` keys in JWKS instead of deleting the old key immediately? (Tokens signed before rotation are still valid until they expire; dropping the key would 401 them. Overlap, then revoke.)
2. What does the partial unique index `WHERE status='active'` buy you over a plain unique index? (It allows many `revoked`/`rotating` rows per listing but at most one `active` — the invariant you actually want.)

### ✅ Lesson 16.5 checklist
- [ ] 📖 Read & understood — I can explain atomic rotation, the partial unique index, and key overlap.
- [ ] 💻 Applied — I wrote a rotation that demotes-then-inserts in one transaction and keeps the old key verifiable.
- [ ] 🔍 Found in Mythos — `rotate_signing_key` RPC + `creator_signing_keys_one_active_per_listing` (MYT-106); `getActiveJwks(['active','rotating'])`.

---

## 🎯 Module 16 mastery checklist
- [ ] I can encrypt data at rest with AES-256-GCM and frame `[iv|tag|ciphertext]` correctly.
- [ ] I can protect data keys with a KEK held outside the database (envelope encryption).
- [ ] I can derive a public JWK and publish a JWKS without leaking private material.
- [ ] I can make encryption pluggable behind a versioned `encryption_method`.
- [ ] I can rotate signing keys atomically with an overlap window so no valid token breaks.

## 🛠️ Mini-project
Build a tiny "secret vault":
1. `encrypt(plaintext)` → `[iv|tag|ciphertext]` with AES-256-GCM using a 32-byte KEK from env; `decrypt()` back. Prove a 1-bit flip of the ciphertext makes decrypt throw.
2. Generate an RSA keypair; store the private key encrypted; expose `/jwks.json` with only `kty/kid/use/alg/n/e`.
3. Add an `encryption_method` column and a `kms` stub that throws `NOT_IMPLEMENTED`.
4. Implement `rotate()` as a single transaction that demotes the old key to `rotating` and inserts a new `active`, guarded by a partial unique index. Verify a token signed by the old key still verifies against the JWKS after rotation.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `backend/src/utils/crypto/signing-keys.ts` | AES-GCM encrypt/decrypt, IV/tag framing, KEK validation, PEM→JWK, versioned method. |
| `backend/src/utils/crypto/jwks.ts` | JWKS document + key shape (`kid/use/alg/n/e`). |
| `backend/src/services/creator-signing-keys.service.ts` | Public-only JWKS query (active+rotating), rotation via RPC. |
| `backend/supabase/migrations/20260528000000_myt106_signing_keys_launch_sessions.sql` | Partial unique index + atomic `rotate_signing_key` RPC. |
| `backend/docs/adr/0004-per-creator-signing-keys-with-aes-gcm-fallback.md` | The decision: per-tenant keys, AES-GCM-now / KMS-later. |

## See also
- [04 — Auth & authorization](04-auth.md) — RS256, JWKS, third-party verification.
- [08 — Security engineering](08-security.md) — secrets, encryption at rest, least privilege.
- [17 — SDK & library design](17-sdk-library-design.md) — the *consumer* side that fetches this JWKS.
- [19 — Integration & protocol design](19-integration-protocol-design.md) — where these tokens are used.
