---
layout: default
title: "picoCTF challenges - writeUps"
# date: 2025-09-16
categories: ctf Blockchain Solidity
---

**Challenge divisions :** [MEDIUM](#medium-challs) . [EASY](#easy-challs) . 

## EASY Chall's 

# Even RSA can be Broken

**CHALLENGE** :
[https://play.picoctf.org/practice/challenge/470?category=2&page=1](https://play.picoctf.org/practice/challenge/470?category=2&page=1)

![CHALLENGE](/images/EVEN_RSA_CAN_BE_BROKEN.PNG)

**Program's Source Code  (given) :** 
```python 
from sys import exit
from Crypto.Util.number import bytes_to_long, inverse
from setup import get_primes

e = 65537

def gen_key(k):
    """
    Generates RSA key with k bits
    """
    p,q = get_primes(k//2)
    N = p*q
    d = inverse(e, (p-1)*(q-1))

    return ((N,e), d)

def encrypt(pubkey, m):
    N,e = pubkey
    return pow(bytes_to_long(m.encode('utf-8')), e, N)

def main(flag):
    pubkey, _privkey = gen_key(1024)
    encrypted = encrypt(pubkey, flag) 
    return (pubkey[0], encrypted)

if __name__ == "__main__":
    flag = open('flag.txt', 'r').read()
    flag = flag.strip()
    N, cypher  = main(flag)
    print("N:", N)
    print("e:", e)
    print("cyphertext:", cypher)
    exit()
```


Executed `nc verbal-sleep.picoctf.net 60565` in terminal to connect to the challenge server

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 60565                                                                                                 

N:22112698953749063109617592166620913818275148140617108348989655249575043047089637270372618525184162089013302540284103055407597480046872073239547197567906818          

e:65537                                           

cyphertext:2013148744206391195361764593325001979080921842718197486889962819056091262679808492399271864230029041559023832822027589585282050922887174003409831450203737 
```
And I got the `N` , `e` and `c`(cipher text)

After observing the `Source Code` , I found that the `N` value is about `1024` bits (**Weakness 1** / vulnerability 1 of this implementation) , and that can be easily factorize in 2025 (at present).So I went for the online tool [Alpertron](https://www.alpertron.com.ar/ECM.HTM) and factorized it . That gave me four divisors :
    - 1 
    - 2
    - 11056349476874531554808796083310456909137574070308554174494827624787521523544818635186309262592081044506651270142051527703798740023436036619773598783953409
    - N itself.

So after seeing those results the `p` & `q`( these are the prime numbers , use to get the value of `N` using the formula `N = p * q` ) values might be the 2nd and the 3rd one among the four divisors. But initially I was a bit confused about p value because in the source code it is mentioned that `p,q` are about 512 bits each. But I wanted to check whether  multiplying `p` and `q` gets our `N` or not  , and I got it exactly. Finally we have values of p , q , c , e , N .

So , to decrypt the `c` cipher text , we have a formula :
```math
m = c^d mod n  where m - original message 
```
Here we need `d` and  to get that we use 
```math
d * e ≡ 1 mod φ(n)
```
So we need to find the inverse of `e mod φ(n)` and that's very easy in python . But wait!  we dont have `φ(n)` . So how do we find it?

We can find `φ(n)` using the formula :
```math
φ(n) = (p-1) * (q-1)
```
⚠️ While analyzing the challenge, I noticed that `N` is even (**Weakness 2**) — which immediately got my attention . In RSA, the product `N = p * q` should be odd if both `p` and `q` are large random primes. The only possible way for `N` to be even is if one of the primes is `2`. Since 2 is the only even prime, I assumed `p = 2` and computed `q = N // 2`, effectively factoring `N` instantly and exposing the private key

**Here is the script that decrypts the message**
```python
from Crypto.Util.number import inverse
from Crypto.Util.number import long_to_bytes
n = 22112698953749063109617592166620913818275148140617108348989655249575043047089637270372618525184162089013302540284103055407597480046872073239547197567906818 
e = 65537 
q = 0 
p = 2 
c = 2013148744206391195361764593325001979080921842718197486889962819056091262679808492399271864230029041559023832822027589585282050922887174003409831450203737 
if (n % 2 == 0):
  q = n // 2
  # print (q)

phi_n = (p-1)*(q-1)
d = inverse(e, phi_n)
# print(d)
m = pow(c , d , n)
message = long_to_bytes(m).decode('utf-8')
print(message)
```
And this script got me the flag.
**FLAG :** `picoCTF{tw0_1$_pr!m3f81fef0a}`

---

# HashCrack

**Challenge:**

![Hash-Crack](/images/hash-crack.PNG)

Author: Nana Ama Atombo-Sackey

Description: A company stored a secret message on a server which got breached due to the admin using weakly hashed passwords. Can you gain access to the secret stored within the server?
Access the server using `nc verbal-sleep.picoctf.net 62644`

Challenge Link : [https://play.picoctf.org/practice/challenge/475?category=2&page=1](https://play.picoctf.org/practice/challenge/475?category=2&page=1)

**Given:** `nc verbal-sleep.picoctf.net 62644`

**Explanation:**
The command `nc verbal-sleep.picoctf.net 62644` instructs the Netcat utility (`nc`) to establish a network connection to the server located at the hostname `verbal-sleep.picoctf.net` on port number `62644`.

**How I approached**

When I ran that command in the Ubuntu terminal, I was prompted with:

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 62644
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash:
```

Firstly, I tried `"password"` as the password, but it said:

```terminal
Incorrect. Goodbye.
```

Then, I observed the given hash `"482c811da5d5b4bc6d497ffa98491e38"` and recognized that it is an MD5 hash (Message Digest Algorithm).

**A brief about the MD5 hash algorithm:**

MD5 is a widely used cryptographic hash function that takes an input of arbitrary length and produces a fixed-length output of 128 bits (16 bytes). This output is often represented as a 32-character hexadecimal number.

Think of it like a digital fingerprint for data. Any change to the original data, no matter how small, will (with a very high probability) result in a completely different MD5 hash. This property is known as the avalanche effect.

MD5 was designed by Ronald Rivest in 1991 as an improvement over an earlier hash function, MD4. It was standardized in RFC 1321 in 1992.

After decrypting that hash using an online tool, I got the password `"password123"`.

So, I proceeded by entering it:

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 62644
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!

Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
Enter the password for the identified hash:
```

Again, I was prompted with a hash (`b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3`) and asked for the password.

This hash is a SHA-1 hash, so I decrypted it using online decrypt tools (e.g., https://md5hashing.net/hash/sha1/b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3).

**A brief about the SHA-1 hash algorithm:**

SHA-1 (Secure Hash Algorithm 1) is a cryptographic hash function designed by the U.S. National Security Agency (NSA) and published in 1995 as a U.S. Federal Information Processing Standard (FIPS).

Here's a brief overview:

- **Input:** Takes an input message of arbitrary length (up to $2^{64} - 1$ bits).
- **Output:** Produces a fixed-size 160-bit (20-byte) hash value, often represented as a 40-character hexadecimal number, known as a message digest.
- **One-way function:** It's computationally infeasible to reverse the process and obtain the original message from its hash.
- **Deterministic:** The same input message will always produce the same SHA-1 hash.
- **Avalanche effect:** Even a small change in the input message will result in a significantly different hash value.

After decrypting that, I got the password `"letmein"`. I proceeded by entering it:

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 62644
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!

Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
Enter the password for the identified hash: letmein
Correct! You've cracked the SHA-1 hash with no secret found!

Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
Enter the password for the identified hash:
```

Then, I was again given a hash (`916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745`) and asked for the password.

This hash is a SHA-256 hash, so I decrypted that and got `"qwerty098"` as the password.

**SHA-256 algorithm brief:**

SHA-256 (Secure Hash Algorithm 256-bit) is another member of the SHA-2 family of cryptographic hash functions, designed by the National Security Agency (NSA) and published by the National Institute of Standards and Technology (NIST) in 2001. It's a significant step up in security compared to SHA-1.

Here's a brief overview:

- **Input:** Takes an input message of arbitrary length (up to $2^{64} - 1$ bits).
- **Output:** Produces a fixed-size 256-bit (32-byte) hash value, typically represented as a 64-character hexadecimal number, known as a message digest.
- **One-way function:** Like MD5 and SHA-1, it's computationally infeasible to reverse the process and obtain the original message from its hash.
- **Deterministic:** The same input will always yield the same SHA-256 hash.
- **Strong Avalanche Effect:** Even a tiny alteration in the input results in a drastically different hash.

So, I entered it:

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 62644
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!

Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
Enter the password for the identified hash: letmein
Correct! You've cracked the SHA-1 hash with no secret found!

Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
Enter the password for the identified hash: qwerty098
Correct! You've cracked the SHA-256 hash with a secret found.
The flag is: picoCTF{UseStr0nG_h@shEs_&PaSswDs!_3eb19d03}
```

Hurray! Finally got the flag!

**Flag:** `picoCTF{UseStr0nG_h@shEs_&PaSswDs!_3eb19d03}`

**Final solution:**

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 62644
Welcome!! Looking For the Secret?

We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!

Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
Enter the password for the identified hash: letmein
Correct! You've cracked the SHA-1 hash with no secret found!

Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
Enter the password for the identified hash: qwerty098
Correct! You've cracked the SHA-256 hash with a secret found.
The flag is: picoCTF{UseStr0nG_h@shEs_&PaSswDs!_3eb19d03}
```

**What I learnt?**

1.  **MD5 algorithm:** Understanding its basic properties as a hashing algorithm and its vulnerability to cracking.
2.  **SHA-1 and SHA-256:** Learning about these more secure hashing algorithms and their increased complexity compared to MD5.
3.  **`nc verbal-sleep.picoctf.net 62644`:** Understanding that this command uses the Netcat utility (`nc`) to establish a direct network connection to a specified host (`verbal-sleep.picoctf.net`) and port (`62644`), allowing for interactive communication with the server. It's used here to interact with the challenge server, receive the hashes, and send the cracked passwords.

---

# ROT 13

**CHALLENGE :** 
[https://play.picoctf.org/practice/challenge/62?category=2&page=1](https://play.picoctf.org/practice/challenge/62?category=2&page=1)

![ROT13](/images/ROT13.PNG)

As the challenge name suggests, the flag is encrypted using **ROT13** method .
So, we can directly write a script to decrypt the flag.  

```python
enc_flag ="cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}"
flag = ""

for i in enc_flag:
  if('a' <= i <= 'z'):
    shifted = ((ord(i) - ord('a'))+13 ) % 26 +ord('a')
    flag+=chr(shifted)
  elif('A' <= i <= 'Z'):
    shifted = ((ord(i) - ord('A'))+13 ) % 26 +ord('A')
    flag+=chr(shifted)
  else:
    flag+=i

print(flag)
```
And I got the flag 

**FLAG :** `picoCTF{not_too_bad_of_a_problem}`

---

## Medium chall's 


# Basic Mod 1

**Challenge**

![basic-mod1](/images/basic-mod-1.PNG)

**Given message :** `128 322 353 235 336 73 198 332 202 285 57 87 262 221 218 405 335 101 256 227 112 140`

Solution 

For the description given in the challenge , wrote a python script that decrypts the Message 

```python 
numbers = [128, 322, 353, 235, 336, 73, 198, 332, 202, 285, 57, 87, 262, 221, 218, 405, 335, 101, 256, 227, 112, 140]

def decrypt_message(numbers):
    decrypted_chars = []
    for num in numbers:
        mod_value = num % 37
        if 0 <= mod_value <= 25:
            decrypted_chars.append(chr(ord('A') + mod_value))
        elif 26 <= mod_value <= 35:
            decrypted_chars.append(str(mod_value - 26))
        elif mod_value == 36:
            decrypted_chars.append('_')
    return "".join(decrypted_chars)

decrypted_message = decrypt_message(numbers)
flag = f"picoCTF{{{decrypted_message}}}"
print(flag)
```

By running this code I got the flag `picoCTF{R0UND_N_R0UND_79C18FB3}` .

---

# Guess My Cheese Part1

**challenge:**

Author: aditin

Description: Try to decrypt the secret cheese password to prove you're not the imposter!
Connect to the program on our server: `nc verbal-sleep.picoctf.net 53407`

challenge link : [https://play.picoctf.org/practice/challenge/473?category=2&page=1](https://play.picoctf.org/practice/challenge/473?category=2&page=1)

Hint: Remember that cipher we devised together Squeexy? The one that incorporates your affinity for linear equations???

**How i went through the challege**

The challenge presented an intriguing scenario: determine the real Squeexy by guessing an encrypted "cheese." Upon connecting to the server using netcat verbal-sleep.picoctf.net 53407, I was greeted with a story about an evil clone and a secret encrypted cheese: "HIJJMOBE." The prompt offered two options: encrypt a cheese ('e') or guess the cheese ('g').

Initially confused about what kind of "cheese" was being referred to, I took a cue from the hint mentioning "top secret and limited edition" cheeses. A quick search for "top cheeses" revealed a list including cheddar, mozzarella, Parmesan, Swiss, and feta.

My breakthrough came when I tried encrypting one of these common cheeses. Choosing 'e' for encrypt and then inputting "feta" yielded an encrypted form: "RKLI." This confirmed that the challenge involved a substitution cipher based on the hint about linear equations 

```terminal 
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 53407

*******************************************
***             Part 1                  ***
***    The Mystery of the CLONED RAT    ***
*******************************************

The super evil Dr. Lacktoes Inn Tolerant told me he kidnapped my best friend, Squeexy, and replaced him with an evil clone! You look JUST LIKE SQUEEXY, but I'm not sure if you're him or THE CLONE. I've devised a plan to find out if YOU'RE the REAL SQUEEXY! If you're Squeexy, I'll give you the key to the cloning room so you can maul the imposter...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  HIJJMOBE
Hint: The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!
Commands: (g)uess my cheese or (e)ncrypt a cheese
What would you like to do?
```
The hint about "linear equations" pointed towards an affine cipher, which uses the form:

Encryption: `y=(mx+b)mod26`

Decryption: `x=((y−b)⋅m−1)mod26`

where:

x represents the numerical value of a letter in the plaintext (A=0, B=1, ..., Z=25).

y represents the numerical value of the corresponding letter in the ciphertext.

m and b are the keys of the cipher. m−1 is the modular multiplicative inverse of m modulo 26.
To find the keys m (which you denoted as 'a') and b, I used the plaintext-ciphertext pairs from "feta" and "RKLI":

f (5) → R (17)
e (4) → K (10)
t (19) → L (11)
a (0) → I (8)
Substituting the first pair into the encryption equation:
17=(5m+b)mod26

Substituting the second pair:
10=(4m+b)mod26

Subtracting the second equation from the first:
17−10=(5m+b)−(4m+b)mod26

7=m mod26

So, m=7.

Now, substituting the value of m back into the second equation:
10=(4⋅7+b)mod26  => 10 =(28+b)mod26  => 10 =(2+b)mod26

b=10−2mod26

b=8mod26

Thus, the encryption key is m=7 and b=8.

To decrypt the secret cheese "HIJJMOBE," I needed the decryption equation. First, I needed to find the modular multiplicative inverse of m=7 modulo 26. This is a number m 
−1
  such that (m⋅m 
−1
 )mod26=1. By testing values or using the Extended Euclidean Algorithm, I found that 7 
−1
 mod26=15 (since 7⋅15=105=4⋅26+1).

Now, the decryption equation becomes:
x=((y−8)⋅15)mod26

Applying this to "HIJJMOBE":

H (7) → ((7−8)⋅15)mod26=(−1⋅15)mod26=−15mod26=11 (L)

I (8) → ((8−8)⋅15)mod26=(0⋅15)mod26=0 (A)

J (9) → ((9−8)⋅15)mod26=(1⋅15)mod26=15 (P)

J (9) → ((9−8)⋅15)mod26=(1⋅15)mod26=15 (P)

M (12) → ((12−8)⋅15)mod26=(4⋅15)mod26=60mod26=8 (I)

O (14) → ((14−8)⋅15)mod26=(6⋅15)mod26=90mod26=12 (M)

B (1) → ((1−8)⋅15)mod26=(−7⋅15)mod26=−105mod26=23 (Z)

E (4) → ((4−8)⋅15)mod26=(−4⋅15)mod26=−60mod26=18 (S)

The decrypted cheese is `LAPPIMZS`. 

Finally, I selected 'g' to guess the cheese and entered "LAPPIMZS" And this got me the flag .

**Final Solution :**

```terminal
sunil-kumar@DESKTOP-GBKN3LB:~$ nc verbal-sleep.picoctf.net 53407

*******************************************
***             Part 1                  ***
***    The Mystery of the CLONED RAT    ***
*******************************************

The super evil Dr. Lacktoes Inn Tolerant told me he kidnapped my best friend, Squeexy, and replaced him with an evil clone! You look JUST LIKE SQUEEXY, but I'm not sure if you're him or THE CLONE. I've devised a plan to find out if YOU'RE the REAL SQUEEXY! If you're Squeexy, I'll give you the key to the cloning room so you can maul the imposter...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  HIJJMOBE
Hint: The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!
Commands: (g)uess my cheese or (e)ncrypt a cheese
What would you like to do?
e

What cheese would you like to encrypt? feta
Here's your encrypted cheese:  RKLI
Not sure why you want it though...*squeak* - oh well!

I don't wanna talk to you too much if you're some suspicious character and not my BFF Squeexy!
You have 2 more chances to prove yourself to me!

Commands: (g)uess my cheese or (e)ncrypt a cheese
What would you like to do?
e

What cheese would you like to encrypt? feta
Here's your encrypted cheese:  RKLI
Not sure why you want it though...*squeak* - oh well!

I don't wanna talk to you too much if you're some suspicious character and not my BFF Squeexy!
You have 1 more chances to prove yourself to me!

Commands: (g)uess my cheese or (e)ncrypt a cheese
What would you like to do?
g


   _   _
  (q\_/p)
   /. .\.-.....-.     ___,
  =\_t_/=     /  `\  (
    )\ ))__ __\   |___)
   (/-(/`  `nn---'

SQUEAK SQUEAK SQUEAK

         _   _
        (q\_/p)
         /. .\
  ,__   =\_t_/=
     )   /   \
    (   ((   ))
     \  /\) (/\
      `-\  Y  /
         nn^nn


Is that you, Squeexy? Are you ready to GUESS...MY...CHEEEEEEESE?
Remember, this is my encrypted cheese:  HIJJMOBE
So...what's my cheese?
LAPPIMZS

         _   _
        (q\_/p)
         /. .\         __
  ,__   =\_t_/=      .'o O'-.
     )   /   \      / O o_.-`|
    (   ((   ))    /O_.-'  O |
     \  /\) (/\    | o   o  o|
      `-\  Y  /    |o   o O.-`
         nn^nn     | O _.-'
                   '--`

munch...

         _   _
        (q\_/p)
         /. .\         __
  ,__   =\_t_/=      .'o O'-.
     )   /   \      / O o_.-`|
    (   ((   ))      ).-'  O |
     \  /\) (/\      )   o  o|
      `-\  Y  /    |o   o O.-`
         nn^nn     | O _.-'
                   '--`

munch...

         _   _
        (q\_/p)
         /. .\         __
  ,__   =\_t_/=      .'o O'-.
     )   /   \      / O o_.-`|
    (   ((   ))        )'  O |
     \  /\) (/\          )  o|
      `-\  Y  /         ) O.-`
         nn^nn        ) _.-'
                   '--`

