# RSA_PAD — MTProto Deep Dive

## 1. Overview (purpose)

`RSA_PAD(data, server_public_key)` is the custom padding + encapsulation that MTProto uses to transmit the client's `p_q_inner_data` to the server securely under the server's RSA public key. Its goals are:

1. Confidentiality of `data` under RSA encryption.
2. Integrity binding of `data` to the ephemeral `temp_key` via a hash.
3. Producing a 2048-bit integer suitable for RSA public-key operation (i.e. an integer in the range ([0, N-1]), where (N) is the RSA modulus).


MTProto implements a specific sequence of padding, hashing, symmetric encryption (AES256-IGE), and final RSA modular exponentiation. The canonical specification (core.telegram.org) gives the exact byte-level algorithm used by clients.

---

## 2. Notation and parameters

Let:

- (d) be the TL-serialized plaintext `data` (a byte string). The protocol requires (|d| \le 144) bytes.
    
- (r) be a byte string of random padding chosen so that the concatenation has length exactly 192 bytes.
    
- (D := d \,|, r) denote concatenation; hence (|D| = 192).
    
- (\operatorname{REV}(x)) denote the byte-wise reversal of string (x) (so REV reverses byte order).
    
- (t) be a freshly generated 32-byte temporary key (``temp_key''), uniformly random.
    
- (H_{SHA256}(x)) denote the 32-byte SHA-256 hash of (x).
    
- (E_{IGE}(M; K, IV)) denote AES-256-IGE encryption of message (M) with key (K) and IV (IV).
    
- (N, e) denote the server RSA public modulus and exponent respectively (with (|N| = 256) bytes = 2048 bits).
    
- Byte lengths are given in bytes; where useful we convert to bits (e.g. 256 bytes = 2048 bits).
    

---

## 3. Formal algorithm (exact steps)

Given plaintext (d) and server public key ((N,e)):

1. **Pad to 192 bytes**: choose random padding (r) such that  
    [D := d,|, r,,\qquad |D| = 192.]  
    (Require (|d| \le 144); otherwise abort.)
    
2. **Byte-reverse the padded data**:  
    [D_{rev} := \operatorname{REV}(D).]
    
3. **Generate temp_key**: sample uniformly random (t\in {0,1}^{256}) (i.e., 32 bytes).
    
4. **Append hash for integrity**:  
    [H := H_{SHA256}\bigl(t,|, D\bigr)\quad(\text{32 bytes}),]  
    [X := D_{rev},|, H.]  
    Then (|X| = 192 + 32 = 224) bytes.
    
5. **AES-IGE encrypt with zero IV**:  
    [Y := E_{IGE}(X; t, IV_0),\qquad IV_0 := 0^{256},]  
    so (|Y| = 224) (IGE preserves block multiple; already a multiple of 16).
    
