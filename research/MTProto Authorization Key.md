

**Document Type:** Technical Security Analysis  
**Protocol:** MTProto v2.0  
**Focus:** Authorization Key Generation & Cryptographic Architecture  
**Target Audience:** Security Researchers, Cryptographers, Protocol Analysts  
**Last Updated:** November 10, 2025

---

## 1. MTProto Auth Key: Overview and Purpose

### 1.1 Fundamental Role

The **Authorization Key** (`auth_key`) is the foundational cryptographic primitive in the MTProto protocol, serving as the long-term shared secret between a client and a Telegram server. This 2048-bit key is generated through an authenticated Diffie-Hellman (DH) key exchange and serves as the root of trust for all subsequent encrypted communications within a session.

### 1.2 Key Characteristics

- **Size:** 2048 bits (256 bytes)
- **Lifetime:** Persistent (permanent keys) or ephemeral (temporary keys with `expires_in` parameter)
- **Scope:** Per-client, per-datacenter relationship
- **Generation:** Requires unencrypted handshake messages followed by authenticated DH exchange
- **Derivation:** Computed as `g^(ab) mod dh_prime` where `a` (server) and `b` (client) are 2048-bit random exponents

### 1.3 Dual Key Architecture

MTProto employs two types of authorization keys:

1. **Permanent Authorization Keys**: Stored persistently on both client and server, forming the basis of the client's identity with the DC.
2. **Temporary Authorization Keys**: Ephemeral keys stored only in server RAM, bound to permanent keys via `auth.bindTempAuthKey`, enabling Perfect Forward Secrecy (PFS) in client-server communications.

---

## 2. Key Concepts and Primitives

### 2.1 Cryptographic Algorithms

#### 2.1.1 Asymmetric Cryptography

- **RSA with OAEP+ Variant**: Used for encrypting the initial `p_q_inner_data` payload
    - **Key Size**: 2048-bit RSA public keys
    - **Fingerprint**: 64 lower-order bits of `SHA1(rsa_public_key)`
    - **Padding**: Custom OAEP+ variant (detailed in section 4.1 of the specification)

#### 2.1.2 Diffie-Hellman Key Exchange

- **Type**: Authenticated DH over safe prime groups
- **Prime Size**: 2048 bits (safe prime)
- **Generator**: Small values (2, 3, 4, 5, 6, or 7)
- **Subgroup Order**: `(p-1)/2` (cyclic subgroup of prime order)

#### 2.1.3 Hash Functions

- **Primary**: SHA-1 (20 bytes output)
- **Usage**:
    - Key fingerprinting
    - Nonce hashing for authentication
    - `auth_key_hash` derivation (64 lower-order bits)
    - Session validation (`new_nonce_hash1/2/3`)

### 2.2 Endianness and Binary Serialization

Critical serialization rules:

- **Large Numbers (DH parameters)**: Big-endian, transmitted as byte strings
- **Small Integers** (`int`, `long`, `int128`, `int256`): Little-endian
- **Exception**: Small integers used in SHA-1 computations are NOT rearranged (maintain little-endian representation in hash input)
- **Hash Interpretation**: SHA-1 output treated as big-endian when extracting numeric values

**Example**: `long x` as lower 64 bits of `SHA1(s)` → Take final 8 bytes of 20-byte SHA-1 output, interpret as 64-bit little-endian integer.

### 2.3 Safe Prime Requirements

The DH prime `dh_prime` (referred to as `p`) must satisfy:

1. **Size**: `2^2047 < p < 2^2048` (exactly 2048 bits)
2. **Safe Prime Property**: Both `p` and `(p-1)/2` must be prime
3. **Generator Validation**: `g` must generate a cyclic subgroup of prime order `(p-1)/2`

#### Quadratic Residue Verification

For generator `g`, verify using quadratic reciprocity law:

|Generator|Condition on `p`|
|---|---|
|`g = 2`|`p mod 8 = 7`|
|`g = 3`|`p mod 3 = 2`|
|`g = 4`|No extra condition|
|`g = 5`|`p mod 5 ∈ {1, 4}`|
|`g = 6`|`p mod 24 ∈ {19, 23}`|
|`g = 7`|`p mod 7 ∈ {3, 5, 6}`|

