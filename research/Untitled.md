
# 1. Formal specification (symbols & sizes)

Let:

- (D) be the TL-serialized data to encrypt, with (|D|\le 144) bytes (spec limit).
    
- Let (L_D = |D|).
    
- Choose random padding bytes so that the padded data has length 192 bytes:  
    [  
    P := D ,|, \text{pad},\qquad |P| = 192.  
    ]  
    (Here (|) denotes byte concatenation. The pad is uniformly random bytes of length (192-L_D).) ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    
- Define byte-reverse operator (\operatorname{rev}(\cdot)) that reverses byte order.  
    [  
    P_{\mathrm{rev}} := \operatorname{rev}(P).  
    ]
    
- Sample a fresh uniformly random 32-byte value (temporary symmetric key):  
    [  
    t \xleftarrow{$} {0,1}^{256}.  
    ]
    
- Let (\mathrm{SHA256}(\cdot)) denote the 32-byte SHA-256 digest. Form  
    [  
    H := \mathrm{SHA256}\bigl(t ,|, P\bigr)\quad\text{(32 bytes).}  
    ]
    
- Form the “data + hash” block (224 bytes):  
    [  
    X := P_{\mathrm{rev}} ,|, H,\qquad |X| = 192+32 = 224.  
    ]
    
- Encrypt (X) with AES-256 in IGE mode under key (t) and zero IV (IGE uses a 256-bit IV for AES-256; spec sets IV = 0):  
    [  
    C := \mathrm{AES256_IGE}_{t,IV=0}(X).  
    ]  
    (Note: (|C|=224) as IGE is a block cipher mode preserving length and the input length is a multiple of 16.) ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    
- Compute  
    [  
    S := \mathrm{SHA256}(C)\quad(\text{32 bytes}),  
    ]  
    [  
    t' := t \oplus S \quad(\text{32 bytes}).  
    ]
    
- Form the 256-byte RSA plaintext block  
    [  
    M := t' ,|, C  
    ]  
    (so (|M| = 32 + 224 = 256) bytes = 2048 bits).
    
- Interpret (M) as a big-endian integer (m). If (m \ge N) (the server RSA modulus), **discard (t)** and restart from sampling (t). Otherwise compute RSA encryption with server public key ((N,e)):  
    [  
    \mathrm{encrypted_data} := c := m^{e} \bmod N  
    ]  
    and serialize (c) as a 256-byte big-endian integer (leading zero bytes if necessary). ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    