6. **Derive adjusted temporary key**:  
    [s := H_{SHA256}(Y)\quad(32\text{ bytes}),]  
    [t' := t \oplus s.]
    
7. **Form RSA input integer of 256 bytes**:  
    [U := t',|, Y.]  
    Note (|U| = 32 + 224 = 256) bytes.
    
8. **Reduction check**: interpret (U) as a big-endian integer (u). If (u \ge N), discard (t) and repeat from step 3 (generate a fresh (t)).
    
9. **RSA encrypt (modular exponentiation)**:  
    [c := U^e \bmod N.]  
    Output `encrypted_data` as the 256-byte big-endian representation of (c).
    

This `encrypted_data` is placed into the `req_DH_params` packet and sent to the server.

---

## 4. Why each step — intuition + formal reasoning

We justify the steps and show how they aim to provide confidentiality and integrity.

### 4.1 Padding to 192 bytes

**Intuition.** RSA operates on integers in ([0,N-1]). MTProto chooses to build a 256-byte RSA input in a structured way: 32 bytes of key material + 224 bytes of AES-encrypted payload. The 224-byte AES payload must itself be of fixed length; MTProto fixes the raw plaintext portion to 192 bytes and adds a 32-byte SHA-256 hash — together 224 bytes.

**Reasoning.** Fixing the byte-length reduces variance that an attacker could exploit (certain malleability attacks rely on variable-length structured plaintexts). It also simplifies the final assembly of a 256-byte integer.

### 4.2 Byte reversal ((D_{rev} = REV(D)))

**Intuition.** The byte reversal is a deterministic permutation that changes the layout of the data before hashing and symmetric encryption. Two practical reasons stated by implementers and observed in historical designs:

1. **Endianness uniformity**: MTProto frequently uses mixed endianness (SHA1 segments, TL fields). Reversal makes the serialized data order independent of certain little-endian interpretations used elsewhere.
    
2. **Simple countermeasure to small-exponent patterns**: when raw payloads have predictable prefixes, reversing bytes breaks simple structures that might leak under low-exponent attacks.
    

**Formal note.** Byte reversal is bijective and does not reduce entropy; it is not a cryptographic primitive by itself but modifies the input distribution seen by subsequent hashing and encryption.

### 4.3 Appending (H = SHA256(t,|,D))

**Intuition.** This step binds the random symmetric key (t) to the plaintext (D). The server, after decryption and AES-IGE decryption using the recovered `t` (we'll see why this works), can recompute the hash and verify that the decrypted payload matches the hash. Thus this provides **integrity** (an authentication tag) inside the RSA-encrypted blob.

**Formal reasoning.** If RSA decryption recovers (U) correctly (i.e. returns the original 256 bytes), the server can parse the first 32 bytes as (t') and the last 224 bytes as (Y). It computes (s=SHA256(Y)) and recovers (t = t' \oplus s). It then decrypts (X = AES_{IGE}^{-1}(Y; t, IV_0)) and splits (X = D_{rev} ,|, H). It verifies that (H \stackrel{?}{=} SHA256(t,|,D)). If the check holds the server is assured that the plaintext originates from a party holding the proper RSA private key OR more precisely that the RSA decryption result is well-formed — note the nuance below under security discussion.

### 4.4 AES256-IGE with zero IV

**Intuition.** AES-IGE is used as an internal encryption (a KEM-like step) to protect the structured plaintext `D_rev || H` under symmetric key `t`. The outer RSA operation is then used to convey `t` safely to the server while simultaneously encrypting `D`.

**Why IGE?** IGE mode is a block cipher chaining mode that provides certain malleability and error-propagation properties desirable in some designs; MTProto historically used IGE across several places. Using AES as a symmetric primitive contributes speed and domain separation (RSA handles key transport; AES handles bulk confidentiality/integrity anchor via hash).

### 4.5 The `t' = t XOR SHA256(Y)` step

**Intuition.** After computing the AES ciphertext (Y) (which depends on (t)), MTProto computes (s = SHA256(Y)) and sends (t' = t \oplus s) as the first 32 bytes of the RSA input. The effect is that the receiver, who recovers (Y) from the RSA decryption, can recompute (s) and recover (t) via (t = t' \oplus s).

**Why not simply send (t) directly?** If the client sent a value that is independent of (Y), then certain algebraic relations might make the 256-byte RSA integer biased or structured in an exploitable way. Mixing (t) with a hash of the AES ciphertext entangles the two halves: the first 32 bytes (adjusted key) are a non-linear function of the later 224 bytes, which reduces straightforward malleability. Operationally this also helps ensure the 256-byte integer is well distributed modulo (N) (while still requiring the `u < N` check and possible repetition).

### 4.6 Rejection sampling ((u < N))

**Intuition.** The 256-byte integer (U) must be less than the RSA modulus (N); otherwise modular exponentiation on (U) would be equivalent to encryption of a larger integer which would be reduced modulo (N), yielding an ambiguous preimage after RSA decryption. To ensure a unique mapping and to avoid subtle biases, implementations check and **repeat** with a fresh (t) if (U \ge N).

**Formal note.** This is rejection sampling on a uniformly distributed 2048-bit space; since (N) is a 2048-bit number very close to (2^{2048}) in size, the probability that a random 256-byte integer is (\ge N) is small; expected number of trials is close to 1.

### 4.7 Final RSA encryption step

Finally, the client computes (c = U^e \bmod N) and sends the 256-byte big-endian integer representation of (c). The server recovers (U) by computing (U = c^d \bmod N) and parses the structure to recover (t) and then (D) and finally `data`.

---

## 5. Security properties, strengths and caveats

### 5.1 Confidentiality

Under standard RSA assumptions (RSA is a trapdoor permutation and the server's private key is secret), the construction provides confidentiality for `data` because recovering (U) requires RSA decryption.

Caveat: MTProto's `RSA_PAD` is a _homegrown_ padding variant (not exactly OAEP). While it includes hashing and symmetric encryption and tries to bind components together, historically custom padding schemes can be fragile. Practical analyses highlight that careful treatment and rigorous proofs (e.g. OAEP under chosen-ciphertext security proofs) are safer. The MTProto design chooses performance and a specific internal structure; implementers should follow the spec exactly.

(See references and public analyses for potential pitfalls in homemade hybrid constructions.)

### 5.2 Integrity / Binding

The `SHA256(t || D)` included inside the AES-encrypted payload provides an integrity check — the server verifies that the decrypted payload matches the hash computed with the recovered symmetric key. Because the hash mixes (t) with (D), an attacker who tries to modify the ciphertext would have to find a collision or break AES/sha256 properties.

However, this is an _ad-hoc_ message authentication strategy inside RSA — it is not a standardized authenticated-encryption wrapper like AES-GCM or deterministic encrypt-then-MAC constructions; correctness depends on correct implementation and no side-channel leakage.

### 5.3 KEM-like behavior

The construction behaves like a hybrid scheme / KEM + DEM (Key Encapsulation Mechanism + Data Encapsulation Mechanism):

- The RSA operation transports `t'` and `Y` together (encapsulating the symmetric key).
    
- AES-IGE with `t` encrypts the actual payload.
    

It is similar in goals to RSA-OAEP + symmetric encryption, but the internal details differ.

---

## 6. Implementation notes (practical and precise)

- **Byte order:** Everything is assembled in big-endian for the RSA integer view; TL-serialization mixes little- and big-endian in some places, so follow the spec exactly.
    
- **Randomness:** `r` and `t` must be high-quality cryptographic randomness (32 bytes for `t`, padding random to achieve 192 bytes).
    
- **Repeat loop must bound attempts:** If repeated attempts rarely fail due to `U >= N`, the implementation should still loop; in practice this is negligible.
    
- **Constant-time concerns:** While RSA public encryption is usually fast, avoid leaking data-dependent timing or branch behavior in the pre-processing code when possible (e.g., through side channels).
    
- **Rely on spec, not heuristics:** The exact placement of the hash, reversal, and XOR are part of the spec — deviating will break interop.
    

---

## 7. Worked size check (byte accounting)

- `d` : ≤144 bytes
    
- random padding `r` : so that `D` is 192 bytes
    
- `D_rev` : 192 bytes
    
- SHA256(t || D) : 32 bytes
    
- `X = D_rev || SHA256(...)` : 224 bytes
    
- AES-IGE of `X` with 32-byte key `t` and zero IV produces `Y` : 224 bytes
    
- SHA256(Y) : 32 bytes
    
- `t' = t XOR SHA256(Y)` : 32 bytes
    
- `U = t' || Y` : 256 bytes → interpreted as 2048-bit integer for RSA
    

---

## 8. Short summary (rigorous)

MTProto's `RSA_PAD`:

1. Fixes a deterministic layout (32-byte adjusted key + 224-byte AES payload) to construct a 256-byte integer `U`.
    
2. Binds symmetric key `t` to payload `D` via `H = SHA256(t || D)` and `t' = t XOR SHA256(Y)` so the receiver can recover `t` after RSA decryption.
    
3. Uses rejection sampling to ensure `U < N` and then performs the standard RSA public-key operation `c = U^e mod N`.
    

This is a hybrid, KEM-like construction tailored for MTProto's specific serialization and size constraints. Follow the canonical steps exactly when implementing.

---

## 9. Further reading / audit suggestions

When implementing or auditing `RSA_PAD`:

- Compare behaviour against RSA-OAEP and known KEM+DEM hybrid constructions.
    
- Carefully check the server-side parsing/verification code (the SHA256 check is central to integrity).
    
- Review the canonical MTProto spec ("Creating an Authorization Key") for exact byte-level wording and any historic updates.
    

---

_Note: This document follows the MTProto authoritative specification for RSA_PAD byte layout and algorithm. For extra security analysis, consult cryptanalysis literature on custom padding and hybrid schemes._