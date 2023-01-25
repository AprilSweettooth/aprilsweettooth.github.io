---
layout: post
title: 'po4jie3mi4ma3'
# author: peteryang
tags: [blog]
description: >
---

This is a short write-up for the challenge po4jie3mi4ma3 I did for Imaginary CTF round 26.
From the title there's a hint, at least for native chinese speakers.

Description:
I managed to leak a database and some of the source code of this Chinese website. However, I'm having a lot of trouble actually getting into the site. I've recreated the relevant bits, maybe you'll have more luck?

The idea came from this [discussion](https://security.stackexchange.com/questions/55817/chinese-or-pinyin-wordlists-for-dictionary-attacks)

```
import itertools
from hashlib import sha512
h = 'bcfae3f2e987121fb0b7527511d99a7ef4ef3213e1c2993b0016758990f5c01fd109ba819e879f8d6cff2bd79e61a7ef62a8beb7eb8fb655ce3b22a63fec1cd0'
with open('pinyin.txt') as file:
    lines = file.readlines()
    lines = [[c for c in line.rstrip().split(' ') if c.islower()] for line in lines]

all_pinyin = [item for sublist in lines for item in sublist]
all_tones = [1,2,3,4]
text = list(itertools.product(*[all_pinyin, all_tones, all_pinyin, all_tones]))
for i in range(len(text)):
    l = sha512(('ictf{' + ''.join([text[i][0], str(text[i][1]), text[i][2], str(text[i][3])]) + '}').encode()).hexdigest()
    if l == h:
        print('ictf{' + ''.join([text[i][0], str(text[i][1]), text[i][2], str(text[i][3])]) + '}')
        break
```
Function to create database
```
def make_user(username):
    default_pass = open("pass.txt").read().strip()
    assert len(default_pass) == 2
    text = get(default_pass, format="numerical").encode()
    db[username] = sha512(b'ictf{'+text+b'}').hexdigest()
```
login page verification
```
@app.route('/login', methods=["POST"])
def login_post():
    user = request.form['user']
    pwd = request.form['pass']
    if db[user] == sha512(pwd.encode()).hexdigest():
        return f"Logged in successfully as {user}"
    else:
        return "Invalid credentials!"
```