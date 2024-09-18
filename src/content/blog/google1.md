---
author: rifting
pubDatetime: 2024-09-17T19:04:36Z
modDatetime: 2024-09-17T19:04:36Z
title: Breaking Google's parental control suite, part 1
slug: family-link-1
featured: true
draft: false
tags:
  - family_link
  - reverse_engineering
description: How a group of friends and I found a major flaw in Google's parental control suite.
---

## Notice

This post is heavily inspired by CoolElectronics/Velzie's [Breaking CrOS](https://blog.coolelectronics.me/breaking-cros-1/) blog posts. I heavily recommend you go check them out if you haven't already.

## Some Context

Family Link is Google's parental control service. It allows parents to see the location of their children and teens, monitor app usage, and restrict when they can use certain apps, among many other things. What's notable about Family Link is that they have something called a "parent access code". The parent access code is a 6-digit TOTP code refreshed every hour that a parent can use to change settings on their child device if it's not connected to the internet.

Browsing the internet about Family Link, I came across a post on reddit suggesting the possibility of the TOTP code potentially being able to be reverse engineered. The idea defintely intrugied me, and I attempted to get in contact with some people in the post comments as well as a good friend of mine, [spencerpogo](https://github.com/spencerpogo).

## Motives

I decided to work on this project simply because I am a victim to Family Link myself, and found it annoying. I haven't ever actually had to use the TOTP codes, but the app does get on my nerves when it reports everything I do to my parents and enforces the bootloader to be locked with **NO WAY TO UNLOCK IT, EVEN WITH PARENTAL PERMISSION!!**

## Initial Brainstorming

As soon as I was in contact with everyone we decided to search for ways to find the TOTP code algorithim.
There were 2 main ideas we came up with

- Reverse Engineering the Family Link website
- Reverse Engineering the Family Link app

## The Website

Taking a look at the website for Family Link, I assume it won't be too hard. However, looking at the JavaScript that was compiled from dart, I realize just how unintentionally obfuscated this JavaScript truly is.

I looked for any calls to `Date.now();`, knowing the TOTP algorithim would be using a timestamp _somewhere_. Eventually I found a function, but it had contacted many other functions which contacted other functions.

I'm sure I could've reverse engineered the algorithim given enough time, but at this point I decided it was a better idea to reverse engineer the app instead.

## The App

The Family Link app was made with the same codebase as the website; however the flutter was compiled to Java and not JavaScript.
We used [Jadx](https://github.com/skylot/jadx) to look into the family link codebase and the decompiled code was much more understandable than the insanely obfuscated dart-javascript monstrosity the website used.

However, everyone who I had gathered for the team, including me, had trouble making use of the decompiled app code too. It looked like we were at a dead end.

## Ash, the ChromeOS Window Manager/System UI comes to the rescue

That's right. Looking up anything related to the parent access code, ironically, on Google, yieleded some unit tests for Family Link.
The unit tests were useless on the own but they gave me the idea to look through Chromium codesearch, a website to search through the code of Chromium/ChromiumOS, for anything related to the parent access code.

To my suprise, under the ChromeOS WM folder, there lied C++ files for generating and validating the parent access codes.
I notified everyone on the development team and [spencerpogo](https://github.com/spencerpogo) made a rust-based POC for generating the parent access codes, essentially translated from the ash C++ files. The issue now though, was that we didn't know where the secret for the TOTP actually was. We knew it had to be sent to the parent somewhere, but it looked like we would have revist the website and/or app to find the shared secret.

## Finding the secret

We went back to the website and inspected the requests it would make, assuming the parent access code seed/secret would be there somewhere.
We found a long base32 string that looked like it was a TOTP code. We plugged it in to the code generator software and...

Nope. Didn't match the parent code.

It was demotivating, but we kept looking through the requests until we found a base64 string. By some miracle, we plugged it into the software and it worked!

However, this would require the supervised child or teen to have physical access to their parents computer, which is a little privacy-invasive in my opinion. We decided to inform the entire team of the exploit anyway, and [r58playz](https://github.com/r58playz) made a bookmarklet to intercept the XHR and grab the shared secret.

We informed a member of the team how to perform the exploit and he understood. He used the bookmarklet and got the seed.

Normally this would be nothing out of the oridinary, but then he told us he was on his supervised account.

For some godforsaken reason, some Google employee didn't see any problems in sending the shared secret to supervised users whenever they vist the flutter webapp. Even worse, supervised children and teens are signed into Chrome on their phones by default, allowing them to grab the shared secret and generate parent access codes in less than a minute.

Part 2 of this series of posts will detail how Google patched the exploit, and will be out soon.

## Credits

Huge thanks to everyone on the team:

- [Spencerpogo](https://github.com/spencerpogo)

- [r58playz](https://github.com/r58Playz)

- Sleepachu

- [Amaan](https://github.com/4MAANR)

- [ProgrammerIn-wonderland](https://github.com/ProgrammerIn-wonderland), [VadSzil42](https://github.com/VadSzil42), & mertcinarsah
