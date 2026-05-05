---
layout: default
title: "picoCTF 2026 — Cryptography Writeups"
categories: [CTF, Cryptography, picoCTF]
---

picoCTF 2026 ran live and I played most of the cryptography track under HackerTroupe. These are the crypto challenges I solved and how each one came together.

**Challenges:** [StegoRSA](#stegorsa) · [Timestamped Secrets](#timestamped-secrets) · [Small Trouble](#small-trouble) · [Shift Registers](#shift-registers) · [Black Cobra Pepper](#black-cobra-pepper) · [ClusterRSA](#clusterrsa) · [Not TRUe](#not-true) · [MSS Advance Revenge](#mss-advance-revenge)

---

# # StegoRSA

**Category:** Cryptography / Steganography
**Points:** 100
**Files:** `flag.enc` (binary blob), `image.jpg` (a JPEG of a key)

![StegoRSA challenge image](/images/StegoRSA-picoctf2026.jpg)

`flag.enc` is unreadable binary — the encrypted flag:

```
&lå”DƒDŠ/¼eZ›“†ß&iïŽLŸƒsÇ ?áG}>½µ5OÝ™]aæ
™ò–ÜŠ£õpù;KÑ{[ðD2rõV/m¥;µTãwrÏ„)¨àw/¹õ»s;˜l¦ÐÒÔ³ |“ö¶ŒSk¶úQ—­M“¿|Å?Am!>#"=ä0_Å–oY(gŸö&ö?9¢K°y€ðn¾ÏŸß©yŠ›–óîM÷
ô;,Èä1lsåèÊïå¸€êï¼pXøŠ/nÍléÁßO;Åw™¦UYR>ü ¡^ùñSÖrÐ+¨ R¦›H¼ÙÁ3 ¤û]è³Ý
t™Ë†
```

The challenge name and the description make the plan obvious — the RSA private key is hidden somewhere in the image, and `flag.enc` is the RSA-encrypted flag. The hint explicitly says metadata.

## Pulling the key

EXIF is the first place to look in a JPEG. Run `exiftool` to dump every field:

```bash
exiftool image.jpg
```

The `Comment` field held a long hex string — clearly a binary key encoded as hex so it would survive being stuffed into a text-only metadata slot.

Pull that field's raw value out and run it back through `xxd` to reconstruct the binary PEM:

```bash
exiftool -b -Comment image.jpg | xxd -r -p > private.pem
```

- `-b` makes `exiftool` print the raw value (no labels/whitespace).
- `xxd -r -p` reverses a plain-hex stream back into binary.

Confirm the result is a real PEM:

```bash
head -1 private.pem
# -----BEGIN RSA PRIVATE KEY-----
```

## Decrypt

Standard `openssl` decrypt:

```bash
openssl pkeyutl -decrypt -inkey private.pem -in flag.enc -out flag.txt
```

**Flag:** `picoCTF{rs4_k3y_1n_1mg_66388eb3}`

Whole challenge is "check the easy steg first." Long hex strings in metadata are almost always wrapping a binary blob — a key, a ZIP, a PNG.

---

# # Timestamped Secrets

**Category:** Cryptography
**Points:** 200
**Files:** `message.txt`, `encryption.py`

`message.txt`:

```
Hint: The encryption was done around 1770242610 UTC
Ciphertext (hex): 71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec
```

`encryption.py`:

{% raw %}
```python
from hashlib import sha256
import time
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

def encrypt(plaintext: str, timestamp: int) -> str:
    timestamp = int(time.time())
    key = sha256(str(timestamp).encode()).digest()[:16]
    cipher = AES.new(key, AES.MODE_ECB)
    padded = pad(plaintext.encode(), AES.block_size)
    ciphertext = cipher.encrypt(padded)
    return ciphertext.hex()

if __name__ == "__main__":
    plaintext = "picoCTF{...}"
    result = encrypt(plaintext, key)
    print(f"Hint: The encryption was done around {timestamp} UTC\n")
    print(f"Ciphertext (hex): {ciphertext.hex()}\n")
```
{% endraw %}

## What's wrong

The interesting line is the key derivation:

{% raw %}
```python
timestamp = int(time.time())
key = sha256(str(timestamp).encode()).digest()[:16]
```
{% endraw %}

The key isn't random — it's `SHA256(unix_timestamp)[:16]`. So the entire keyspace collapses to "which second did the encryption run." Instead of 2^128 possible keys, there's effectively one key per second.

`message.txt` then makes it trivial by handing us the approximate timestamp: `1770242610 UTC`. Even with a generous window of a few thousand seconds, that's a brute force a CPU finishes in well under a second.

Two other small details from the script:
- AES-ECB is used (no IV needed for guessing)
- The plaintext is PKCS#7-padded before encryption — so a correct key produces a clean `unpad`, while wrong keys almost always raise

That second point gives a clean way to detect a hit: if `unpad` succeeds *and* the result contains `picoCTF`, the key is correct.

## Brute force

Walk ±500 seconds around the hinted timestamp, rebuild the key the same way the script does, try AES-ECB decrypt, validate by `unpad` + `picoCTF` substring:

{% raw %}
```python
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

ct = bytes.fromhex("71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec")
hint_ts = 1770242610

for ts in range(hint_ts - 500, hint_ts + 500):
    key = sha256(str(ts).encode()).digest()[:16]
    cipher = AES.new(key, AES.MODE_ECB)
    try:
        pt = unpad(cipher.decrypt(ct), 16).decode()
        if "picoCTF" in pt:
            print(f"Match found at ts={ts}")
            print(f"Flag: {pt}")
            break
    except Exception:
        continue
```
{% endraw %}

Run it:

```bash
python3 solve.py
```

**Flag:** `picoCTF{sa3S_sEc9t_91609b3c}`

AES-128 isn't the problem here — deriving the key from `time.time()` is. Anything seeded by the wall clock can be re-derived by anyone who knows roughly when it ran. Use `secrets.token_bytes(16)` (or `os.urandom(16)`) and there's nothing left to guess.

---

# # Small Trouble

**Category:** Cryptography
**Points:** 200
**Files:** `message.txt`, `encryption.py`

`encryption.py`:

{% raw %}
```python
from Crypto.Util.number import getPrime, inverse, bytes_to_long
import random

# Generate two large primes (1048 bits each)
p = getPrime(1048)
q = getPrime(1048)
n = p * q
phi = (p - 1) * (q - 1)

# compute d
d = getPrime(256)

# Compute the public exponent
e = inverse(d, phi)

# Encrypt a flag
flag = b'picoCTF{...}'
m = bytes_to_long(flag)
c = pow(m, e, n)

# Output for the challenge
with open("message.txt", "w") as f:
    f.write(f"n = {n}\n")
    f.write(f"e = {e}\n")
    f.write(f"c = {c}\n")
```
{% endraw %}

`message.txt`:

```
n = 5373023917854193881488148836268626600844966281094228434713980008645456420843143769937802968738417683912196360187822314641531289831416849843834294947162108743385885201143962499668253981778933829477918509308138753994630824563305549208699656576046998743101679277346011300445088928426443263440407332108573920095572495887336787501868602859232364962889738516998648659421472975661443315919229381772808125492399458662756452065224512066490631968796992359996527817272426440609170087107595860015529546235930679259402315566488964177389716401713764517137452524909467959220676180314062831730087845639199434832005991020618034043100384713370100627
e = 4175583310868652795300389965113565284418414727600716553878889121570817121530394955885654470345498424147652782860439153533821520467215768811709779080727072026933970849370441405003785761938004614591363765462007017522440468621348121750194346049690778543561287741523269727453809859056306759364812353299075154947548484490321009692565003725743721641948720327197056869530828909832774202847759363409628259971009275969423090139856416850801771947794840407742824548811917984728009733304386574925900513975416080481447775928972308069784695578233900533243790494512972121530886816598355680939019684668868018118371194226699363499439201267833521093
c = 2500819896564584617073762334459041098841594691174535309936704141022838857040696686628635484780998955836325607547138326719982847437563730437151322010563996486228600552820177768908015959236557652633765699415304236411634510473064786230642124480616654513063794965767198092919247311938298130695586430013122005242323679167603021982633547351911610485945201515119856661443203318319791821169501171710665128421312624273078300503544919054280485405394300019718867804862663270565462369993000366913519915434789703258524585016270657421220390872493953080216730400140949931138140355541842913665050790770947242092493065556893672338451256709457258034
```

## What's wrong

Standard 2-prime RSA setup. The vulnerability is in this single line:

```python
d = getPrime(256)
e = inverse(d, phi)
```

In a normal RSA setup, you pick a small public exponent (`e = 65537`) and `d` ends up being roughly the same size as `n`. Here, the roles are inverted: `d` is **deliberately picked first**, only 256 bits wide, and `e` is computed as its inverse. That makes `e` enormous (close to `n` in size) — and a small `d` paired with a huge `e` is the textbook **Wiener's attack** trigger.

Wiener proved that when `d < n^(1/4) / 3`, you can recover `d` from the public `(n, e)` alone, using nothing fancier than the continued-fraction expansion of `e/n`.

For a 2096-bit `n`, `n^(1/4)` is roughly 524 bits. A 256-bit `d` sits well inside the vulnerable region.

## Why continued fractions

When `d` is small there exists an integer `k` such that:

```
e · d - k · phi(n) = 1
```

Dividing through by `d · phi(n)` gives `e/phi(n) - k/d = 1/(d · phi(n))`, and because `phi(n) ≈ n`, the public ratio `e/n` ends up extremely close to `k/d`. So `k/d` shows up as one of the convergents of the continued-fraction expansion of `e/n`. Walk the convergents, test each candidate `d`, and one of them will be correct.

The verification trick: pick any small `m` (say `m = 2`) and check `(m^e)^d ≡ m (mod n)`. If equality holds, `d` is the real private exponent.

## Solver

```python
from Crypto.Util.number import long_to_bytes

n = 5373023917854193881488148836268626600844966281094228434713980008645456420843143769937802968738417683912196360187822314641531289831416849843834294947162108743385885201143962499668253981778933829477918509308138753994630824563305549208699656576046998743101679277346011300445088928426443263440407332108573920095572495887336787501868602859232364962889738516998648659421472975661443315919229381772808125492399458662756452065224512066490631968796992359996527817272426440609170087107595860015529546235930679259402315566488964177389716401713764517137452524909467959220676180314062831730087845639199434832005991020618034043100384713370100627
e = 4175583310868652795300389965113565284418414727600716553878889121570817121530394955885654470345498424147652782860439153533821520467215768811709779080727072026933970849370441405003785761938004614591363765462007017522440468621348121750194346049690778543561287741523269727453809859056306759364812353299075154947548484490321009692565003725743721641948720327197056869530828909832774202847759363409628259971009275969423090139856416850801771947794840407742824548811917984728009733304386574925900513975416080481447775928972308069784695578233900533243790494512972121530886816598355680939019684668868018118371194226699363499439201267833521093
c = 2500819896564584617073762334459041098841594691174535309936704141022838857040696686628635484780998955836325607547138326719982847437563730437151322010563996486228600552820177768908015959236557652633765699415304236411634510473064786230642124480616654513063794965767198092919247311938298130695586430013122005242323679167603021982633547351911610485945201515119856661443203318319791821169501171710665128421312624273078300503544919054280485405394300019718867804862663270565462369993000366913519915434789703258524585016270657421220390872493953080216730400140949931138140355541842913665050790770947242092493065556893672338451256709457258034

def rational_to_contfrac(x, y):
    """Continued-fraction expansion of x/y."""
    while y:
        a = x // y
        yield a
        x, y = y, x % y

def convergents_from_contfrac(frac):
    """Yields (numerator, denominator) for each convergent of a continued fraction."""
    n0, d0 = 0, 1
    n1, d1 = 1, 0
    for a in frac:
        n2, d2 = a * n1 + n0, a * d1 + d0
        yield n2, d2
        n0, d0, n1, d1 = n1, d1, n2, d2

def wiener_attack(e, n):
    """Walk convergents of e/n, return d when (2^e)^d ≡ 2 (mod n)."""
    frac = rational_to_contfrac(e, n)
    for k, d_candidate in convergents_from_contfrac(frac):
        if k == 0:
            continue
        if pow(pow(2, e, n), d_candidate, n) == 2:
            return d_candidate
    return None

d = wiener_attack(e, n)

if d:
    print(f"Found d: {d}")
    m = pow(c, d, n)
    print(f"Flag: {long_to_bytes(m).decode()}")
else:
    print("Attack failed. d could not be recovered.")
```

Run:

```bash
python3 sol.py
```

Output:

```
Found d: 99641183421513562958550257557678677794071501743172109932360603363874217034877
Flag: picoCTF{sm4ll_d_7b4790fc}
```

**Flag:** `picoCTF{sm4ll_d_7b4790fc}`

A small `d` is sometimes used to speed up signing on constrained devices (phones, smart cards). Wiener's bound (`d < n^(1/4) / 3`) is the hard line you cannot cross — and Boneh–Durfee later pushed the breakable region all the way to `d < n^0.292`. In short: pick `e = 65537`, let `d` be whatever size it ends up.

---

# # Shift Registers

**Category:** Cryptography
**Points:** 200
**Files:** `chall.py`, `output.txt`

`chall.py`:

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
from Crypto.Random import get_random_bytes

key = bytes_to_long(get_random_bytes(126))

def steplfsr(lfsr):
    b7 = (lfsr >> 7) & 1
    b5 = (lfsr >> 5) & 1
    b4 = (lfsr >> 4) & 1
    b3 = (lfsr >> 3) & 1

    feedback = b7 ^ b5 ^ b4 ^ b3
    lfsr = (feedback << 7) | (lfsr >> 1)
    return lfsr

def encrypt_lfsr(pt_bytes):
    output = bytearray()
    lfsr = key & 0xFF
    for p in pt_bytes:
        lfsr = steplfsr(lfsr)
        ks = lfsr
        output.append(p ^ ks)
    return bytes_to_long(bytes(output))

pt = b"[redacted]"
ct = encrypt_lfsr(pt)

print(long_to_bytes(ct).hex())
```

`output.txt`:

```
21c1b705764e4bfdafd01e0bfdbc38d5eadf92991cdd347064e37444e517d661cea9
```

## What's wrong

A Linear Feedback Shift Register stream cipher. On every byte it shifts the register, computes a feedback bit by XOR-ing taps at bits 7, 5, 4, 3, and XORs the new register value into the plaintext byte to produce one ciphertext byte. Standard construction.

The vulnerability is in the seed:

```python
key = bytes_to_long(get_random_bytes(126))
...
lfsr = key & 0xFF
```

The script generates 126 random bytes (1008 bits of entropy), then immediately throws away everything except the **lowest 8 bits**. The LFSR's effective initial state is 1 byte. That leaves only **256 possible starting states** — trivially brute-forceable.

A second helpful detail: the flag format. picoCTF flags always start with `pico`, so we have a free 4-byte known plaintext. Once the right seed is tried, `b"pico"` will appear in the decoded output. That makes detection unambiguous and removes the need to look at every candidate.

## Brute force

Walk every 8-bit start state (0–255), run the exact same step function as the script, XOR the keystream against the ciphertext, and check for `b"pico"`:

```python
from Crypto.Util.number import long_to_bytes

ct_hex = "21c1b705764e4bfdafd01e0bfdbc38d5eadf92991cdd347064e37444e517d661cea9"
ct_bytes = bytes.fromhex(ct_hex)

def steplfsr(lfsr):
    # Taps at bits 7, 5, 4, 3 (same as chall.py)
    b7 = (lfsr >> 7) & 1
    b5 = (lfsr >> 5) & 1
    b4 = (lfsr >> 4) & 1
    b3 = (lfsr >> 3) & 1

    feedback = b7 ^ b5 ^ b4 ^ b3
    # 8-bit register — keep only the lowest 8 bits
    lfsr = ((feedback << 7) | (lfsr >> 1)) & 0xFF
    return lfsr

for state_guess in range(256):
    lfsr = state_guess
    candidate_pt = bytearray()

    for c in ct_bytes:
        lfsr = steplfsr(lfsr)
        candidate_pt.append(c ^ lfsr)

    if b"pico" in candidate_pt:
        print(f"Found State: {state_guess}")
        print(f"Flag: {candidate_pt.decode(errors='ignore')}")
        break
```

One note on the step function: the script's original `(feedback << 7) | (lfsr >> 1)` doesn't mask to 8 bits, but because the feedback bit gets shifted into bit 7 and `lfsr >> 1` strips the previous bit 0, the effective state stays in [0, 255]. Adding `& 0xFF` in the solver is just defensive.

Run:

```bash
python3 sol.py
```

Output:

```
Found State: <state>
Flag: picoCTF{l1n3ar_f33dback_sh1ft_r3g}
```

**Flag:** `picoCTF{l1n3ar_f33dback_sh1ft_r3g}`

LFSR security has two knobs: the **tap polynomial** sets the period (max `2^n - 1` for a primitive polynomial), and the **state size** sets the brute-force work factor. The taps here are fine — but the seed was masked down to 8 bits, so the work factor collapsed to 256. The fix is one character: drop the `& 0xFF` and seed the register with the full 126 bytes.

---

# # Black Cobra Pepper

**Category:** Cryptography
**Points:** 200
**Files:** `chall.py`, `output.txt`

`output.txt`:

```
d7481d89f1aaf5a857f56edd2ae8994c
8c7d66558130eb5796d131beb43c9934
```

Two 16-byte ciphertexts. The first is `AES(pt1, key)` for a known `pt1 = "72616e646f6d64617461313131313131"` (`"randomdata111111"` in ASCII), the second is `AES(flag, key)` — same key. We need to recover `flag` without ever learning `key`.

`chall.py` (relevant parts — the full file implements every AES primitive by hand):

```python
def split(full_key): ...
def glue(parts):     ...
def rot_word(word):  return str(word[2:]) + str(word[0:2])

def sub_word(word):  return word         # ← no-op
def rcon(word):      return word         # ← no-op
def sub_bytes(state): return state       # ← no-op

def gen_keys(master_key):
    # Standard AES key schedule, but sub_word and rcon do nothing.
    ...

def shift_rows(state): ...
def gmul(a, b):        ...
def mix_columns(s):    ...

def AES(plaintext, key):
    ciphertext = plaintext
    round_keys = gen_keys(key)
    ciphertext = xor(bytes.fromhex(round_keys[0]), bytes.fromhex(ciphertext)).hex()
    for i in range(1, 10):
        ciphertext = to_matrix(ciphertext)
        sub_bytes(ciphertext)        # identity
        shift_rows(ciphertext)
        mix_columns(ciphertext)
        ciphertext = from_matrix(ciphertext)
        ciphertext = xor(bytes.fromhex(round_keys[i]), bytes.fromhex(ciphertext)).hex()
    ciphertext = to_matrix(ciphertext)
    sub_bytes(ciphertext)            # identity
    shift_rows(ciphertext)
    ciphertext = from_matrix(ciphertext)
    ciphertext = xor(bytes.fromhex(round_keys[10]), bytes.fromhex(ciphertext)).hex()
    return ciphertext

flag = [redacted]
key  = [redacted]
pt1  = "72616e646f6d64617461313131313131"

print(AES(pt1, key))
print(AES(flag, key))
```

## What's wrong

At first glance the file looks like a clean from-scratch AES — `gen_keys`, `sub_bytes`, `shift_rows`, `mix_columns`, 10 rounds + final round, all the right pieces in the right order. But every nonlinear primitive is a no-op:

```python
def sub_bytes(state): return state
def sub_word(word):   return word
def rcon(word):       return word
```

`sub_bytes` (the per-round S-box substitution) and `sub_word` / `rcon` inside the key schedule are the *only* nonlinear pieces in AES. Strip them out and what's left — ShiftRows, MixColumns, the key schedule's XOR/rotation, AddRoundKey — is **completely linear over GF(2)**.

## Cancelling the key

For any GF(2)-linear function `F`:

```
F(a ⊕ b) = F(a) ⊕ F(b)
```

So this broken AES satisfies:

```
AES(p, k) = AES(p, 0) ⊕ AES(0, k)
```

That decomposition is the entire vulnerability. With `ct1 = AES(pt1, k)` and `ct2 = AES(flag, k)`:

```
ct1 ⊕ ct2 = (AES(pt1, 0) ⊕ AES(0, k)) ⊕ (AES(flag, 0) ⊕ AES(0, k))
          = AES(pt1, 0) ⊕ AES(flag, 0)
          = AES(pt1 ⊕ flag, 0)             # by linearity again
```

The key term `AES(0, k)` cancels itself out. Now invert one block of the broken AES under a **zero key** and XOR with the known `pt1`:

```
flag = pt1 ⊕ AES_decrypt(ct1 ⊕ ct2, key=0)
```

One known-plaintext pair is enough. The key never has to be recovered.

## Building the inverse

Decryption needs `inv_shift_rows` and `inv_mix_columns` — every other helper (`gen_keys`, `to_matrix`, `from_matrix`, `gmul`, `xor`) comes straight from `chall.py`. There's no `inv_sub_bytes` because `sub_bytes` was identity in the encryption direction, so the inverse is identity too.

## Solver

Helpers (`gen_keys`, `to_matrix`, `from_matrix`, `gmul`, `mix_columns`, `shift_rows`) come straight from the broken script; I added the inverses.

```python
def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

def split(full_key):
    k = full_key
    sub_keys = ["", "", "", ""]
    for i in range(len(k)):
        sub_keys[i % 4] += k[0]
        k = k[1:]
    return sub_keys

def glue(parts):
    k = ""
    for i in range(32):
        k += parts[i % 4][0]
        parts[i % 4] = parts[i % 4][1:]
    return k

def rot_word(word):  return word[2:] + word[0:2]
def sub_word(word):  return word
def rcon(word):      return word

def gen_keys(master_key):
    keys = []
    k = master_key
    for _ in range(11):
        keys.append(k)
        sub_keys = split(k)
        sub_keys[-1] = rot_word(sub_keys[-1])
        sub_keys[-1] = sub_word(sub_keys[-1])
        sub_keys[-1] = rcon(sub_keys[-1])
        sub_keys[0] = xor(bytes.fromhex(sub_keys[0]), bytes.fromhex(sub_keys[-1])).hex()
        sub_keys[1] = xor(bytes.fromhex(sub_keys[1]), bytes.fromhex(sub_keys[0])).hex()
        sub_keys[2] = xor(bytes.fromhex(sub_keys[2]), bytes.fromhex(sub_keys[1])).hex()
        sub_keys[3] = xor(bytes.fromhex(sub_keys[3]), bytes.fromhex(sub_keys[2])).hex()
        k = glue(sub_keys)
    return keys

def to_matrix(key):
    bytes_list = [int(key[i:i+2], 16) for i in range(0, 32, 2)]
    array = [[0]*4 for _ in range(4)]
    for i in range(16):
        array[i % 4][i // 4] = hex(bytes_list[i])[2:]
    return array

def from_matrix(matrix):
    r = ""
    for col in range(4):
        for row in range(4):
            r += matrix[row][col].zfill(2)
    return r

def shift_rows(state):
    state[1][0], state[1][1], state[1][2], state[1][3] = state[1][1], state[1][2], state[1][3], state[1][0]
    state[2][0], state[2][1], state[2][2], state[2][3] = state[2][2], state[2][3], state[2][0], state[2][1]
    state[3][0], state[3][1], state[3][2], state[3][3] = state[3][3], state[3][0], state[3][1], state[3][2]
    return state

def inv_shift_rows(state):
    state[1][0], state[1][1], state[1][2], state[1][3] = state[1][3], state[1][0], state[1][1], state[1][2]
    state[2][0], state[2][1], state[2][2], state[2][3] = state[2][2], state[2][3], state[2][0], state[2][1]
    state[3][0], state[3][1], state[3][2], state[3][3] = state[3][1], state[3][2], state[3][3], state[3][0]
    return state

def gmul(a, b):
    if isinstance(b, str):
        b = int(b, 16)
    p = 0
    for _ in range(8):
        if b & 1:
            p ^= a
        a <<= 1
        if a & 0x100:
            a ^= 0x11b
        b >>= 1
    return p

def mix_columns(s):
    ss = [[0]*4 for _ in range(4)]
    for c in range(4):
        ss[0][c] = hex(gmul(0x02,s[0][c]) ^ gmul(0x03,s[1][c]) ^ int(s[2][c],16) ^ int(s[3][c],16))[2:].zfill(2)
        ss[1][c] = hex(int(s[0][c],16) ^ gmul(0x02,s[1][c]) ^ gmul(0x03,s[2][c]) ^ int(s[3][c],16))[2:].zfill(2)
        ss[2][c] = hex(int(s[0][c],16) ^ int(s[1][c],16) ^ gmul(0x02,s[2][c]) ^ gmul(0x03,s[3][c]))[2:].zfill(2)
        ss[3][c] = hex(gmul(0x03,s[0][c]) ^ int(s[1][c],16) ^ int(s[2][c],16) ^ gmul(0x02,s[3][c]))[2:].zfill(2)
    for i in range(4):
        for j in range(4):
            s[i][j] = ss[i][j]
    return s

def inv_mix_columns(s):
    ss = [[0]*4 for _ in range(4)]
    for c in range(4):
        ss[0][c] = hex(gmul(0x0e,s[0][c]) ^ gmul(0x0b,s[1][c]) ^ gmul(0x0d,s[2][c]) ^ gmul(0x09,s[3][c]))[2:].zfill(2)
        ss[1][c] = hex(gmul(0x09,s[0][c]) ^ gmul(0x0e,s[1][c]) ^ gmul(0x0b,s[2][c]) ^ gmul(0x0d,s[3][c]))[2:].zfill(2)
        ss[2][c] = hex(gmul(0x0d,s[0][c]) ^ gmul(0x09,s[1][c]) ^ gmul(0x0e,s[2][c]) ^ gmul(0x0b,s[3][c]))[2:].zfill(2)
        ss[3][c] = hex(gmul(0x0b,s[0][c]) ^ gmul(0x0d,s[1][c]) ^ gmul(0x09,s[2][c]) ^ gmul(0x0e,s[3][c]))[2:].zfill(2)
    for i in range(4):
        for j in range(4):
            s[i][j] = ss[i][j]
    return s

def AES_decrypt(ciphertext, key):
    state = ciphertext
    rks = gen_keys(key)
    state = xor(bytes.fromhex(rks[10]), bytes.fromhex(state)).hex()
    state = to_matrix(state)
    inv_shift_rows(state)
    state = from_matrix(state)
    for i in range(9, 0, -1):
        state = xor(bytes.fromhex(rks[i]), bytes.fromhex(state)).hex()
        state = to_matrix(state)
        inv_mix_columns(state)
        inv_shift_rows(state)
        state = from_matrix(state)
    state = xor(bytes.fromhex(rks[0]), bytes.fromhex(state)).hex()
    return state

# --- exploit ---
pt1 = "72616e646f6d64617461313131313131"
ct1 = "d7481d89f1aaf5a857f56edd2ae8994c"
ct2 = "8c7d66558130eb5796d131beb43c9934"
zero_key = "00" * 16

delta    = xor(bytes.fromhex(ct1), bytes.fromhex(ct2)).hex()
diff_pt  = AES_decrypt(delta, zero_key)
flag_hex = xor(bytes.fromhex(pt1), bytes.fromhex(diff_pt)).hex()
print(bytes.fromhex(flag_hex).decode())
```

Run:

```bash
python3 sol.py
```

Output:

```
picoCTF{spi1cy!}
```

**Flag:** `picoCTF{spi1cy!}`

Strip the S-box and "AES" stops being AES. The S-box is the *only* nonlinear component in the cipher — without it, encryption becomes a GF(2)-linear map, every algebraic shortcut opens up, and a single known-plaintext pair is enough to cancel the key out completely. This is exactly why every modern block cipher leans heavily on its S-box (or an equivalent nonlinear layer). Lose nonlinearity, lose the cipher.

---

# # ClusterRSA

**Category:** Cryptography
**Difficulty:** Medium
**Files:** `message.txt`

`message.txt`:

```
n  = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e  = 65537
ct = 2630159242114455882250729812770100011736485763047361297871782218963814793905003742546116295910618429
```

Description: *"a message has been encrypted using RSA, but this time something feels... more crowded than usual."*
Hints: *"RSA usually means two primes... but what if someone got greedy?"* and *"prime factor decomposition"*.

## What's wrong

Standard textbook RSA uses exactly two large primes — `n = p · q` — and security relies on factoring `n` being computationally infeasible. For a 97-digit (~322-bit) `n`, two-prime RSA would already be considered weak by modern standards but still inconvenient to factor without specialized hardware/time.

The wording — *"more crowded than usual"* / *"greedy"* — and the size of `n` itself point at **multi-prime RSA**: instead of two primes, the modulus is built from three, four, or more. This is a real optimisation (it speeds up CRT-based decryption), but at a fixed total bit-length each individual prime gets smaller, and small primes are dramatically easier to find. With four 25-digit primes, the largest factor an attacker has to find is itself only 25 digits — well within trial division / Pollard's rho range, and almost certainly already in factordb.

The "Cluster" in the name is the giveaway: the modulus is a *cluster* of primes rather than a pair.

## Factoring n

Trial division up to 10M produced nothing. Pollard's rho / Brent would have worked given a few minutes, but the modulus is small enough to be a likely factordb hit. A query to **factordb.com** returned the full factorisation immediately:

```
n = p1 · p2 · p3 · p4

p1 = 9671406556917033397931773
p2 = 9671406556917033398314601
p3 = 9671406556917033398439721
p4 = 9671406556917033398454847
```

Four primes, each ~25 digits. Notice they're all close to `9.67 × 10^24` — the prime generator was clearly clustering candidates around a base value, which is why the title is "Cluster".

## Decrypting

Decryption is identical to 2-prime RSA, but the totient generalises to a product over all the prime factors:

```
phi(n) = ∏ (pi - 1)
```

From there: `d = e^(-1) mod phi(n)`, then `m = c^d mod n`, then convert the integer back to bytes.

## Solver

```python
n  = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e  = 65537
ct = 2630159242114455882250729812770100011736485763047361297871782218963814793905003742546116295910618429

primes = [
    9671406556917033397931773,
    9671406556917033398314601,
    9671406556917033398439721,
    9671406556917033398454847,
]

# phi(n) = product of (p_i - 1) for all prime factors
phi = 1
for p in primes:
    phi *= (p - 1)

# Recover the private exponent and decrypt
d = pow(e, -1, phi)
m = pow(ct, d, n)

# Convert the integer back to bytes
h = hex(m)[2:]
if len(h) % 2:
    h = '0' + h
print(bytes.fromhex(h).decode())
```

Run:

```bash
python3 sol.py
```

Output:

```
picoCTF{mul71_rsa_8c9fb77d}
```

**Flag:** `picoCTF{mul71_rsa_8c9fb77d}`

Multi-prime RSA splits the same bit-budget across more primes, which makes each prime smaller and the modulus dramatically easier to factor. The math is unchanged — only `phi(n)` generalises. Two practical workflow notes: (1) for any RSA modulus under ~512 bits, **check factordb first** before spinning up Pollard's rho or CADO-NFS; (2) modulus sizes that look "weird" for a 2-prime setup (too small, or factors clearly clustered) are usually a multi-prime tell.

---

# # Not TRUe

**Category:** Cryptography
**Points:** 400
**Files:** `encrypt.py`, `public.txt`

Title: "Not TRUe" → NTRU. The hint confirms it: *"is there a lattice attack that allows you to compute the private key given the public information?"*

`encrypt.py`:

{% raw %}
```python
from random import randint
from sage.all import *

N = 48
p = 3
q = 509

R = PolynomialRing(ZZ, 'x')
x = R.gen()
R_modq = PolynomialRing(Integers(q), 'x').quotient(x**N - 1, 'xbar')
R_modp = PolynomialRing(Integers(p), 'x').quotient(x**N - 1, 'xbar')

def gen_poly():
    return R([randint(-1, 1) for _ in range(N)])

def gen_msg(text):
    binary_str = ''.join(format(ord(char), '08b') for char in text)
    padding_length = (N - (len(binary_str) % N)) % N
    binary_str += '0' * padding_length
    chunks = [binary_str[i:i+N] for i in range(0, len(binary_str), N)]
    polynomials = [R([int(bit) for bit in chunk]) for chunk in chunks]
    return polynomials

def encrypt(h, m):
    r = gen_poly()
    return R_modq(p*(h*r) + m)

def generate_keys():
    while True:
        f = gen_poly()
        g = gen_poly()
        try:
            f_p_inv = R_modp(f)**-1
            f_q_inv = R_modq(f)**-1
            break
        except:
            continue
    h = R_modq(p*(f_q_inv*g))
    private_key = (f, g, f_p_inv, f_q_inv)
    public_key = h
    return public_key, private_key

with open("flag.txt", "r") as f:
    flag = f.read().strip()

public_key, private_key = generate_keys()
print(f"h = {public_key.list()}")

ciphertext = []
encoded = gen_msg(flag)
for part in encoded:
    ciphertext.append(encrypt(public_key, part))
ct = [c.list() for c in ciphertext]
print(f"ct = {ct}")
```
{% endraw %}

`public.txt`:

```
N = 48
p = 3
q = 509
h = [171, 218, 364, 196, 397, 127, 385, 467, 30, 314, 298, 455, 307, 40, 97, 200, 347, 301, 219, 258, 432, 469, 155, 311, 431, 184, 276, 170, 125, 403, 49, 326, 125, 416, 409, 506, 226, 441, 245, 373, 426, 503, 420, 298, 449, 453, 243, 206]
ct = [[461, 481, 501, 448, 105, 391, 58, 322, 90, 349, 410, 60, 502, 116, 410, 262, 416, 479, 291, 90, 410, 226, 253, 25, 233, 286, 474, 255, 165, 274, 491, 285, 14, 485, 7, 400, 53, 23, 155, 98, 322, 35, 16, 90, 103, 472, 204, 62], [291, 437, 68, 214, 302, 166, 368, 455, 132, 49, 396, 265, 452, 488, 377, 334, 139, 232, 283, 277, 258, 14, 14, 161, 388, 176, 288, 339, 291, 315, 484, 49, 51, 421, 350, 492, 389, 97, 229, 431, 426, 285, 292, 412, 255, 1, 130, 39], [179, 159, 95, 353, 267, 160, 349, 483, 413, 480, 482, 103, 331, 499, 417, 255, 136, 352, 384, 235, 387, 371, 132, 108, 498, 84, 29, 267, 157, 424, 453, 74, 219, 151, 470, 104, 89, 348, 166, 96, 304, 424, 447, 124, 413, 273, 401, 333], [6, 387, 353, 98, 2, 486, 96, 387, 316, 317, 472, 289, 330, 169, 308, 427, 465, 344, 457, 249, 383, 380, 2, 407, 265, 48, 391, 142, 157, 340, 3, 0, 469, 245, 346, 172, 47, 175, 188, 145, 238, 436, 210, 369, 259, 135, 350, 167], [150, 202, 32, 135, 325, 152, 498, 59, 67, 81, 6, 14, 158, 328, 289, 292, 463, 255, 217, 295, 258, 418, 151, 56, 472, 19, 266, 184, 176, 259, 411, 470, 96, 319, 303, 459, 46, 159, 464, 287, 306, 215, 349, 23, 384, 322, 313, 262], [367, 133, 319, 404, 341, 202, 195, 406, 496, 232, 281, 17, 18, 91, 257, 460, 263, 445, 203, 329, 27, 199, 499, 454, 467, 345, 52, 464, 93, 410, 35, 15, 226, 263, 198, 166, 441, 59, 193, 96, 49, 486, 297, 72, 304, 150, 40, 140]]
```

The flag is encoded as bits, padded to a multiple of `N`, chunked into N=48-bit polynomials with coefficients in `{0, 1}`, and each chunk is encrypted separately. Six ciphertext blocks total, public `h`, and parameters `(N, p, q) = (48, 3, 509)`.

## What's wrong

Standard NTRU is a lattice-based cryptosystem whose security depends on finding a short vector `(f, g)` in a 2N-dimensional lattice built from the public key `h`. Real NTRU parameter sets use **N = 701** (NTRU-HRSS) or **N = 1277** (NTRU Prime) — at those sizes the lattice is well out of reach for LLL or even BKZ with reasonable block sizes.

This challenge uses **N = 48**, which gives a 96×96 lattice. LLL solves SVP-approximations in lattices of that dimension in seconds. The chosen parameters are the entire bug — the construction is otherwise textbook NTRU.

## NTRU recap

Everything happens in the polynomial ring `Z[x] / (x^N - 1)`, with coefficients reduced mod `q` (large) or mod `p` (small). All polynomial multiplication is cyclic convolution.

- **Private key:** ternary polynomials `f`, `g` (coefficients in `{-1, 0, 1}`)
- **Public key:** `h = p · f_q_inv · g  mod q` (where `f_q_inv` is the inverse of `f` mod `q`)
- **Encrypt:** `c = p · h · r + m  mod q` for a fresh random ternary `r`
- **Decrypt:** `a = f · c  mod q`, centered to `(-q/2, q/2]`, then `m = f_p_inv · a  mod p`

Why decryption works:
```
f · c = p · (f · h) · r + f · m
      = p · (p · g) · r + f · m              (since h = p · f_q_inv · g, so f · h = p · g)
      = p² · g · r + f · m
```
Modulo `p`, the `p² · g · r` term vanishes (it has all coefficients divisible by `p`), leaving `f · m mod p`. Multiplying by `f_p_inv` recovers `m`.

## The attack — NTRU lattice + LLL

The public-key relation `h · f ≡ p · g  (mod q)` means that **`(f, g)` is a vector in a specific 2N-dimensional integer lattice**. Build the basis:

```
L = [ I_N     H    ]
    [  0    q · I_N ]
```

where `H` is the **circulant matrix** of `h` — each row is a cyclic shift of `h`'s coefficient vector. Polynomial multiplication mod `(x^N - 1)` is exactly multiplication by a circulant.

Why `(f, g)` lies in the row span: any combination `f · L` gives `(f · I_N | f · H) = (f | f · h mod q) = (f | p · g)`, so `(f, p·g)` (and equivalently `(f, g)` after dividing the right half by `p`) is reachable.

Why it's **short**: ternary coefficients give an expected norm of about `√(2N/3) ≈ 5.6`. The Gaussian-heuristic norm for a random vector in this lattice is roughly `√(N · q) ≈ √(48 · 509) ≈ 156`. The target is ~30× shorter than expected — exactly the kind of gap LLL is built to find.

## Solver

```python
from sage.all import *

N = 48; p = 3; q = 509
h = [171,218,364,196,397,127,385,467,30,314,298,455,307,40,97,200,347,301,219,258,
     432,469,155,311,431,184,276,170,125,403,49,326,125,416,409,506,226,441,245,373,
     426,503,420,298,449,453,243,206]

ct_blocks = [
    [461,481,501,448,105,391,58,322,90,349,410,60,502,116,410,262,416,479,291,90,
     410,226,253,25,233,286,474,255,165,274,491,285,14,485,7,400,53,23,155,98,322,
     35,16,90,103,472,204,62],
    [291,437,68,214,302,166,368,455,132,49,396,265,452,488,377,334,139,232,283,277,
     258,14,14,161,388,176,288,339,291,315,484,49,51,421,350,492,389,97,229,431,426,
     285,292,412,255,1,130,39],
    [179,159,95,353,267,160,349,483,413,480,482,103,331,499,417,255,136,352,384,235,
     387,371,132,108,498,84,29,267,157,424,453,74,219,151,470,104,89,348,166,96,304,
     424,447,124,413,273,401,333],
    [6,387,353,98,2,486,96,387,316,317,472,289,330,169,308,427,465,344,457,249,383,
     380,2,407,265,48,391,142,157,340,3,0,469,245,346,172,47,175,188,145,238,436,
     210,369,259,135,350,167],
    [150,202,32,135,325,152,498,59,67,81,6,14,158,328,289,292,463,255,217,295,258,
     418,151,56,472,19,266,184,176,259,411,470,96,319,303,459,46,159,464,287,306,
     215,349,23,384,322,313,262],
    [367,133,319,404,341,202,195,406,496,232,281,17,18,91,257,460,263,445,203,329,
     27,199,499,454,467,345,52,464,93,410,35,15,226,263,198,166,441,59,193,96,49,
     486,297,72,304,150,40,140]
]

def circ(h, i):
    return [h[(j - i) % N] for j in range(N)]

rows = []
for i in range(N):
    r = [0]*(2*N)
    r[i] = 1
    for j in range(N):
        r[N+j] = circ(h, i)[j]
    rows.append(r)
for i in range(N):
    r = [0]*(2*N)
    r[N+i] = q
    rows.append(r)

L = matrix(ZZ, rows).LLL()

def pmul(a, b, mod, n):
    res = [0]*n
    for i in range(n):
        for j in range(n):
            res[(i+j) % n] = (res[(i+j) % n] + a[i]*b[j]) % mod
    return res

def center(v, mod):
    return [(x if x <= mod//2 else x - mod) for x in v]

R_p = PolynomialRing(GF(p), 'x').quotient(x^N - 1)

for row in L:
    f = list(row[:N])
    if all(c in (-1, 0, 1) for c in f) and any(c != 0 for c in f):
        try:
            f_p_inv = list(R_p(f)^-1)
            while len(f_p_inv) < N:
                f_p_inv.append(0)
            bits = ""
            ok = True
            for blk in ct_blocks:
                a   = center(pmul(f, blk, q, N), q)
                a_p = [((x % p) + p) % p for x in a]
                m   = [x % p for x in pmul([int(c) for c in f_p_inv], a_p, p, N)]
                if not all(b in (0, 1) for b in m):
                    ok = False
                    break
                bits += ''.join(str(b) for b in m)
            if ok:
                flag = ''.join(chr(int(bits[i:i+8], 2)) for i in range(0, len(bits)-7, 8))
                print("FLAG:", flag.rstrip('\x00'))
                break
        except Exception:
            continue
```

Run under SageMath — either a local install or sagecell.sagemath.org:

```bash
sage solve.sage
```

Output:

```
FLAG: picoCTF{th4ts_s0_N0t_TRU3_0729b9da}
```

**Flag:** `picoCTF{th4ts_s0_N0t_TRU3_0729b9da}`

NTRU's security rests entirely on the hardness of finding short vectors in the NTRU lattice. At N=48 the lattice is small enough that LLL recovers `(f, g)` in seconds. Production NTRU parameter sets (N=701, N=1277) put the lattice well beyond LLL/BKZ reach — the same construction is sound at the right size. Two takeaways: (1) the title is the hint — when a challenge name plays on a known cipher, *that's the cipher*; (2) any time a cryptosystem has a "short secret in a structured lattice" property, LLL is the first thing to try.

---

# # MSS Advance Revenge

**Category:** Cryptography
**Difficulty:** Hard

Description: *"last time we went easy on you. you'll never get the flag this time!"* Hint: *"what are lattices?"*

## Setup from chall.py

```python
MASTER_KEY = hashlib.sha256(flag).digest()
coeffs = [bytes_to_long(MASTER_KEY)]
for i in range(29):
    co = hashlib.sha256(long_to_bytes(coeffs[-1])).digest()
    coeffs.append(bytes_to_long(co))
```

A 1024-bit prime `p`. Polynomial has 30 coefficients, all SHA-256 outputs (so 256 bits each). The first coefficient is `bytes_to_long(sha256(flag))` — that's what we recover.

```python
pairs = []
for i in range(20):
    x = random.randint(0, p)
    y = eval_poly(x, coeffs)
    pairs.append((x, y))
```

The polynomial is then evaluated at **20** random points mod p. The flag is finally AES-CBC-encrypted with `MASTER_KEY`.

## Why interpolation fails

Recovering 30 coefficients from 20 evaluations is underdetermined — infinitely many degree-29 polynomials fit. Linear algebra alone won't do it.

## The structural weakness

`p` is 1024 bits but every coefficient is 256 bits — about 4× shorter than a "random" coefficient mod `p`. That asymmetry is what LLL exploits.

## Lattice construction

For each pair `(x_i, y_i)`:
```
c_0 · x_i^29 + c_1 · x_i^28 + ... + c_29  ≡  y_i  (mod p)
```

Define `R[i] = (x_i^29, x_i^28, …, x_i^0, -y_i) mod p` and the secret vector
```
u = (c_0, c_1, …, c_29, 1)
```
Then `R · u ≡ 0 (mod p)`, so `u` lies in the right kernel of the 20×31 matrix `R` over `GF(p)`.

`u` has norm ~`√30 · 2^256 ≈ 2^260`. The Gaussian-heuristic norm for this 31-dim lattice is `(p^20)^(1/31) = p^(20/31) ≈ 2^661`. The target sits roughly `10^121`× shorter than expected — LLL surfaces it as the first reduced row.

### Building a 31×31 integer basis

1. RREF `R` over `GF(p)` → 20 pivot columns, 11 free columns.
2. Read off 11 null-space vectors (a `1` in the free column, `0` in the others, `-RREF` values in the pivots).
3. Stack them on top of `p · e_j` for each pivot column `j`.

That's a clean integer basis for the lattice — no HNF needed.

## Attack

1. Build the 20×31 matrix `R`.
2. Gaussian-eliminate over GF(p) for the 11-dim null space.
3. Stack 11 null vectors + 20 pivot rows → 31×31 basis.
4. Run LLL with high mpfr precision (entries are ~1024-bit).
5. Top reduced row is `±u = ±(c_0, …, c_29, 1)`.
6. `MASTER_KEY = c_0.to_bytes(32, 'big')`.
7. AES-CBC decrypt → flag.

## Solver

{% raw %}
```python
#!/usr/bin/env python3
import sys, hashlib
from fpylll import IntegerMatrix, LLL, GSO, FPLLL
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from Crypto.Util.number import long_to_bytes, bytes_to_long

p = 150233963364811123102560869891204445059343246972167500039119137622001848527229247571288652204631455945092273380806032327077651738068540910843410504854827674845314143508453173296654376806571766597620963791745111797012208241509139917759823365206942453706659122856724967732136257811616585841754878847472198826669

pairs = [(57483385504637565708312069240970284372392590407833530947885251909611617078422540315509448539320629509063483865108443088184264805175147759611152846554076260142833252113543717827897216068612240527926696393016361205367557102601429391094360413778740692501531581260529634749249847668223548810382717733624576842778, 138968714166339241836981729538971185372378134691158067595545467293006121988985804245915400594715942227510087764779824825388605291036505527435260047531080882173972029990549019413447280797438552743191258845200531798437652535883431720569077755912802376652369439247016511759499068800476907566002386408259350201066), (125976382964055357666395625913501866889717852756505486984828636311938695478480252994260161854401817401212564499038490028261628019604546411402446142681750114675555042230847725222812180450399829338898242980528514924323831245448040202313502105931483404362223399657693222109326276618427357189387606562139341142413, 40011302976960626919956893637583874081724540672450205486512747101888415379367889144448246893201910702834903399660582089604446014681917177447039994953526900903424224427324873754299366707039689394290564340692155700041217478913139034183974295434906362257300813156856106692195812720925915402307506261423031145105), (46833889339708166009373633214694450947166244658350514176589552236977224403344899334211807699959819331324393990034094102727860005515659591185095721137931377825936706221428436778620648559831940363460864885410003222678352343561060492792792469451840221305554697197748494807435925068262176035852333513898367325719, 64048247977778447117266213382716215295651789659115873831980210015783343833744600847236117519918441176622432731837423101258384806257098399092134228756097313240093511397677665041027863077754018456710822942719302983548353683295014497748729981829588319199298984066726045819205927814185608409632651886856833375387), (20629479609007950587633767809124250677504241875356377464059318560770645639362603335742578200776447528047579280724056974543993941258365712441048639365976076303267251614708703974490361281156652241459309400272570770225208121556084273479707862186839302368443819437721079291378061343261159465657426847049459492903, 141041573326741168760972020490575406265807714424555879957356619571960658252362519349036141223975082490829723496929191847720840346639260531490513358227881932315737101656758922896453817265491104665003609171306457206216346536258559526447944118182251048827148510522450115597857142234635656669805567570647614638778), (139206689051499226531941646620108840418650885505865867962675300842662945317566083558175833603084145001897194839320153816527349598263758533198566775292887265653749002536970655705467303961369876146256084290604350489503912003914822693682613927001056894691006172686582198651472747590514240808214127690240336439745, 110958512256419240089792856363768146203412455925993195177503220742344335341433810919118179378979561600409681936631836051506354990212808691762584991969232552178715935479828015034536384876541997253550173170170879816482643528694581311125179003203763343494874212551462200740663808488173030813247736145356023893215), (38666811738675121108905598750361412030786675081545329198757237062142392893849470730774766892426317409235669913646562979902986392560664924352040112563732374489314741612779397845062633912471016020353842249653986931976025470550112792428557987128452770708074591454007880458610317744230868153138685981596717880889, 75336160676307453570900750969199707684527904455743884995619487203620757588342174647505197939590758917804936643223220654379461312049597757261061503139043549336539020747490374852984465134940671840370604391198140769370969580722281261145190230911889976657399466512176318892791428142370724985126099645834873351415), (1303173830584953017842193012950165921030466624092484174372600734350360086848957967054646198070337485352214042930180254427839740150929557681803314004205466156404345905421592601900530638328037123577188904678056917591227334737012583362477488487044303715708904533824674756622773050997356260884644816097490846881, 145904952368705876958391464107430613797798793212156317625081701580783697910860002904767472723132898442505351429701237507454760930868582922525089959701291951679175174780237490598880295928253073103324206387366356631706906089358605850725935379786827672107714445210020626854077852335848013325173222704445213110193), (115938126286740882463247597389814278118448912241561126401412654952365759372879389717584063965209474953774627744226208806888671803425379348477161770813788707977986849626543629366789086887342037531558433923317281417835154541604835396909151661418865740322681982266893379161101419711640476456790304345718772771250, 134504531235205418118426728040710413732022607848886697388433286308333579071581023560227456364446671981362135395005941108813617610123099780485592266622320890706312404864064615277019647866829244728741201447859744368497068578857469055127166032591212897864407513612397971273295923304053841620008863552493243671070), (119685630867678350497164684474801586548009002684350422621546664129527607397641259506406862509683912153031396176426516255897739787823426532473071453990939247649895301684794301388610668231500263909135278890467243265575952199516408364867370570368982153440639444286218923468775548365572660695593039349126793066563, 32532236370126868546570159162111751393031586963713688920935162940585130962399210878543497753067621372569516257887845187914136413905433617072428616121672859023816116522039783798691523524443992012041862553299148238805925553780474506004765105299028478719580089959469713578834378514538137954546875117022236117274), (11601182366471176660175557414279226028320148060009810021029464479147637115219425757694878119003629472428685883482408644202958153305159449302982120291039053576610341520928599868432347664284904317600166446605878410243643287808610081658731871812391011250343958771584795053112529037390724861615904555300097215665, 45597537848321997282824244496322833837765616963800246772239296934303966160263244382325249186203153261379825169191439661916110988795750721746159994643973671677874797920932975919939238949253444349999410438914096267458124403510754603471221205183290673168006989779434654688123487272196710226580475623197290488205), (126650650318598704434185602138717988274295388402603540213804403842690505983174630591901557400216314888391705161489495044171250044438455842507217563952078117941083103618406067490200727002569971052842029484962000408487375456655860437422450554301953996492469688798372667423529098555607562128688680207032811560374, 129505744534002866843389827926362363815035976388104719537745766006464086522903963897640888541888658040701230915872496597082828308219293975202243078397251262398810238627368211745107169579961453910292029775242207710080092507553815892427543407058076712012233541425638788643807509959344167364376116943922901076683), (128025055132632388630664208760060371694370132606487519009339921570850458851730083433291918241820586872522990600212053856965595945557438717625751972827378287321560227849875007020139663313661428232930326868080964926597993379808223644922097915573210655944133959024718047009096849351079880901011526842421895407877, 89156129830425869474740428211274051194756566374029053672099044706554050880717873397062733212287196390204878654229484557863995328679395170620544573003985134120082086339571385318056234359414751649180354132563580874933164391550728568473906231527653962374778691484493376863065881094402892477138599504894021726600), (30275278536220586139268244668661090186029897148415169410968531312968369678075652920722093887103179594166629440419788371226429357516797122328399382264930444789225753960712557887925303599465840847191234759961539156003085513128754262854436857691498761533983171611855167968360223827428029281210952061596696134813, 111902270206822184956895119987139629486773826634980690248044515147592521710882485011099324818865413174650343589076296618502165047563669249317320831930730290168782976638647217140990278312063393667854156427852609389465527223606754119373106846846452933084432461884785973030928941661875459443088973468766543721317), (37384513812172304382113714313159621302343237594524304370424142111648022560778384597665767052680037088713255603348989587723879303146311157416446056982184722593987857661858241138432349480104350827871931521538549227006408166013805612026496518209758965662837989444461491194764933461689336366367101969844147900877, 52872821348229002977205825905524309850298317503493724451602117605833122907214500900513541058190895379471572893048888688373752392067536357829974834536365760918632608793950393717404952232708963247109858639298968354761477100497528500917384130747488466095124253235444009076284430587227286084886332146435308649748), (138890616024905269406690254836035411892123307895183792149986215345458148746691546920592177143648218601163722222249603344132204939441682448709951985048449928640504786029803956178726702000970939058602523148356619285539264767988958834500947084939876639271048180762734001196756997103974690955677867674587673120640, 133799252755117827938090052911823559137508125639191628342911695588456257960196532889257536219251846963366345100550602687293474220724297553962873581382357085797860985501801887916642303190844877172480613547982232122910613106974436772115925696696719744115445772702259691238249862638736868167198339275810206837010), (133657514678375591157390245756184257133097031065618964845605199778615123129636914447169999352917035147063792275844525669303964086003842243601002949903538446035892659481109959225783928088915078963187618893785938288868829081886712298283504189154373173463135031313248526564858698762834992522486082849696787088561, 143751548491798160655986549106040694382071948577996499558483050910480273680077187063280748386249769956661196458971591661499156099067766451151781407033131051831314069830607378259005802924441765528662935406624423114153261030705065123964497789149173645631385563306504480935316319666747747633399624395586517330704), (130949081649821984865890222143869498436585809939507348840515098213664558173219137705777711371304371684348458627306113575007311873606965772163163967400814090817391442327586949966680786189873947188298240162060602922958420531063428897976848580308633934329792404244484799872042349498500307648975683089028483361359, 89302373431010463170228189496629651577093117713377963821015309835984096529152669682801041581261796295137633343099737756808968357232748890417084863043689577674107843651747748013901324322513373483848549506093882226316126088778453250809891311787131704292499719525980248126316927284347566992806775246124012983211), (13922844849989479863592822228662221609386627640181126897258351421336935806015692396267285763335558004100976086604303809070924535454047636249209205509804339458820306734566730696862916886233720757336331682141219098213691504325261199250083014907891307941797953273732375173044464681498874353635670393353928992755, 20585849323315174701225594622676822148896393442825714608523358624225546353316051487028083390642759712556029792474527102442295127983566000764221320486629275281098368470171597080324484728596645544928040854634988872777232250279511333409881679171853479902664899356965608796774306338414630031930676842034573611564), (127569739666600322018252593830193445381512025223259738562007407570465059819289290514316978777811455981156433218807980914475051807678074317958047381614437580492520524517547225382995832170852533534765861222817339985037940419800358200006401704347149510200489159251829091572160763100544426383603322660786639165173, 23803068771080158608514432374953365149938386778152568624546845711842134082412609201267040048509057934039190750645452863965185994001005295550441187827468748503513277064511336261105189749749215636715953417643762259060171624121427188459320403494112387640148881566943097930059254463471340539353650841788868859340), (74503114546407544777412468137796167393351152448581874005630061566090601240758200507388313139849223917479735361312339483280607799025922704799486550629882464793577640761068782005715329119183817395532161839559651925330928097375437128745628687611254830820675731724341947411597393194921435322658327623510127304268, 103936917442011191932464416259579313292154641618078760401714161905862916078472840888629463873099236806495979691042700278518957443067735754238481647592790753091412991076558995616492624863555962638491015842328549409624170195048184648458236563293859397835290922013315530158796187742129769770675287135757234253087)]

enc_flag = ('00000000000000000000000000000000',
            '20773a1ecc72c4561eb26c037865bfe99bab18166f5e67885766551e1fb55dbee70f8f18931c28c55f58bf62da487b7f67d880acb03c3324e4a3eea51faa0d59fd7c84a06c770cae02e2a30d1056492a')

xs = [pair[0] for pair in pairs]
ys = [pair[1] for pair in pairs]
n, m, dim = 30, 20, 31

# 20×31 matrix over GF(p)
R_rows = []
for i in range(m):
    row = [pow(xs[i], n - 1 - j, p) for j in range(n)]
    row.append((-ys[i]) % p)
    R_rows.append(row)

# RREF over GF(p)
M = [list(r) for r in R_rows]
pivot_cols, row_ptr = [], 0
for col in range(dim):
    if row_ptr >= m:
        break
    piv = next((r for r in range(row_ptr, m) if M[r][col] % p), None)
    if piv is None:
        continue
    M[row_ptr], M[piv] = M[piv], M[row_ptr]
    inv = pow(M[row_ptr][col], -1, p)
    M[row_ptr] = [x * inv % p for x in M[row_ptr]]
    for r in range(m):
        if r != row_ptr and M[r][col] % p:
            f = M[r][col]
            M[r] = [(M[r][j] - f * M[row_ptr][j]) % p for j in range(dim)]
    pivot_cols.append(col)
    row_ptr += 1

free_cols = [j for j in range(dim) if j not in pivot_cols]

# Null-space vectors
null_vecs = []
for fc in free_cols:
    v = [0] * dim
    v[fc] = 1
    for i, pc in enumerate(pivot_cols):
        v[pc] = (-M[i][fc]) % p
    null_vecs.append(v)

# 31×31 lattice basis
basis_rows = list(null_vecs)
for pc in pivot_cols:
    v = [0] * dim
    v[pc] = p
    basis_rows.append(v)

# LLL
FPLLL.set_precision(3200)
A = IntegerMatrix(dim, dim)
for i in range(dim):
    for j in range(dim):
        A[i, j] = basis_rows[i][j]

M_gso = GSO.Mat(A, float_type="mpfr")
M_gso.update_gso()
LLL.Reduction(M_gso, delta=0.99, eta=0.51)()

# Recover the flag
for i in range(dim):
    row  = [int(A[i, j]) for j in range(dim)]
    last = row[-1]
    if abs(last) != 1:
        continue
    sign  = last
    c_vec = [sign * row[j] for j in range(n)]
    if not all(cj >= 0 for cj in c_vec):
        continue

    # Verify against the SHA-chain
    coeffs = [c_vec[0]]
    for _ in range(29):
        co = hashlib.sha256(long_to_bytes(coeffs[-1])).digest()
        coeffs.append(bytes_to_long(co))

    y_test = 0
    for j in range(n):
        y_test = (y_test * xs[0] + coeffs[j]) % p
    if y_test != ys[0]:
        continue

    master_key = c_vec[0].to_bytes(32, 'big')
    iv     = bytes.fromhex(enc_flag[0])
    ct     = bytes.fromhex(enc_flag[1])
    cipher = AES.new(master_key, AES.MODE_CBC, iv)
    flag   = unpad(cipher.decrypt(ct), 16)
    print(flag.decode())
    sys.exit(0)
```
{% endraw %}

```bash
pip install fpylll cysignals pycryptodome
python3 solve.py
```

**Flag:** `picoCTF{MSS_Advance_but_we_brought_it_back_and_made_it_harder!!!}`

The "harder" part of the challenge is making the system underdetermined (20 < 30). The original LLL-friendly structure — coefficients much smaller than `p` — is still there, and that's the real bug.