**Implementation Note**: Due to infrequent server changes to `dh_prime`, clients are recommended to embed pre-verified `(g, p)` pairs into application code to avoid runtime verification overhead.

### 2.4 Nonce System

The protocol employs a three-nonce authentication system:

1. **`nonce` (128 bits)**: Client-generated random identifier for the handshake session
2. **`server_nonce` (128 bits)**: Server-generated random response, binds server to session
3. **`new_nonce` (256 bits)**: Client-generated secret, transmitted only in RSA-encrypted payload

**Security Property**: The `(nonce, server_nonce)` pair uniquely identifies a "temporary session" and prevents replay attacks. An attacker cannot reuse encrypted messages in a parallel session because the server generates a unique `server_nonce` for each handshake.

### 2.5 Key Derivation Functions

#### `auth_key_hash`

```
auth_key_hash = SHA1(auth_key)[12:20]  // 64 lower-order bits
```

#### Session Validation Hashes

```
new_nonce_hash1 = SHA1(new_nonce || 0x01 || auth_key_aux_hash)[12:20]
new_nonce_hash2 = SHA1(new_nonce || 0x02 || auth_key_aux_hash)[12:20]
new_nonce_hash3 = SHA1(new_nonce || 0x03 || auth_key_aux_hash)[12:20]
```

Where `auth_key_aux_hash = SHA1(auth_key)[0:8]` (64 higher-order bits).

#### `server_salt` Derivation

```
server_salt = new_nonce[0:8] XOR server_nonce[0:8]
```

---

## 3. Auth Key Generation Process

### 3.1 High-Level Protocol Flow

```
Client                                    Server
  |                                         |
  |--------- req_pq_multi (nonce) -------->|
  |                                         |
  |<--- resPQ (server_nonce, pq, fps) -----|
  |                                         |
  | [Factor pq into p, q]                   |
  | [Generate new_nonce]                    |
  | [Encrypt p_q_inner_data with RSA]      |
  |                                         |
  |----- req_DH_params (encrypted_data) -->|
  |                                         |
  |                                         | [Decrypt with private key]
  |                                         | [Generate random a]
  |                                         | [Compute g_a = g^a mod p]
  |                                         | [Encrypt server_DH_inner_data]
  |                                         |
  |<-- server_DH_params_ok (enc_answer) ---|
  |                                         |
  | [Decrypt with new_nonce key]            |
  | [Validate dh_prime, g, g_a]             |
  | [Generate random b]                     |
  | [Compute g_b = g^b mod p]               |
  | [Compute auth_key = g_a^b mod p]        |
  | [Encrypt client_DH_inner_data]          |
  |                                         |
  |--- set_client_DH_params (enc_data) --->|
  |                                         |
  |                                         | [Decrypt with new_nonce key]
  |                                         | [Compute auth_key = g_b^a mod p]
  |                                         | [Verify auth_key_hash uniqueness]
  |                                         |
  |<----- dh_gen_ok (new_nonce_hash1) -----|
  |                                         |
  | [Verify new_nonce_hash1]                |
  | [Session established]                   |
```

### 3.2 Detailed Step-by-Step Analysis

#### Step 1: Initial PQ Request

**Message Type:** `req_pq_multi`

```tl
req_pq_multi#be7e8ef1 nonce:int128 = ResPQ;
```

- Client generates 128-bit random `nonce`
- Sent unencrypted (no `auth_key` exists yet)
- Identifies client in this handshake session

#### Step 2: Server PQ Response

**Message Type:** `resPQ`

```tl
resPQ#05162463 nonce:int128 server_nonce:int128 pq:string 
              server_public_key_fingerprints:Vector<long> = ResPQ;
```

**Parameters:**

- `pq`: Product of two distinct odd primes (typically ≤ 2^63-1)
- `server_nonce`: 128-bit random server identifier
- `server_public_key_fingerprints`: List of available RSA key fingerprints