MUNCH.............

YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{ChEeSy696d4adc}
```
**Flag:** ` picoCTF{ChEeSy696d4adc}`

**What I Learnt:**

1. **Understanding Modular Arithmetic (How Mod Works):** I gained a deeper understanding of how the modulo operation works, especially in the context of cryptography, where it ensures results stay within a specific range (in this case, 0-25 for the alphabet).

2. **Affine Cipher Encryption and Decryption:** This challenge provided hands-on experience with the affine cipher, a type of monoalphabetic substitution cipher. I learned how to:
Use the encryption equation y=(mx+b)mod26.
Determine the keys (m and b) by analyzing plaintext-ciphertext pairs.
Calculate the modular multiplicative inverse of m to use in the decryption equation x=((y−b)⋅m−1)mod26.


This challenge was a fantastic introduction to basic cryptography and the power of linear equations in creating ciphers!

---

# Guess My Cheese Part2

**Challenge**

The imposter was able to fool us last time, so we've strengthened our defenses!
Here's our list(You can download the list from the site) of cheeses.
Connect to the program on our server: nc verbal-sleep.picoctf.net 58138

**Hints** 
- I heard that SHA-256 is the best hash function out there!
- Remember Squeexy, we enjoy our cheese with exactly 2 nibbles of hexadecimal-character salt!
- Ever heard of rainbow tables?

**Wait what is Salt in the second Hint ?**
When talking about security, "salt" is just random data added to a password before it's scrambled (hashed).

Here's why that's important:

Imagine you have a secret password, say "mysecret". A website doesn't store "mysecret" directly. Instead, it turns it into a garbled code, like "abc123xyz". This is called hashing.

Now, if a hacker steals that garbled code "abc123xyz", they could try to look it up in a giant dictionary of pre-computed codes (called a rainbow table) to find your original password. Also, if someone else used "mysecret" as their password, their garbled code would be "abc123xyz" too, making it easy for hackers to spot identical passwords.

This is where salt comes in.

When you add a unique, random "salt" to your password before it's hashed, even if two people have the same password, their final garbled codes will be completely different.

For example:

Your password: "mysecret" + Unique Salt "1a2b" = Scrambled Code "XCV456"
Someone else's password: "mysecret" + Different Unique Salt "3c4d" = Scrambled Code "QWE789"
Now, "XCV456" and "QWE789" are totally different, even though the original password was the same. This makes those rainbow tables useless and slows down attackers significantly because they have to try to guess each password and salt combination individually.

In short, salt adds an extra layer of randomness to make your hashed password much harder to crack.

**Solution**
![guess my cheese](/images/guessmycheese2img1.PNG)

As you can see it provides us some SHA-256 Hash and it’s clear that it’s salted.

Now what is SHA-256 first?

SHA-256 is a type of cryptographic hash function, meaning it takes any input (like text or data) and turns it into a fixed-size string of characters, typically 64 characters long when written in hexadecimal. It’s part of the SHA-2 family, which is a set of algorithms created by the NSA to provide secure ways of verifying data.

Now, let’s dive back into the challenge. If you take a closer look at the Cheese List, you’ll notice that the cheese names aren’t consistent. Some names mix uppercase and lowercase letters, and even include symbols like parentheses. This inconsistency means that different text encodings can represent these names as bytes in different ways. Some common encodings to consider are UTF-8, UTF-16 (both little-endian and big-endian), and Latin-1.

The goal here is to pair each cheese name with salt values, but with so many cheese names in the list, how do we efficiently handle this? For every cheese name (and its variations), we combine it with salt values that range from 0 to 255. The salt can be appended to the end, prepended to the beginning, or inserted at various points within the cheese name itself. This creates a wide range of potential combinations. The next step is to hash each combination. For every possible combination of cheese name, salt value, case variation, and encoding, we will calculate the SHA-256 hash.

Next, as the hash-cracking tools run, we will compare the computed hash with the target hash. If a match is found, we’ve successfully identified the correct combination.

I found the python code that automates the entire process : 

```python
import hashlib
import sys
import time

