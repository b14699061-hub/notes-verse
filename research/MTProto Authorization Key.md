# 🧩 MTProto Authorization Key (`auth_key`) Generation Protocol

The **MTProto Authorization Key ($\mathbf{auth\_key}$)** Generation Protocol establishes a 2048-bit symmetric master key between the client and the Telegram Data Center (DC) via an authenticated Diffie-Hellman (DH) exchange.  
This process forms the **operational and mathematical foundation** for all subsequent encrypted communication.

---

## 🔑 Auth Key: Logical & Operational Flow

| Primitive | Size | Role | Mathematical Form |
| :--- | :--- | :--- | :--- |
| $\mathbf{auth\_key}$ | 2048-bit | Master Symmetric Key | $\mathbf{auth\_key} = g^{ab} \pmod{p}$ |
| $\mathbf{p}$ (`dh_prime`) | 2048-bit | DH Safe Prime Modulus | Must satisfy $2^{2047} < p < 2^{2048}$ |
| $\mathbf{g}$ | Small Integer | DH Generator | Must generate subgroup of order $(p-1)/2$ |
| $\mathbf{a}, \mathbf{b}$ | 2048-bit | Server/Client Private Exponents | Random, high-entropy secrets |

### 🔸 Cryptographic Primitives & Secrets

The protocol uses a mix of standard and custom constructions:

* **Asymmetric Encryption** → **RSA (2048-bit)** with custom `RSA_PAD` for secure secret transmission  
* **Symmetric Encryption** → **AES256-IGE** for encrypting DH parameters & later traffic  
* **Hashing** → **SHA1**, **SHA256** for key derivation, validation, and padding  

The exchange is bound and authenticated with three **nonces**:

| Nonce | Size | Generator | Transport |
| :--- | :--- | :--- | :--- |
| $\mathbf{nonce}$ | 128-bit | Client | Plaintext in Phase 1 |
| $\mathbf{server\_nonce}$ | 128-bit | Server | Plaintext in Phase 1 |
| $\mathbf{new\_nonce}$ | 256-bit | Client | Encrypted via `RSA_PAD` |

---

## 🧮 Phase 1: Parameter Exchange & Nonce Transmission

The client securely transmits its secret $\mathbf{new\_nonce}$ to the server.

1. **C → S (Initial Request)**  
   `req_pq_multi(nonce)`

2. **S → C (Server Response)**  
   `resPQ(nonce, server_nonce, pq, fps)`  
   * Server provides $\mathbf{pq} = p \cdot q$ (product of two distinct primes).

3. **C Processing (PoW & Encrypt)**  
   * Factor $\mathbf{pq} \to \mathbf{p}, \mathbf{q}$  
   * Generate $\mathbf{new\_nonce}$  
   * Serialize: `inner_data(p, q, new_nonce)`  
   * Encrypt with RSA:  
     $\mathbf{encrypted\_data} = \text{RSA\_PAD}(\text{inner\_data}, \text{server\_pubkey})$

4. **C → S (DH Params Request)**  
   `req_DH_params(encrypted_data)`

---

## ⚙️ Phase 2: Diffie–Hellman Key Computation

Secrets are now exchanged under $\mathbf{new\_nonce}$-derived encryption.

5. **S Processing (DH Public Share)**  
   * Decrypt `encrypted_data` → retrieve `new_nonce`  
   * Generate private $\mathbf{a}$, compute $\mathbf{g^a}$  
   * Derive temporary AES key/IV from $(new\_nonce, server\_nonce)$  
   * Encrypt:  
     $\mathbf{encrypted\_answer} = \text{AES256\_IGE}(\text{server\_DH\_inner\_data}(g, p, g^a), \text{tmp\_key}, \text{tmp\_iv})$

6. **S → C (DH Params Response)**  
   `server_DH_params_ok(encrypted_answer)`

7. **C Processing (DH Key Derivation)**  
   * Decrypt `encrypted_answer` using `tmp_aes_key/iv`  
   * Verify `p`, `g`, `g^a` are safe:  
     - $p$ is safe prime  
     - $1 < g^x < p-1$  
   * Generate private $\mathbf{b}$, compute $\mathbf{g^b}$  
   * Compute master key:  
     $\mathbf{auth\_key} = (g^a)^b \pmod{p}$  
   * Encrypt share:  
     $\mathbf{enc\_data} = \text{AES256\_IGE}(\text{client\_DH\_inner\_data}(g^b), \text{tmp\_key}, \text{tmp\_iv})$

8. **C → S (Client Public Share)**  
   `set_client_DH_params(enc_data)`

9. **S → C (Final Verification)**  
   * Compute $\mathbf{auth\_key} = (g^b)^a \pmod{p}$  
   * Check for collision  
   * Send `dh_gen_ok(new_nonce_hash1)`  
   * Client verifies $\mathbf{new\_nonce\_hash1}$ to confirm successful key agreement

---

## 🔒 MTProto `RSA_PAD` Protocol

## 1. Formal Specification (Client Side)

Let $D$ be the TL-serialized data to encrypt (max $|D| \le 144$ bytes).