**Client Tasks:**

- Verify echoed `nonce` matches sent value
- Factor `pq` into primes `p` and `q`
- Select RSA public key by fingerprint

#### Step 3: Client RSA Payload Generation

**Message Types:**

```tl
p_q_inner_data_dc#a9f55f95 pq:string p:string q:string 
    nonce:int128 server_nonce:int128 new_nonce:int256 dc:int = P_Q_inner_data;

p_q_inner_data_temp_dc#56fddf88 pq:string p:string q:string 
    nonce:int128 server_nonce:int128 new_nonce:int256 dc:int expires_in:int = P_Q_inner_data;
```

**Client Operations:**

1. Generate 256-bit random `new_nonce` (critical secret)
2. Serialize `p_q_inner_data` structure
3. Apply RSA_PAD encryption: `encrypted_data = RSA_PAD(data, server_public_key)`

**DC Identifier Encoding:**

- Standard DC: Use DC ID directly
- Test servers: Add 10000 to DC ID
- Media (non-CDN) DC: Negate DC ID

**Key Type Selection:**

- **Permanent Key**: Use `p_q_inner_data_dc`
- **Temporary Key**: Use `p_q_inner_data_temp_dc` with `expires_in` (seconds)

#### Step 4: DH Parameters Request

**Message Type:** `req_DH_params`

```tl
req_DH_params#d712e4be nonce:int128 server_nonce:int128 p:string q:string 
              public_key_fingerprint:long encrypted_data:string = Server_DH_Params;
```

**Security Analysis:**

- Attacker could intercept and replace with own factorization
- **Mitigation**: Modified `new_nonce` would invalidate all subsequent messages (authenticated with `new_nonce_hash`)
- Attack only results in attacker generating their own key (achievable without interception)

#### Step 5: Server DH Parameters Response

**Message Type:** `server_DH_params_ok`

```tl
server_DH_params_ok#d0e8075c nonce:int128 server_nonce:int128 
                            encrypted_answer:string = Server_DH_Params;
```

**Decrypted Inner Data:**

```tl
server_DH_inner_data#b5890dba nonce:int128 server_nonce:int128 g:int 
                             dh_prime:string g_a:string server_time:int = Server_DH_inner_data;
```

**Encryption Method:**

- `encrypted_answer` is encrypted using AES-IGE with key derived from `new_nonce`
- Server cannot compute this without knowing `new_nonce` (proves server decrypted RSA payload)

**Parameters:**

- `g`: DH generator (2, 3, 4, 5, 6, or 7)
- `dh_prime`: 2048-bit safe prime `p`
- `g_a`: Server's DH public value `g^a mod p`
- `server_time`: Server timestamp (for clock synchronization)

**Client Validation Requirements:**

1. **Prime Validation:**
    
    - Verify `2^2047 < dh_prime < 2^2048`
    - Verify `dh_prime` is prime (Miller-Rabin test)
    - Verify `(dh_prime - 1) / 2` is prime
    - **Optimization**: 15 Miller-Rabin iterations initially (error < 10^-9), continue in background
2. **Generator Validation:**
    
    - Verify `g` generates subgroup of order `(p-1)/2`
    - Apply quadratic reciprocity conditions (see Section 2.3)
3. **Public Value Range Check:**
    
    - Verify `1 < g_a < dh_prime - 1`
    - **Recommended**: Verify `2^(2048-64) < g_a < dh_prime - 2^(2048-64)`

**Performance Optimization:** Cache validated `(g, dh_prime)` pairs. Server rarely changes these values; current `dh_prime` value is documented in specification.

#### Step 6: Client DH Key Computation

**Message Type:** `set_client_DH_params`

```tl
set_client_DH_params#f5045f1f nonce:int128 server_nonce:int128 
                             encrypted_data:string = Set_client_DH_params_answer;
```

**Client Operations:**

1. Generate 2048-bit random `b` with high entropy
2. Compute `g_b = g^b mod dh_prime`
3. Compute `auth_key = (g_a)^b mod dh_prime`
4. Serialize and encrypt `client_DH_inner_data`