target_sha256_hash = "1a36f0299a51eaae9c3d773a0e35e0e50c19d4754ca376e065808691430b6438" # REMEMBER TO REPLACE "GIVEN_HASH" WITH YOUR ACTUAL TARGET HASH!

# Supported encodings for testing
encoding_formats = ["utf-8", "utf-16-le", "utf-16-be", "latin-1"]

# Case transformations to apply
def case_original(text):
    return text

def case_lower(text):
    return text.lower()

def case_upper(text):
    return text.upper()

case_transformations = {
    "original": case_original,
    "lower": case_lower,
    "upper": case_upper,
}

# We will load the cheese from the cheese list
with open("cheeseList.txt", "r") as file: # Corrected filename for consistency
    cheese_names = [line.strip() for line in file if line.strip()]

match_found = False

def check_hash(candidate_bytes, method, extra_info, cheese, case_type, encoding, salt):
    """
    Compute SHA-256 hash for a given byte sequence and compare it to the target hash.
    If a match is found, display relevant details and return True.
    """
    global match_found
    computed_hash = hashlib.sha256(candidate_bytes).hexdigest()

    if computed_hash == target_sha256_hash:
        print("\n[!!] VALID MATCH FOUND!")
        print("=" * 40)
        print(f"[+] Cheese Name  : {cheese}") # Adjusted spacing for alignment
        print(f"[+] Case Variant : {case_type}") # Adjusted spacing
        print(f"[+] Encoding     : {encoding}") # Adjusted spacing
        print(f"[+] Salt Value   : (0x{salt:02x})") # Adjusted spacing
        print(f"[+] Method Used  : {method}") # Adjusted spacing
        print(f"[+] Extra Info   : {extra_info}") # Adjusted spacing
        print(f"[+] SHA-256 Hash : {computed_hash}") # Adjusted spacing
        try:
            decoded_candidate = candidate_bytes.decode(encoding)
        except Exception:
            decoded_candidate = repr(candidate_bytes)
        print(f"[+] Candidate String ({encoding}): {decoded_candidate}")
        print("=" * 40)
        match_found = True
        return True
    return False