| **Step**                 | **Operation (Symbolic)**                                      | **Size/Derivation**    | **Intuition/Rationale**                                                                          |
| ------------------------ | ------------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------ |
| **1. Padding**           | $P := D ,                                                     |                        | , \text{pad}$                                                                                    |
| **2. Reverse**           | $P_{\mathrm{rev}} := \operatorname{rev}(P)$                   | $                      | P_{\mathrm{rev}}                                                                                 |
| **3. Key**               | $t \xleftarrow{\$} \{0,1\}^{256}$                             | $                      | t                                                                                                |
| **4. Integrity Hash**    | $H := \mathrm{SHA256}\bigl(t ,                                |                        | , P\bigr)$                                                                                       |
| **5. Plaintext Block**   | $X := P_{\mathrm{rev}} ,                                      |                        | , H$                                                                                             |
| **6. Symmetric Encrypt** | $C := \mathrm{AES256\_IGE}_{t,IV=0}(X)$                       | $                      | C                                                                                                |
| **7. Ciphertext Hash**   | $S := \mathrm{SHA256}(C)$                                     | $                      | S                                                                                                |
| **8. Masked Key**        | $t' := t \oplus S$                                            | $                      | t'                                                                                               |
| **9. RSA Plaintext**     | $M := t' ,                                                    |                        | , C$                                                                                             |
| **10. Check Modulus**    | $m = \mathrm{BEint}(M)$. If $m \ge N$: **RETRY from Step 3.** | $m < N$ (RSA Modulus). | RSA requirement: plaintext integer must be strictly less than the modulus for inversion to hold. |
| **11. RSA Encrypt**      | $\mathbf{encrypted\_data} := c := m^{e} \bmod N$              | $                      | c                                                                                                |

---

## 2. Decryption and Verification (Server Side)

The server receives the ciphertext $c$ and performs RSA decryption, followed by symmetric decryption and a mandatory integrity check.

|**Step**|**Operation (Symbolic)**|**Mathematical Derivation/Check**|**Security Rationale**|
|---|---|---|---|
|**1. RSA Decrypt**|$m := c^{d} \bmod N$. Parse $M = \mathrm{BEbytes}(m)$.|$M = t' ,||
|**2. Key Recovery**|$S := \mathrm{SHA256}(C)$<br><br>  <br><br>$t := t' \oplus S$|$t = (t \oplus S) \oplus S$.|Recovers the original symmetric key $t$. The XOR binding ensures $t$ is consistent with the received $C$.|
|**3. Symmetric Decrypt**|$X := \mathrm{AES256\_IGE}^{-1}_{t,IV=0}(C)$|$X = P_{\mathrm{rev}} ,||
|**4. Payload Reconstruct**|$P := \operatorname{rev}(P_{\mathrm{rev}})$|$P = D ,||
|**5. Integrity Check**|$\text{Verify } H \stackrel{?}{=} \mathrm{SHA256}\bigl(t ,||, P\bigr)$|

---

## 3. Concise Mathematical Summary

The entire `RSA_PAD` procedure for a plaintext $D$ of $|D| \le 144$ bytes is summarized as:

1. **Input:** $D$.
    
2. **Padding:** $P = D \,||\, \text{pad}$ ($|P|=192$).
    
3. **Reverse:** $P_{\mathrm{rev}} = \operatorname{rev}(P)$.
    
4. **Key & Hash:** $t \xleftarrow{\$} \{0,1\}^{256}$, $H = \mathrm{SHA256}(t \,||\, P)$.
    
5. **Symmetric Block:** $X = P_{\mathrm{rev}} \,||\, H$.
    
6. **Ciphertext:** $C = \mathrm{AES256\_IGE}_{t,0}(X)$.
    
7. **Masked Key:** $t' = t \oplus \mathrm{SHA256}(C)$.
    
8. **RSA Input:** $M = t' \,||\, C$.
    
9. **Output (if $\mathrm{BEint}(M) < N$):** $c = \mathrm{BEint}(M)^{e} \bmod N$.
    

The security relies on **RSA** for key encapsulation ($t$), and the **SHA256** bindings (the integrity hash $H$ and the key mask $t'$) for strong anti-malleability and authentication of the symmetric envelope.
## 🧩 Derived Secrets & Session Identifiers

The **master key** is used to derive secondary secrets for message integrity and routing.

| Secret | Derivation | Purpose |
| :--- | :--- | :--- |
| **Server Salt ($\mathbf{server\_salt}$)** | $\mathbf{new\_nonce}[0..8] \oplus \mathbf{server\_nonce}[0..8]$ | Prevents replay attacks, used in outer container |
| **Key Hash ($\mathbf{auth\_key\_hash}$)** | $\text{SHA1}(\mathbf{auth\_key})[12..20]$ | Identifies session channel |
| **Message Keys** | $\text{KDF}(\mathbf{auth\_key}, \mathbf{msg\_key})$ | Derives per-message AES key and IV |

---

## 📘 Summary

The **MTProto Auth Key Generation** process:
- Authenticates both client and server via RSA
- Performs DH key exchange under AES256-IGE encryption
- Produces a **unique, session-bound 2048-bit master key**
- Derives per-session and per-message secrets securely and efficiently