**Decrypted Inner Data:**

```tl
client_DH_inner_data#6643b654 nonce:int128 server_nonce:int128 
                              retry_id:long g_b:string = Client_DH_Inner_Data;
```

- `retry_id`: 0 on first attempt; equals `auth_key_aux_hash` on retry after `dh_gen_retry`
- `g_b`: Client's DH public value

#### Step 7: Server Validation and Response

**Server Operations:**

1. Decrypt and validate `client_DH_inner_data`
2. Verify `1 < g_b < dh_prime - 1` (range check recommended as in Step 5)
3. Compute `auth_key = (g_b)^a mod dh_prime`
4. Compute `auth_key_hash = SHA1(auth_key)[12:20]`
5. Check for collision with existing keys

**Response Types:**

```tl
dh_gen_ok#3bcbf734 nonce:int128 server_nonce:int128 
                   new_nonce_hash1:int128 = Set_client_DH_params_answer;

dh_gen_retry#46dc1fb9 nonce:int128 server_nonce:int128 
                      new_nonce_hash2:int128 = Set_client_DH_params_answer;

dh_gen_fail#a69dae02 nonce:int128 server_nonce:int128 
                     new_nonce_hash3:int128 = Set_client_DH_params_answer;
```

**Response Semantics:**

- **`dh_gen_ok`**: Success, `auth_key` established
    
    - Client validates `new_nonce_hash1 = SHA1(new_nonce || 0x01 || auth_key_aux_hash)[12:20]`
    - Initialize `server_salt = new_nonce[0:8] XOR server_nonce[0:8]`
    - Store time offset: `time_offset = server_time - local_time`
- **`dh_gen_retry`**: `auth_key_hash` collision detected
    
    - Client must regenerate `b` and return to Step 6
    - Set `retry_id = auth_key_aux_hash` in next attempt
- **`dh_gen_fail`**: Invalid request parameters
    
    - Client must restart from Step 1

### 3.3 Idempotency and Retry Logic

**Server Behavior:**

- Caches responses for up to 10 minutes
- Re-sends identical response if exact duplicate query received (all parameters match)
- Forgets response once client sends next protocol message (implies receipt)

**Client Behavior:**

- May retry queries on timeout
- Must use exact same parameters in retry for idempotent response

**Error Handling:**

- `-404`: Invalid request or server has forgotten session → Restart handshake
- `-444`: DC type mismatch (test DC ID used with production server or vice versa)

---

## 4. Security Considerations and Implications

### 4.1 Attack Surface Analysis

#### 4.1.1 RSA Padding Oracle Attacks

**Mitigation**: Custom OAEP+ variant prevents padding oracle attacks. The specification references "4.1" for detailed padding description, indicating awareness of OAEP vulnerabilities.

#### 4.1.2 Man-in-the-Middle (MitM) Attacks

**Threat Model**: Attacker intercepts handshake and attempts to perform key substitution.

**Protections:**

1. **RSA Authentication**: `p_q_inner_data` encrypted with server's public key ensures only legitimate server can decrypt `new_nonce`
2. **Nonce Binding**: All subsequent messages authenticated with `new_nonce_hash`, preventing message replay in forged sessions
3. **Session Uniqueness**: `(nonce, server_nonce)` pair uniquely identifies session; attacker cannot create parallel session with same parameters

**Residual Risk**: Attacker could complete handshake independently, but this is equivalent to creating a new session (no advantage gained).

#### 4.1.3 Small Subgroup Attacks

**Vulnerability**: If `g_a` or `g_b` fall into small subgroups, `auth_key` entropy is reduced.

**Mitigations:**

1. **Safe Prime Requirement**: Ensures `(p-1)/2` is prime, limiting subgroups
2. **Range Validation**: Recommended check `2^(2048-64) < g_a, g_b < p - 2^(2048-64)` ensures values are in the large subgroup
3. **Generator Verification**: Quadratic residue checks ensure `g` generates subgroup of order `(p-1)/2`