# Start brute-force testing
start_time = time.time()
print("[*] Starting cheese cracking operation....")

for cheese in cheese_names:
    for case_type, transform_func in case_transformations.items():
        modified_cheese = transform_func(cheese)

        for encoding in encoding_formats:
            try:
                cheese_bytes = modified_cheese.encode(encoding)
            except Exception:
                continue

            for salt_value in range(256):
                salt_byte = bytes([salt_value])
                salt_hex_str = format(salt_value, "02x")

                try:
                    salt_hex_bytes = salt_hex_str.encode(encoding)
                except Exception:
                    salt_hex_bytes = salt_hex_str.encode("utf-8") # Fallback encoding

                # Test different variations of salted hashes
                tests = [
                    (cheese_bytes + salt_byte, "append_raw", "raw byte appended"),
                    (salt_byte + cheese_bytes, "prepend_raw", "raw byte prepended"),
                    (cheese_bytes + salt_hex_bytes, "append_hex", "hex string appended"),
                    (salt_hex_bytes + cheese_bytes, "prepend_hex", "hex string prepended"),
                ]

                for candidate, method, extra_info in tests:
                    if check_hash(candidate, method, extra_info, cheese, case_type, encoding, salt_value):
                        break # Breaks out of the 'tests' loop
                if match_found:
                    break # Breaks out of the 'salt_value' loop

                # Insert salt byte at every possible index
                for i in range(len(cheese_bytes) + 1):
                    candidate = cheese_bytes[:i] + salt_byte + cheese_bytes[i:]
                    if check_hash(candidate, "insert_raw", f"at index {i}", cheese, case_type, encoding, salt_value):
                        break # Breaks out of the 'insert_raw' loop
                if match_found:
                    break # Breaks out of the 'salt_value' loop

                # Insert hex-encoded salt at every index
                for i in range(len(cheese_bytes) + 1):
                    candidate = cheese_bytes[:i] + salt_hex_bytes + cheese_bytes[i:]
                    if check_hash(candidate, "insert_hex", f"at index {i}", cheese, case_type, encoding, salt_value):
                        break # Breaks out of the 'insert_hex' loop
                if match_found:
                    break # Breaks out of the 'salt_value' loop
            if match_found: # This 'if' is outside the salt_value loop, breaking the encoding loop
                break
        if match_found: # This 'if' is outside the encoding loop, breaking the case_type loop
            break
    if match_found: # This 'if' is outside the case_type loop, breaking the cheese loop
        break

