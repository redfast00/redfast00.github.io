---
layout: post
title:  "Reversing element alchemy games"
date:   2016-08-21 00:00:00 +0000
category: Hacking
tags: [Android, Reversing]
---
# What are element alchemy games?

An element alchemy game (later EAG) is a game where you start of with the four
natural elements (air, fire, water and earth) and combine them in order to form
other elements. For example: earth + fire = lava. You then use the newly created
elements to make other elements.

## Little Alchemy (well, that was easy)

I was stuck in [Little Alchemy](https://littlealchemy.com/), so I decided to
reverse engineer the [Android app](https://play.google.com/store/apps/details?id=com.sometimeswefly.littlealchemy).
After downloading it to my computer using `adb`, I unpacked the APK using [apktool](https://github.com/iBotPeaches/Apktool).
I quickly saw that the app was written in JavaScript?! After a bit of googling,
I found out that it was built with [cordova](https://cordova.apache.org/).
I looked for files with information about the recipes. I found two files:
`assets/www/resources/base.json` containing the recipes and
`assets/www/resources/en-us/names.json` containing a mapping between item_ids and names. Bingo!
Afterwards, I wrote a script to convert them to [my format](https://github.com/redfast00/element-alchemy-cheater).

## The joybits family (are they even trying?)

I wanted to try to reverse another app and looking for a challenge, I downloaded
[Doodle God](https://play.google.com/store/apps/details?id=joybits.doodlegodfree2)
and [Doodle God Evil](https://play.google.com/store/apps/details?id=com.joybits.doodledevil_free).
Sadly enough, there wasn't much of a challenge: I unpacked the app with apktool, and
found a whole bunch of reaction_xyz.txt files in `assets/data`. I banged
together a Python script to parse them. Onto the next one!

## Doodle Alchemy (regex FTW)

Finally, a bit of a challenge. I downloaded
[Doodle Alchemy](https://play.google.com/store/apps/details?id=com.byril.alchemy).
I unpacked the app with apktool, but didn't find any recipe files. Since the
game was playable offline, the recipes had to be somewhere in the APK and were
probably embedded in the code. I converted the apk to a Java jar using dj2-dex2jar
and then tried to decompile it with JD-GUI. JD-GUI errored because it does not
support Java 5+ language features. I then tried the opensource
[procyon](https://bitbucket.org/mstrobel/procyon/wiki/Java%20Decompiler).
That worked much better, and I found the element names in `com.byril.alchemy.objects.ElementName.java`
and the recipes in `com.byril.alchemy.scenes.GameScene.java`. After some
wrestling with regex patterns on http://regexr.com, I managed to write a scipt to convert the recipes
and names to [my format](https://github.com/redfast00/element-alchemy-cheater).

## Alchemy Fusion 2 and Alchemy Classic (same old, same old)

This was basically the same as Doodle Alchemy: decompiling the app into Java
source code and then grepping for the names of elements.

# Conclusion

I wrote a [Python program](https://github.com/redfast00/element-alchemy-cheater) for easily
cheating in element alchemy games. I also included all recipes in JSON.
This was a fun exercise, let me know if you want me to reverse another app.