**Specification Emphasis**: "IMPORTANT: Apart from the conditions on the Diffie-Hellman prime dh_prime and generator g, both sides are to check that g, g_a and g_b are greater than 1 and less than dh_prime - 1."

#### 4.1.4 Replay Attacks

**Protection Mechanism**: `(nonce, server_nonce)` pair included in both plaintext and encrypted portions of messages.

- Server generates unique `server_nonce` per session
- Attacker cannot replay messages from one session in another (different `server_nonce`)
- `new_nonce_hash1/2/3` values validate that server has correct `auth_key` and `new_nonce`

#### 4.1.5 Integer Factorization Attacks

**Threat**: Attacker factorizes `pq` before client to inject own `new_nonce`.

**Analysis**:

- `pq ≤ 2^63-1` (64-bit integer) is easily factorized by modern computers
- **Not a vulnerability**: Even if attacker factors first and modifies `new_nonce`, they cannot create valid `new_nonce_hash` without knowledge of server's `a` value
- Attacker gains no advantage beyond generating their own independent `auth_key`

### 4.2 Perfect Forward Secrecy (PFS) Implementation

#### 4.2.1 Temporary Key Architecture

**Mechanism**: `p_q_inner_data_temp_dc` with `expires_in` parameter creates ephemeral keys.

**PFS Properties:**

1. Temporary keys stored only in server RAM
2. Server may discard before `expires_in` expiration
3. Temporary keys bound to permanent `auth_key` via `auth.bindTempAuthKey` method
4. Client generates new temporary key upon expiration

**Security Benefit**: Compromise of permanent `auth_key` does not expose past session traffic encrypted with expired temporary keys.

#### 4.2.2 PFS Limitations

**Consideration**: Permanent `auth_key` itself does NOT provide forward secrecy. PFS only applies when using temporary key infrastructure.

**Best Practice**: Rotate temporary keys frequently (configurable `expires_in` value).

### 4.3 Cryptographic Parameter Choices

#### 4.3.1 2048-bit DH Modulus

**Strength Assessment**:

- Estimated security level: ~112 bits (as of 2025)
- Adequate for current threat models
- May require upgrade to 3072-bit or elliptic curves in future (post-quantum consideration)

#### 4.3.2 SHA-1 Usage

**Controversy**: SHA-1 has known collision vulnerabilities (SHAttered attack, 2017).

**Risk Analysis in MTProto Context:**

- **Fingerprinting**: Collision attacks on RSA key fingerprints would require finding a malicious key with same fingerprint (second-preimage, not collision)
- **Nonce Hashing**: `new_nonce_hash` values used for authentication; collision attack not directly applicable
- **auth_key_hash**: Hash collision would require finding different `auth_key` with same 64-bit hash (birthday attack requires ~2^32 operations, but attacker cannot control `auth_key` value)

**Verdict**: SHA-1 usage is defensible in MTProto's specific constructions, but SHA-256 migration would improve long-term security posture.

#### 4.3.3 Small DH Generators

**Design Choice**: `g ∈ {2, 3, 4, 5, 6, 7}` simplifies quadratic residue verification.

**Security Impact**:

- No weakness introduced (verified generators ensure correct subgroup order)
- Optimization allows fast runtime checks vs. arbitrary generator validation

### 4.4 Implementation Vulnerabilities

#### 4.4.1 Entropy Requirements

**Critical**: Both `b` (client exponent) and `new_nonce` must be generated with cryptographically secure random number generator (CSPRNG).

**Insufficient Entropy**: Predictable `b` or `new_nonce` allows attacker to compute `auth_key`.

#### 4.4.2 Timing Attacks

**Vulnerability**: Prime validation and modular exponentiation may leak timing information.

**Mitigation**: Use constant-time cryptographic libraries for:

- Miller-Rabin primality testing
- Modular exponentiation (`g^b mod p`)

#### 4.4.3 Side-Channel Attacks

**Concern**: Embedded devices with limited resources may leak information through power analysis or EM emissions during DH computation.

**Recommendation**: Use hardened cryptographic implementations on sensitive platforms.

---

## 5. Relation to Message Keys and Session IDs

