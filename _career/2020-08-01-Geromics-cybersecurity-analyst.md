---
layout: career
title: 'Geromics-Cybersecurity Analyst'
# author: peteryang
tags: [career]
img: cyber.jpeg
display-order: 1
description: >
---

Company: Geromics (Cambridge) 

Role: Penetration Testing

- See my full [documentation](https://medium.com/@sqhy2)

---

![](/public/img/geromics/plz.png)
<!--add caption?-->

During my research at Geromics, I've learnt a lot about security issues and the tools to find vulnerabilities. It was a good practice for me to try my previous Pen Testing knowledge and to help the company reduce potential attacks. Below I will list the things I've done during the period, the potential issues I found and their countermeasures.
Just to clarify terms at the start… a potential vulnerability is defined as a possible error in the code, for example, that could lead to an attack. However, a vulnerability is a verified error. These terms are vaguely defined, but below I'll call it 'finding a vulnerability' only if we find an exploit. Until then, I'll just call it a potential vulnerability ;-)

---

## JSON Token Attack

See my detailed description [here](https://medium.com/geromics/a-simple-json-token-opens-an-attack-surface-5a968a0f0f96).
Tools used:
- Burp suite

Used to intercept the request made to and from the server. Used as a 'repeater' to test the token response on localhost:

```
{
 "session": [
   {
     "id": "01dfc782-5921-41c5-816c-d4a0bf467cf5",
     "clientName": "FileUploader",
     "createdAt": 1655125286245,
     "rate": 5,
     "modifiedAt": 1659607222175
   }
 ]
}
```
![](/public/img/geromics/token.jpeg)
- Postman (similar to burp):

Used to send GET/POST requests to the actual server.

---

## Caddy

Caddy is used in our DevOps deployment as a reverse proxy.

- If Caddy is broken into, everything else, e.g. the database, is potentially exposed to attacks!

![](/public/img/geromics/cad.png)

Potential vulnerabilities:
- Fuzzing the URL when Caddy tries to parse it → possible server crash.
- Contrary to common 'URL fuzzing', we do not want to find hidden directories or files, because we know there is nothing sensitive on the server.

Tools:
- AFL++ (There are some great videos by LiveOverflow as well as a hands-on exercise for learning it).
- go-fuzz For fuzzing applications written in Go.

We AFL++ is great but (from what I know) all afl++ could fuzz is a binary with C/C++ compiler while Caddy is written in Go. So I used gofuzz. This [blog](https://medium.com/@pliutau/fuzz-testing-in-go-d22eb85e461b) explains very clearly the steps to fuzz the binary.
The entry point is in the source code:

```
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request, next caddyhttp.Handler) error {
```

Countermeasures:
- When we sought advice from the author of Caddy on [this thread](https://github.com/caddyserver/caddy/issues/4889),

![](/public/img/geromics/git.jpeg)

it seems that 'URL' is parsed into different parts and we want to fuzz the hostname which is in the 'host' header of the request, and in the TLS handshake's SNI field (Server Name Identification). But because of the way caddy filters invalid input, as soon as the hostname is invalid, it will not do anything further.

- The next thing I looked at is Web Application Firewall (WAF) for Caddy. For now I do not think Caddy has a default WAF implemented but one package called Coraza is integrated with Caddy and seems to be a popular choice.

---

## JWT
We use JWT for all communication between our AWS server and our 'middleware' application for verification and validation.

Potential vulnerabilities:

- Revoke Access problem: Basically when we found some institute or person has bad intentions and we tried to add them to a blacklist, there is a 3–5 mins gap when they still have the access to the server if issued with JWT previously.

Countermeasures:

I try to prevent similar attacks by having a parameter called 'valid' when sending the request. Setting this flag will make whatever the request the user sent invalid. The problem about revoke access is that the user already got a JWT token, which means the user will still be able to use the Clinoverse backend. Therefore a server restart will make that change effective immediately, but that will also drop other users. They will need to authenticate again. Token refresh may solve this issue, while checking the user status upon renewing the token.

```
{
  users: [
    {
      "userDid": "did:clinoverse:...",
      "publicKey": "...",
      "valid": true,
      "createdAt": 1234567890123
    },
    .
    .
    .
  ],
  .
  .
  .
}
```

So why aren't we worrying about this too much? The hacker needs more time to make the cryptographic hash valid for the modified token. If the attacker doesn't have other information than the algorithm for crypto-analysis, the only option will be brute-forcing. Other preventions are about database implementation like Redis. Using redis will not require the blacklist so there is no revoke problem(no 'invalid' interval). without redis, the request is made with JWT to the server. But in order to get less time, we will keep the current structure.

---

## Data Encryption
Our client side application encrypts the users genome data.
Potential Vulnerability:

- From this article, it uses poattck to get the plaintxt from just IV and padding oracle. What we are trying to do with the genome data is to encrypt each line of the file with CBC scheme. And the IV (initialisation vector) for each line is related to the rsID an identifier for the SNP data).
- Because the data is limited to certain characters and words, it is vulnerable to a statistical attack. See example below:

```
rsid       chromosome position genotype
rs11240777 1          798959   GG
...
```

Countermeasure:
- Encryption scheme change: We effectively chop the 128 bits private key 'in half'. One 'half' is used to generate a 'master key' by concatenating with a constant "A" and hashing. The other 'half' is used to generate an IV for SNPs by concatenating it with a constant "B" and hashing. Then the genotype of each SNP is encrypted using a specific 64-bit 'row key' which is used to XOR with the matching row. So even if the rsIDs are limited, we have different IV mingled with it to make the encryption different for each rsID. For example, for the same GG type, we always have different ciphertexts.

```
constant_A, constant_B
SHA3(<private_key> + constant_A) -> master_key
SHA3(<private_key> + constant_B) -> IV
row_key_1 = AES( <master key>, <IV> + <SNP ID 1> )
...
row_1 xor row_key_1 -> ciphertext_1
...
```

- We can also make a restriction on the file size (which prevents a malicious user who uploads random data to get heavy storage on our backend) so that users cannot get as much information as they want.

```
{ fileType: 'genome' }
if filesize > ...:
   reject!
```

---

## Twisted Attack
As part of the 'registration' process the App generates public-private key pair for the user (to serve as their Decentralised IDentifier).
Potential vulnerabilities:
- This is an interesting one related to elliptic curve attack. I found this blog post.
- When looking into the way to compromise the user's private key. It summarises to the point that private key could be revealed as soon as the user uses their keys to interact with an attacker using an insecure secp256k1 implementation.

Tools:
- CoCalc: to run sagews file, checking the implementation of the attack.
- Sage

![](/public/img/geromics/co.jpeg)

Countermeasures:
So what do we do then? Other than checking the curve, if it is secp256k1, we actually do not reveal even the public key anywhere!

```
Client →POST(auth<u_id, u_msg, u_smsg>) → Clinoverse →OK(jwt(u_id, u_role)) → Client
```

Using standard libraries for secp256k1 means that this vulnerability is purely academic.

---

## Probing the APK

Our Clinoverse product has a mobile app for both users and doctors.
Tools used:
- Android studio (emulator) + objection

I mainly used emulators in android studio. After running as root and installing our app's APK file with ADB, I set up Frida server to run objection, which is a runtime mobile exploration toolkit. It is very powerful for pen testing to perform numerous functions like SSLPinning bypass, root detection bypass and more.

- Android physical device + scrcpy + Burp

This time I used burp to intercept the web traffic between the server and app with a certificate installed on a real android device.

- Apkleaks

I just learnt this tool from this [talk](https://www.youtube.com/watch?v=HmDY7w8AbR4) given by Jason Haddix on NahamCon2022. It found a bunch of dependency packages as well as something below which seems to be vulnerable:

```
[Facebook_Secret_Key]
- fbd5cd62bc65a09'],['e1031be262c7ed1b1dc9227a4axxxxxx
- FBF_HASH = "12345678xxxxxxxxxxxxxxxxxxxxxxxx
```

If these 'secrets' are from the android phone, then attackers might backtrace to find app api leakage from the user. And that is a huge problem. Or It might be the authentication keys between our application to some packages we used from Facebook. But I am totally new in this area so I'll just leave it there.

---

## Miscellaneous Tools

# wfuzz

It is actually my first finding. I used wfuzz to get any hidden directory and files. Then I found this page with a pop-up:

```
                  https://geromics.co.uk/demo
```
![](/public/img/geromics/sign.jpeg)

Well, it seems like a common SQL injection CTF challenge and tbh the password and Username can be easily guessed/fuzzed. However, nothing confidential was being stored here.

# Trufflehog

Then for the repository, trufflehog did help me discover some AWS key secrets that are surprisingly stored in GitHub (in plain sight).

Potential vulnerabilities include:

- Sensitive data leakage
- Secret compromise
- Social engineering attacks

Countermeasure:

Remove them!

---

It was truly a great journey and I learnt that Pen testing is more than just testing the part of the application where strongly protected. We cannot spend all the time evaluating the strength of windows while leaving the door wide open. We need to try all sorts of ways to test in all directions. Thanks for all the help I get from Dan and Yuceer on the way. Also I am grateful to people who delivered us such great guest talks!