end_time = time.time()

if not match_found:
    print("[!] No matching cheese and salt combination was found.")
else:
    print(f"\n[+] Execution completed in {end_time - start_time:.2f} seconds.")

```
After execution, this is the output
```terminal 
[*] Starting cheese cracking operation....

[!!] VALID MATCH FOUND!
========================================
[+] Cheese Name  : Curworthy
[+] Case Variant : lower
[+] Encoding     : utf-8
[+] Salt Value   : (0xa8)
[+] Method Used  : append_raw
[+] Extra Info   : raw byte appended
[+] SHA-256 Hash : 1a36f0299a51eaae9c3d773a0e35e0e50c19d4754ca376e065808691430b6438
[+] Candidate String (utf-8): b'curworthy\xa8'
========================================

[+] Execution completed in 16.91 seconds.

```

We’ve finally obtained the cheese name and its corresponding salt in hexadecimal format, now let’s verify it!

![guess my cheese](/images/guessmycheesepart2img2.jpg)

Yeah! I got the Flag `picoCTF{cHeEsYaf31a5c0}`

---

# HideToSee

Challenge Link: [https://play.picoctf.org/practice/challenge/351?category=2&page=2](https://play.picoctf.org/practice/challenge/351?category=2&page=2)

**Description**
The challenge involved extracting hidden information from an image using steganography. The extracted data was an encrypted text file, which, when decoded, revealed the flag.

**Solution**

**Steganography**
To extract the hidden data, I used **Steghide** to retrieve the data from the image. The following command was used:

```bash
steghide info atbash.jpg
```
atbash.jpg is the image file provided with the challenge. The command extracts the hidden data from the image.

**Decryption**
The extracted file was encrypted. The image description gave a hint toward the Atbash Cipher. I used an Atbash cipher decoder to decrypt the contents of the file.

**Flag**
The decrypted text revealed the flag:

```bash
picoCTF{atbash_crack_7142fwd9}
```
**Summary**
In this challenge, I utilized Steghide to extract hidden data from an image, identified the Atbash Cipher for decryption, and successfully obtained the flag.

---

# Morse Code 

**Challenge** 

![morse code](/images/morsecode.PNG)

Here in this challenge I was given by a Audio file , you can have it from the website .

**What is Morse code ?**

`Morse code` is a communication system that encodes text characters (letters, numbers, and punctuation) into standardized sequences of two different signal durations: `dots` (short signals, also called `dits`) and `dashes` (long signals, also called "`dahs`"). It was originally developed in the 1830s by Samuel Morse and Alfred Vail for use with the electrical telegraph.

The key to Morse code lies in its rhythmic structure and precise timing:

Dot (dit): A short signal.
Dash (dah): A long signal, ideally three times the duration of a dot.
Space between dits and dahs within a character: Equal to one dot duration.
Space between characters (letters/numbers) within a word: Equal to three dot durations.
Space between words: Equal to seven dot durations.
This system allows for communication through various mediums by simply turning a signal "on" or "off" for varying durations.

**Chart of the Morse code , 26 letters and 10 numerals**

![morseCodeChart](/images/morsecodechart.PNG)


**Solution** 

Since it is an Audio File , I went for a online tool 

**TOOl LINK** : [https://morsefm.com/](https://morsefm.com/)

There I uploaded the file , and got the text `WH47 H47H 90D W20U9H7` 

As the description of the Challenge says that ` put underscores in place of pauses, and use all lowercase ` , The Flag is 

`picoCTF{wh47_h47h_90d_w20u9h7}`

---

# Rail - Fence 

**challenge** 

![rail-fence](/images/railfence.PNG)

**Given Message** : `Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6`

As it is in the description of  the challenge , the message is scrambled using the technique or method called `Rail-fence` also called zigzag cipher . 

**What is the Rail-Fence cipher ?**

The `Rail Fence Cipher` is a classic example of a `transposition cipher`. This means that instead of substituting letters for other letters (like in a Caesar cipher), it rearranges the original letters of the plaintext to create the `ciphertext`. It gets its name from the way the text is written, resembling a `zig-zag pattern` along the `rails` of a fence.

The `key` for the Rail Fence cipher is the `number of rails used for encryption`.

**How it works**

**Encryption**

In the `rail fence cipher`, the `plaintext` is written downwards diagonally on successive `rails` of an imaginary fence, then moving up when the bottom rail is reached, down again when the top rail is reached, and so on until the whole plaintext is written out. The `ciphertext` is then read off in rows

For example, to encrypt the message 'HELLO WORLD' with 3 "rails", write the text as:

```text
H . . . O . . . L . . .
. E . L . W . R . D .
. . L . . . O . . .
```

**Decryption** 

To Decrypt the Rail Fence Cipher :
- Arrange rails as same as while encrypting ( no. of rows = rails/key  ) .

- Number Columns should be equal to the length of the `Cipher text` in each row / rail .

- Place some place holder where you can fill the cipher letter in the rails . (like encrypting , placeholders should place diagonally) .

- Now replace the place holder with the cipher letter  and it should be done as row after row (means replace all the place holders in the first row then move to the  next)

- Read the text diagonally ,` The PlainText `.

**Solution** 

```text 