### 5.1 Hierarchical Key Structure

```
auth_key (2048-bit)
    ↓
├─→ auth_key_hash (64-bit) ────→ Session identification
├─→ auth_key_aux_hash (64-bit) ─→ new_nonce_hash computation
└─→ server_salt (64-bit) ───────→ Message authentication

    ↓ (Used in KDF for each message)

msg_key (128-bit)
    ↓
├─→ aes_key (256-bit) ──────────→ AES-IGE encryption key
└─→ aes_iv (256-bit) ───────────→ AES-IGE initialization vector
```

### 5.2 Message Key Derivation

**Purpose**: Each MTProto message encrypted with unique keys derived from `auth_key` and message content.

**KDF (Simplified):**

```
msg_key = SHA256(auth_key_segment || plaintext)[8:24]  // 128 bits
aes_key, aes_iv = KDF(auth_key, msg_key)
```

**Security Properties:**

- Different `msg_key` for each message (content-dependent)
- `auth_key` serves as long-term secret input to per-message KDF
- Compromise of single message key does not expose `auth_key` or other message keys

### 5.3 Session Identification

#### `auth_key_hash`

- **Derivation**: `SHA1(auth_key)[12:20]` (64 lower-order bits)
- **Purpose**: Uniquely identifies client-server session
- **Usage**: Included in encrypted message headers for session routing

#### `server_salt`

- **Derivation**: `new_nonce[0:8] XOR server_nonce[0:8]`
- **Purpose**: Prevents replay of messages across sessions with same `auth_key`
- **Rotation**: Periodically updated by server to maintain freshness

### 5.4 Temporary Key Binding

**Process (`auth.bindTempAuthKey`):**

1. Client generates temporary `auth_key_temp` using `p_q_inner_data_temp_dc`
2. Client sends binding request encrypted with permanent `auth_key`:
    
    ```
    {  perm_auth_key_id: hash(auth_key_perm),  temp_auth_key_id: hash(auth_key_temp),  expires_at: timestamp}
    ```
    
3. Server verifies both keys belong to same client
4. All subsequent messages use `auth_key_temp` for encryption

**Security Rationale:**

- Binding proves client controls permanent key (authentication)
- Temporary key used for actual traffic encryption (forward secrecy)
- Compromise of temporary key after expiration does not expose past traffic

### 5.5 Multi-DC Architecture

**Key Scope**: Each `auth_key` is specific to a single datacenter (DC).

**DC Identification**:

- Encoded in `p_q_inner_data_dc.dc` field during key generation
- Media DC vs. regular DC distinguished by sign (negative for media)
- Test vs. production indicated by DC ID offset (+10000 for test)

**Security Implication**: Compromise of one DC's `auth_key` does not affect other DCs (isolation).

---

## 6. Appendix: Reference Implementation Notes

### 6.1 Current DH Prime (dh_prime)

The specification provides the current server `dh_prime` value in hexadecimal (big-endian byte order):

```
C7 1C AE B9 C6 B1 C9 04 8E 6C 52 2F 70 F1 3F 73
98 0D 40 23 8E 3E 21 C1 49 34 D0 37 56 3D 93 0F
48 19 8A 0A A7 C1 40 58 22 94 93 D2 25 30 F4 DB
FA 33 6F 6E 0A C9 25 13 95 43 AE D4 4C CE 7C 37
20 FD 51 F6 94 58 70 5A C6 8C D4 FE 6B 6B 13 AB
DC 97 46 51 29 69 32 84 54 F1 8F AF 8C 59 5F 64
24 77 FE 96 BB 2A 94 1D 5B CD 1D 4A C8 CC 49 88
07 08 FA 9B 37 8E 3C 4F 3A 90 60 BE E6 7C F9 A4
A4 A6 95 81 10 51 90 7E 16 27 53 B5 6B 0F 6B 41
0D BA 74 D8 A8 4B 2A 14 B3 14 4E 0E F1 28 47 54
FD 17 ED 95 0D 59 65 B4 B9 DD 46 58 2D B1 17 8D
16 9C 6B C4 65 B0 D6 FF 9C A3 92 8F EF 5B 9A E4
E4 18 FC 15 E8 3E BE A0 F8 7F A9 FF 5E ED 70 05
0D ED 28 49 F4 7B F9 59 D9 56 85 0C E9 29 85 1F
0D 81 15 F6 35 B1 05 EE 2E 4E 15 D0 4B 24 54 BF
6F 4F AD F0 34 B1 04 03 11 9C D8 E3 B9 2F CC 5B
```