This is the _entire_ `RSA_PAD` procedure from the spec (I’ve used symbolic names (P,t,X,C,t',M) to match the steps). ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))

---

# 2. Decryption (server) — invertibility & verification

Let the server receive (c). It performs standard RSA decryption with private key ((N,d)):

1. Compute  
    [  
    m := c^{d} \bmod N,  
    ]  
    and parse (m) as 256 bytes to recover:  
    [  
    m = t' ,|, C.  
    ]
    
2. Compute  
    [  
    S := \mathrm{SHA256}(C),\qquad t = t' \oplus S.  
    ]  
    (Because (t' = t \oplus \mathrm{SHA256}(C)).)
    
3. Decrypt the AES-IGE:  
    [  
    X := \mathrm{AES256_IGE}^{-1}_{t,IV=0}(C).  
    ]  
    Split (X) as (X = P_{\mathrm{rev}} ,|, H) (192 bytes || 32 bytes).
    
4. Reconstruct (P) by reversing:  
    [  
    P = \operatorname{rev}(P_{\mathrm{rev}}).  
    ]
    
5. Verify integrity:  
    [  
    \text{Check that } H \stackrel{?}{=} \mathrm{SHA256}(t ,|, P).  
    ]  
    If the check passes, accept and parse (P = D ,|, \text{pad}) and recover the original (D). If it fails, reject.
    

This sequence is deterministic and exact; each operation is reversible under the secrecy of RSA private key and the computed (t). The SHA256 check guarantees that decryption is _authentic_ (i.e., the AES key (t) corresponds to the ciphertext) and that the recovered (P) is bound to that key.

---

# 3. Correctness proof (concise)

- RSA decryption yields the original (m) because RSA is an invertible permutation modulo (N) when (m<N). The spec enforces (m<N) by retrying on failure. ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    
- From (m) we extract (t') and (C). Since (S=\mathrm{SHA256}(C)) is deterministic, (t = t' \oplus S) recovers the exact original (t).
    
- Given the correct (t), AES-IGE decryption recovers (X). Splitting (X) into (P_{\mathrm{rev}}) and (H) gives the claimed values; reversing obtains (P).
    
- The check (H \stackrel{?}{=} \mathrm{SHA256}(t|P)) prevents tampering: any modification of (C) would (except with negligible probability) change (S), thus change computed (t), and finally make the hash mismatch. Therefore, only a correctly formed (m) will pass verification.
    

Hence decryption is correct and authenticated.

---

# 4. Why each step? (Intuition & security justification)

I’ll go line-by-line through the novel/ nontrivial operations and explain their purpose.

### (a) Pad to fixed 192 bytes: (P = D ,|, \text{pad}), (|P|=192)

- **Why 192?** The RSA input block must ultimately be exactly 256 bytes (2048 bits). The scheme reserves 32 bytes for the symmetric key material (t), and 224 bytes for AES ciphertext. AES ciphertext must be a multiple of 16; 224 is (14\times16). The plaintext region (D) is therefore limited to (\le 144) so that after padding it occupies 192 bytes. Fixing the length prevents length-leakage attacks on the TL structure and simplifies formatting. ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    

### (b) Byte reverse (P_{\mathrm{rev}} = \operatorname{rev}(P))

- **Why reverse?** This is a small but deliberate transformation to ensure that the most significant bytes of the future RSA big-endian integer (m) are strongly influenced by low-order bytes of (P); it helps avoid structural patterns in the high-order bytes of (m) that could interact badly with RSA (e.g., small exponent attacks or predictable high-order bits). Reversing makes patterns less predictable in the big-endian integer representation used for RSA.
    

### (c) Generate random (t) (32 bytes) and append (H=\mathrm{SHA256}(t|P))

- **Why include (H=\mathrm{SHA256}(t|P))?** It ties the random key (t) and data (P) together cryptographically. This serves as an _integrity_ tag for (P) that depends on (t). Because (H) is appended before symmetric encryption, anyone who could manipulate ciphertext would still need to forge a 32-byte SHA256 output to pass verification — computationally infeasible.
    

### (d) AES256-IGE encryption of (X) under key (t), IV=0

- **Why envelope with symmetric encryption?** The scheme uses hybrid encryption: symmetric encryption for bulk data (fast) combined with RSA to encrypt the symmetric key (classic KEM/DEM pattern). IGE mode is used in MTProto historically (it preserves length and has particular diffusing properties across blocks; note that IGE is not a standard authenticated mode). The subsequent SHA256 binding and XOR step introduce authenticity. Zero IV is allowed because the mode and later steps provide the necessary variability (the overall construction includes randomness via (t) and random padding); still, IV=0 must be used carefully — here it’s acceptable because (t) is unique per message.
    

### (e) (t' = t \oplus \mathrm{SHA256}(C)) and form (M=t'|C)

- **Why XOR the key with (\mathrm{SHA256}(C)) and place it before (C)?** This is a blinding/anti-malleability trick: the RSA-encrypted block contains both (C) and a masked version of the key. When the server decrypts, it recomputes (\mathrm{SHA256}(C)) and XORs to recover (t). The effect is twofold:
    
    1. If an attacker tampers with (C) (flips bits), then (\mathrm{SHA256}(C)) changes unpredictably, and the recovered (t) will be wrong — causing the AES integrity check to fail.
        
    2. It prevents certain forms of direct structure exploitation of the RSA plaintext: the high-order 32 bytes are not simply the raw key but the key masked by a function of the ciphertext. This couples the two halves of (M) tightly so any independent manipulation of (t') or (C) breaks consistency.
        

This is analogous to creating a small KEM: you hide the symmetric key in an RSA envelope but do so with an additional binding that ties the key to the ciphertext itself.

### (f) Retry if (m \ge N)

- **Why check (m<N)?** RSA encryption requires the integer representative (m) to be strictly less than modulus (N). If (m\ge N), exponentiation mod (N) will still produce some (c) but the standard correspondence between plaintext integers in ([0,N-1]) and ciphertexts breaks (and using (m\ge N) could leak information or make decryption ambiguous). The spec therefore requires re-sampling (t) until (m<N). This is standard when mapping arbitrary fixed-length byte strings to RSA messages.
    

---

# 5. Security properties (what it achieves, and limitations)

**What it provides:**

1. **Confidentiality**: The data (D) is ultimately protected because an attacker seeing (c) requires RSA private key to recover (M), and without server private key the envelope is infeasible to open.
    
2. **Randomized encryption**: The randomness comes from both the random padding bytes and the random temporary key (t). This prevents deterministic RSA encryption of the same (D).
    
3. **Integrity binding**: The SHA256 check (H = \mathrm{SHA256}(t|P)) ensures that after decryption the server can detect tampering. The XOR (t' = t\oplus \mathrm{SHA256}(C)) binds (t) to the ciphertext (C) and adds protection against certain malleability attacks.
    
4. **Hybrid KEM/DEM flavor**: Conceptually this is a KEM (key encapsulation) where (t) is the encapsulated key, and (C) is the authenticated DEM ciphertext; both are packaged into RSA input.
    

**Limitations / design choices to be aware of:**

- **No explicit MAC on AES payload**: AES-IGE is not an authenticated mode. Authentication is achieved indirectly by the SHA256 binding. This is acceptable but nonstandard compared to using e.g. AES-GCM or an HMAC in the symmetric envelope.
    
- **Reliance on specific transforms (reverse, XOR)**: These are nonstandard but justified here as pragmatic defenses (avoid structure in big-endian integer, bind halves). Because the spec is fixed, implementations must follow them exactly.
    
- **Padding and length constraints**: The scheme fixes sizes (192, 224, 256) and enforces them; the client must ensure (L_D\le144). That’s a strict but simple rule.
    

---

# 6. Compact mathematical flow (summary with equations)

Starting from (D) with (|D|\le 144):

1. (P = D ,|, \text{pad},; |P|=192.)
    
2. (P_{\mathrm{rev}} = \operatorname{rev}(P).)
    
3. (t \xleftarrow{$} {0,1}^{256}.)
    
4. (H = \mathrm{SHA256}(t ,|, P).)
    
5. (X = P_{\mathrm{rev}} ,|, H) (224 bytes).
    
6. (C = \mathrm{AES256_IGE}_{t,0}(X).)
    
7. (t' = t \oplus \mathrm{SHA256}(C).)
    
8. (M = t' ,|, C) (256 bytes), (m=\mathrm{BEint}(M)).
    
9. If (m \ge N) then go to step 3.
    
10. (c = m^e \bmod N.) Output (c).
    

Server decryption inverts these steps and verifies  
[  
\mathrm{SHA256}\bigl(t ,|, P\bigr) \stackrel{?}{=} H,  
]  
which guarantees integrity.

(Algorithm source: MTProto spec, “Creating an Authorization Key”, section describing `RSA_PAD`.) ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))

---

# 7. Practical notes for implementation

- **Big-endian vs little-endian:** RSA integer interpretation is big-endian per spec; careful byte ordering when converting (M\leftrightarrow m). ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))
    
- **Repeat loops**: The loop `sample t; form m; if m>=N then repeat` has negligible expected retries (probability ( \approx \frac{\text{size of range }[N,2^{2048}-1]}{2^{2048}}), usually tiny).
    
- **Constant-time**: When implementing, be cautious to avoid timing leaks (especially when comparing hashes or processing padding). Compare hashes in constant-time where feasible.
    
- **Entropy**: (t) must be high-quality randomness (32 bytes). The padding bytes too should be random.
    

---

# 8. Final intuition (one paragraph)

`RSA_PAD` is a carefully engineered _hybrid envelope_: pick a fresh random symmetric key (t), symmetrically encrypt the data that has been fixed to a canonical length and augmented with a hash binding to (t), then mask the key with a hash of the ciphertext and package both into a 2048-bit RSA plaintext. The result is a randomized RSA encryption whose two halves (masked key and ciphertext) are cryptographically bound so that any tampering of one breaks recovery of the other. This yields practical confidentiality and integrity in a form that fits neatly into 2048-bit RSA constraints and the MTProto handshake’s format requirements. The exact sequence (reverse bytes, fixed lengths, SHA256/XOR, AES-IGE) are pragmatic choices to avoid structural weaknesses and to simplify parsing while preserving security. ([Telegram](https://core.telegram.org/mtproto/auth_key "Creating an Authorization Key"))

---

If you want, I can:

- produce a formalized pseudo-code / reference implementation (Python) that precisely follows the spec (safe for cryptographic review), or
    
- produce a small proof sketch showing that an adversary who can forge a ciphertext would imply breaking SHA256 preimage/collision or RSA, i.e., formal reductions at an informal level.
    

Which would you like next?