Given cipher text  : Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6

Number of rails : 4 



* _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _
_ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ *
_ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _ * _ * _ _ _
_ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _


T _ _ _ _ _ a _ _ _ _ _ $ _ _ _ _ _ * _ _ _ _ _ 7 _ _ _ _ _ N _ _ _ _ _ 6 _ _ _ _ _ D _ _ _ _ _ 8 _ _ _ _ _ D _
_ h _ _ _ l _ g _ _ _ : _ W _ _ _ 3 _ D _ _ _ * _ H _ _ _ 3 _ C _ _ _ 3 _ 1 _ _ _ N _ * _ _ _ * _ 3 _ _ _ 8 _ 7
_ _ e _ f _ _ _ $ _ s _ _ _ H _ R _ _ _ 0 _ 5 _ _ _ 3 _ F _ _ _ 3 _ 8 _ _ _ N _ 4 _ _ _ 3 _ D _ _ _ F _ D _ _ _
_ _ _ $ _ _ _ _ _ i _ _ _ _ _ 3 _ _ _ _ _ 3 _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ * _ _ _ _ _ N _ _ _ _ _ 6 _ _ _ _


So , the TEXT is :


The flag is: WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7

NOTE : $ - space 
       * - hiphen

```

So the **FLAG** is ,  is `picoCTF{WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7}`

---

# Read My Cert

**Challenge** 

![challengeimage](/images/readmycertt.PNG)

**Hint** 
- Download the certificate signing request and try to read it.

**Here's a breakdown of a potential thought process and solution approach:**

1. Initial Reconnaissance: What Am I Looking At?

    The first thing you'd notice are the distinct `-----BEGIN CERTIFICATE REQUEST-----` and `-----END CERTIFICATE REQUEST-----` lines. This immediately tells you that you're dealing with a `PEM (Privacy-Enhanced Mail)` encoded Certificate Signing Request.
    - Thought process here: "Okay, this isn't just random gibberish. It's a structured piece of data. My goal is to decode it and see what's inside."

2. The Right Tool for the Job: `OpenSSL`

    For anything related to certificates, keys, and CSRs, `OpenSSL` is your best friend. It's the standard toolkit for these operations.

    - My thought process here: "How do I decode a CSR? I remember openssl being the go-to tool for certificate stuff. I need to find the specific command to read a CSR."

    - Action: A quick search (e.g., "openssl read csr," "decode csr openssl") would quickly lead you to the command: `openssl req -in [your_csr_file].csr -noout -text`.

3.  Execution and Discovery

    You'd save the provided CSR text into a file (let's say challenge.csr) and then run the openssl command.

![readmycertimg2](/images/readmycertimg2.PNG)

1. The Flag 

    Running that commang got me the flag `picoCTF{read_mycert_41d1c74c}`

**Flag**

- `picoCTF{read_mycert_41d1c74c}`

**What i learnt**

- The CSR and how it works 

    1. A Certificate Signing Request (CSR) is an encoded text block that you send to a Certificate Authority (CA)   to get a digital certificate. It contains your public key and identifying information (like your domain name). It's essentially the application for your website's digital ID.

    2. You create a CSR (with your public key) on your server and send it to a Certificate Authority (CA).

        The CA verifies your identity using the CSR and then issues a signed digital certificate that you install back on your server.

---

# Vigenere

**Challenge :** 

**Description** 

Can you decrypt this message?
Decrypt this message using this key "CYLAB".

**The given message** : `rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_2951c89f}`

**Hint** 

- https://en.wikipedia.org/wiki/Vigen%C3%A8re_cipher

**Solution**

Hint and the challenge name itself says that to encrypt that message they used `VIGENERE CIPHER` technique . So what it is ? 

**Vigenere Cipher**

The `Vigenère cipher` is a method of encrypting alphabetic text by using a series of different Caesar ciphers based on the letters of a keyword. It's a polyalphabetic substitution cipher, meaning that each letter of the plaintext can be encrypted to a different ciphertext letter, depending on its position and the key. This makes it significantly more secure than simple substitution ciphers (like the Caesar cipher), which always substitute a given plaintext letter with the same ciphertext letter.

For nearly three centuries, the Vigenère cipher was considered unbreakable and was even known as "le chiffre indéchiffrable" (the indecipherable cipher). However, it was eventually broken using methods like Kasiski examination and frequency analysis.

**How it Works: The Vigenère Square (Tabula Recta)**

The Vigenère cipher typically uses a Vigenère Square (also known as a tabula recta). This is a 26x26 grid of alphabets.

- The first row is the standard alphabet (A to Z).
- Each subsequent row is a Caesar cipher shift of the row above it, moving one position to the left. So, the second row starts with B, the third with C, and so on.

**Here's the  representation of the Vigenère Square:**

![VignereCipherTable](https://pages.mtu.edu/~shene/NSF-4/Tutorial/VIG/FIG-VIG-Table.jpg)

**Encryption Process**

To encrypt a plaintext message using the Vigenère cipher, you need:

- Plaintext: The message you want to encrypt.
- Keyword: A secret word or phrase.

The steps are as follows:

- Prepare the Keyword: Repeat the keyword until its length matches the length of the plaintext. For example, if your plaintext is "ATTACKATDAWN" and your keyword is "LEMON", you'd extend the key to "LEMONLEMONLE".
- Align Plaintext and Key: Write the plaintext and the extended key aligned character by character.
- Encrypt Each Letter: For each letter in the plaintext:
    - Find the plaintext letter in the top row of the Vigenère Square.
    - Find the corresponding key letter in the leftmost column (or vice versa, but consistency is key).
    - The ciphertext letter is found at the intersection of the row (of the key letter) and the column (of the      plaintext letter).

**Encryption Example:**

Let's encrypt `Plaintext: ATTACKATDAWN` with `Keyword: LEMON`

1. Plaintext: A T T A C K A T D A W N
2. Extended Key: L E M O N L E M O N L E

Now, encrypt each pair using the Vigenère Square:

A (Plaintext) + L (Key): Find 'A' in the top row, 'L' in the left column. Their intersection is L.

T (Plaintext) + E (Key): Find 'T' in the top row, 'E' in the left column. Their intersection is X.

T (Plaintext) + M (Key): Find 'T' in the top row, 'M' in the left column. Their intersection is F.

A (Plaintext) + O (Key): Find 'A' in the top row, 'O' in the left column. Their intersection is O.

C (Plaintext) + N (Key): Find 'C' in the top row, 'N' in the left column. Their intersection is P.

K (Plaintext) + L (Key): Find 'K' in the top row, 'L' in the left column. Their intersection is V.

A (Plaintext) + E (Key): Find 'A' in the top row, 'E' in the left column. Their intersection is E.

T (Plaintext) + M (Key): Find 'T' in the top row, 'M' in the left column. Their intersection is F.

D (Plaintext) + O (Key): Find 'D' in the top row, 'O' in the left column. Their intersection is R.

A (Plaintext) + N (Key): Find 'A' in the top row, 'N' in the left column. Their intersection is N.

W (Plaintext) + L (Key): Find 'W' in the top row, 'L' in the left column. Their intersection is H.

N (Plaintext) + E (Key): Find 'N' in the top row, 'E' in the left column. Their intersection is R.

**Ciphertext:** `LXFOPVEFRNHR`

**Decryption Process**

To decrypt a ciphertext message, you need:

- Ciphertext: The encrypted message.
- Keyword: The same secret word or phrase used for encryption.

The steps are as follows:

- Prepare the Keyword: Repeat the keyword until its length matches the length of the ciphertext.
- Align Ciphertext and Key: Write the ciphertext and the extended key aligned character by character.
- Decrypt Each Letter: For each letter in the ciphertext:
    - Find the key letter in the leftmost column of the Vigenère Square. This determines the row you will use.
    - Within that key letter's row, find the ciphertext letter.
    - The plaintext letter is found at the top of the column where the ciphertext letter was located.

**Decryption Example:**

Let's decrypt `Ciphertext: LXFOPVEFRNHR` with `Keyword: LEMON`

- Ciphertext: L X F O P V E F R N H R
- Extended Key: L E M O N L E M O N L E

Now, decrypt each pair using the Vigenère Square:

L (Key) + L (Ciphertext): Go to row 'L'. Find 'L' in that row. The column header is A.

E (Key) + X (Ciphertext): Go to row 'E'. Find 'X' in that row. The column header is T.

M (Key) + F (Ciphertext): Go to row 'M'. Find 'F' in that row. The column header is T.

O (Key) + O (Ciphertext): Go to row 'O'. Find 'O' in that row. The column header is A.

N (Key) + P (Ciphertext): Go to row 'N'. Find 'P' in that row. The column header is C.

L (Key) + V (Ciphertext): Go to row 'L'. Find 'V' in that row. The column header is K.

E (Key) + E (Ciphertext): Go to row 'E'. Find 'E' in that row. The column header is A.

M (Key) + F (Ciphertext): Go to row 'M'. Find 'F' in that row. The column header is T.

O (Key) + R (Ciphertext): Go to row 'O'. Find 'R' in that row. The column header is D.

N (Key) + N (Ciphertext): Go to row 'N'. Find 'N' in that row. The column header is A.

L (Key) + H (Ciphertext): Go to row 'L'. Find 'H' in that row. The column header is W.

E (Key) + R (Ciphertext): Go to row 'E'. Find 'R' in that row. The column header is N.

**Decrypted Plaintext:** `ATTACKATDAWN`

**Algebraic Description**

The Vigenère cipher can also be described using modular arithmetic. Assign each letter a numerical value from 0 to 25 (A=0, B=1, ..., Z=25).

Let P be the plaintext letter, K be the key letter, and C be the ciphertext letter.

- Encryption: 
    - Ci=(Pi+Ki)(mod26)

    - Where Pi is the numerical value of the i-th plaintext letter, Kiis the numerical value of the i-th key letter(repeated), and Ci is the numerical value of the i-th ciphertext letter.

- Decryption: 
    - Pi=(Ci−Ki+26)(mod26)
    - We add 26 before taking the modulo to ensure a positive result even if Ci−Ki is negative.

**Algebraic Example (Encryption):**

**Let's encrypt 'A' (0) with key 'L' (11) from "ATTACKATDAWN" and "LEMON":**

C=(0+11)(mod26)

C=11(mod26)

C=11 (which corresponds to 'L')

**Let's encrypt 'T' (19) with key 'E' (4):**

C=(19+4)(mod26)

C=23(mod26)

C=23 (which corresponds to 'X')

**Algebraic Example (Decryption):**

**Let's decrypt 'L' (11) with key 'L' (11):**

P=(11−11+26)(mod26)

P=26(mod26)

P=0 (which corresponds to 'A')

**Let's decrypt 'X' (23) with key 'E' (4):**

P=(23−4+26)(mod26)

P=45(mod26)

P=19 (which corresponds to 'T')

**So , To get the flag I wrote python script that decrypts the Vigenere Cipher and Here it is :**

**Python code for decryption** 
```python 
def vigenere_decrypt(ciphertext, key):
    
    plaintext = ""
    key = key.upper()  # Ensure key is uppercase
    ciphertext = ciphertext.upper()  # Ensure ciphertext is uppercase
    key_index = 0

    for char in ciphertext:
        if 'A' <= char <= 'Z':  # Process only alphabetic characters
            # Convert character to a number (A=0, B=1, ...)
            cipher_char_value = ord(char) - ord('A')
            key_char_value = ord(key[key_index % len(key)]) - ord('A')

            # Vigenere decryption formula: P = (C - K + 26) mod 26
            decrypted_char_value = (cipher_char_value - key_char_value + 26) % 26

            # Convert number back to character
            plaintext += chr(decrypted_char_value + ord('A'))

            # Move to the next key character
            key_index += 1
        else:
            # If the character is not an alphabet, append it as is
            plaintext += char
            # Do not increment key_index for non-alphabetic characters
            
    return plaintext