**Recommendation**: Embed this pre-verified value in production clients to avoid runtime validation overhead.

### 6.2 Performance Considerations

**Primality Testing**:

- Full validation: Expensive on mobile devices
- Initial validation: 15 Miller-Rabin iterations (~1 nanosecond error probability)
- Background validation: Continue iterations to reduce error further

**Caching**:

- Cache validated `(g, dh_prime)` pairs across sessions
- Cache RSA public keys by fingerprint

**Constant-Time Operations**:

- Use constant-time implementations for:
    - Modular exponentiation
    - Prime validation
    - RSA decryption

### 6.3 Error Recovery

**Network Failures**:

- Retry with identical parameters (server caches responses for 10 minutes)
- Server re-sends same response if query matches exactly

**Validation Failures**:

- `dh_gen_fail`: Restart from Step 1
- `dh_gen_retry`: Regenerate `b`, return to Step 6 with `retry_id` set

**Timeouts**:

- Server forgets temporary data after inactivity
- Client must detect timeout and restart handshake

---

## 7. Summary and Conclusions

### 7.1 Strengths

1. **Authenticated DH**: RSA encryption of `new_nonce` ensures only legitimate server can complete handshake
2. **Session Uniqueness**: `(nonce, server_nonce)` pairing prevents replay attacks across sessions
3. **PFS Support**: Temporary key infrastructure enables forward secrecy when used correctly
4. **Range Validation**: Recommended checks against small subgroup attacks
5. **Idempotency**: Server response caching provides resilience to network failures

### 7.2 Weaknesses and Concerns

1. **SHA-1 Dependency**: While not critically vulnerable in current usage, SHA-256 migration would improve security margin
2. **2048-bit DH**: Adequate now, but may need upgrade path to 3072-bit or post-quantum alternatives
3. **Complexity**: Multi-round protocol with multiple encryption layers increases implementation error surface
4. **Client-Side Factorization**: Requiring clients to factor `pq` adds computational burden (though mitigated by small `pq` size)

### 7.3 Best Practices for Implementers

1. **Use CSPRNG**: Ensure high-quality entropy for `nonce`, `new_nonce`, and `b` generation
2. **Validate Rigorously**: Implement all recommended range and primality checks
3. **Embed Known Primes**: Cache pre-verified `dh_prime` values to avoid runtime validation
4. **Constant-Time Crypto**: Use hardened libraries to prevent timing side-channels
5. **Rotate Temporary Keys**: Frequently regenerate ephemeral keys to maximize PFS benefits
6. **Handle Errors Gracefully**: Implement robust retry logic with proper timeout handling

### 7.4 Research Directions

1. **Post-Quantum Resistance**: Explore migration to lattice-based or code-based key exchange
2. **Hash Function Upgrade**: Evaluate SHA-256/SHA-3 migration without breaking backward compatibility
3. **Formal Verification**: Apply formal methods to prove protocol security properties
4. **Side-Channel Hardening**: Develop mitigation strategies for resource-constrained devices

---

## References

- [MTProto Authorization Key Specification](https://core.telegram.org/mtproto/auth_key)
- [MTProto Perfect Forward Secrecy](https://core.telegram.org/api/pfs)
- [Binary Data Serialization](https://core.telegram.org/mtproto/serialize)
- [TL Language Specification](https://core.telegram.org/mtproto/TL)
- [Example Authorization Key Generation](https://core.telegram.org/mtproto/samples-auth_key)

---

**Document Status:** Complete Deep-Dive Analysis  
**Confidence Level:** High (based on official specification)  
**Recommended Review Cycle:** Annually or upon protocol updates