# --- Example Usage ---
if __name__ == "__main__":
    encrypted_text = "rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_2951c89f}"
    encryption_key = "CYLAB"

    decrypted_message = vigenere_decrypt(encrypted_text, encryption_key)
    print(f"Encrypted Text: {encrypted_text}")
    print(f"Encryption Key: {encryption_key}")
    print(f"Decrypted Message: {decrypted_message}")

```

After execution, this is the output

```terminal
Encrypted Text: rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_2951c89f}
Encryption Key: CYLAB
Decrypted Message: PICOCTF{D0NT_US3_V1G3N3R3_C1PH3R_2951A89H}
PS C:\Users\Sunil Kumar\Desktop\vigenere>
```

Hence I got the , `picoCTF{D0NT_US3_V1G3N3R3_C1PH3R_2951A89H}`

---

# Credstuff

**Challenge** 

![credstuff](/images/credstuff.PNG)

In this challenge we have been given `leak.tar` file  , that file contains large number of usernames and passwords .

So , To solve the challenge we need to find the password that matches with the user name  and decrypt it to get the flag .

So firstly , I have seperated the usernames and passwords and made two files `usernames.txt` and `passwords.txt`(just copied and pasted the usernames in usernames.txt and passwords in passwords.txt) .

**Solution** 

```terminal 
sunil-kumar@DESKTOP-GBKN3LB:/mnt/c/Users/Sunil Kumar/Desktop$ LINE_NUMBER=$(grep -n "cultiris" usernames.txt | cut -d: -f1)
sunil-kumar@DESKTOP-GBKN3LB:/mnt/c/Users/Sunil Kumar/Desktop$ awk "NR==$LINE_NUMBER" passwords.txt
cvpbPGS{P7e1S_54I35_71Z3}
```
Here with the linux commands i got the encrypted flag `cvpbPGS{P7e1S_54I35_71Z3}`

After some observation , i came to know that the flag is encrypted using `ROT13` mechanism 

**What is ROT13 ?**

`ROT13`(short for `rotate by 13 places`) is a simple letter substitution cipher that shifts each letter in the alphabet 13 positions. It's a special case of the Caesar cipher, where the shift is always 13. This means "A" becomes "N", "B" becomes "O", and so on

So we can use online tools to decode that or use python script 

**Python script that decrypts the ROT13 Cipher**

```python 
def rot13_decrypt(text):
    
    result = ""
    for char in text:
        if 'a' <= char <= 'z':
            result += chr(((ord(char) - ord('a') + 13) % 26) + ord('a'))
        elif 'A' <= char <= 'Z':
            result += chr(((ord(char) - ord('A') + 13) % 26) + ord('A'))
        else:
            result += char
    return result

encrypted_text = "cvpbPGSP7e1S_54I35_71Z3"
decrypted_text = rot13_decrypt(encrypted_text)
print(f"Encrypted: {encrypted_text}")
print(f"Decrypted: {decrypted_text}")

```
**Output:**
```terminal 
Encrypted: cvpbPGSP7e1S_54I35_71Z3
Decrypted: picoCTFC7r1F_54V35_71M3
```

So the Flag is : 

**Flag :**  `picoCTF{C7r1F_54V35_71M3}`

---
 