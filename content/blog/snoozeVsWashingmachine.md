+++
title = "snooze vs washing machine: the epic hack"
date = 2022-04-30
+++

My college's hostel has a rather dubious washing machine. It's truly 
a lifesaver because I hate doing laundry by hand. But using it costs 
a bit extra, which honestly, I don't mind paying for. What is more 
interesting to me is that the washing machine uses an Android/iOS app 
to turn it on, and this app makes sure you don't use more than what you 
have actually paid for. Now I don't know about you, but this really 
gave me a "sussy baka" vibe (as the cool kids say nowadays? or is 
this meme dead? IDK). You see, the washing machines are placed where 
the wifi signal is too weak to be useful, and I didn't see any 
Ethernet cables around the machines too, which made me suspect that 
the washing machine itself isn't really connected to the internet. 

<p align="center">
  <img class="no-border" style="height: 300px;" src="/washingmachine.png" />
</p>

Hmm, so here's the deal. A machine with no internet, and an app on 
your phone that talked to the machine. Well, the first thing I had to 
really figure out was how my phone and the machine communicated with 
each other. This was rather easy tho because it was common knowledge 
among everyone who used the washing machine that your phone should 
have both Bluetooth and location services turned on whenever you 
use the washing machine's app. So, I think it's somewhat obvious 
that our little app and the machines both talked to each other using 
Bluetooth LE. 

About a year ago, I looked at the Bluetooth LE protocol because I 
got curious about how my smartwatch (a Mi Band 4) worked, so I have 
a general understanding of how Bluetooth LE worked. Here's a quick 
summary. Bluetooth LE uses a specification called GATT (Generic 
ATTribute profile). The basic idea is every device has a profile, 
which has several "services". Each service has several 
"characteristics". Characteristics are what really hold data, and 
services group relevant characteristics together. Profiles define 
how your device gets accessed, so we don't have to understand it. 
Each service and characteristic has a 128-bit UUID to uniquely 
identify them. That's the general gist. So, my first step would be 
to look at all the services and characteristics that are advertised 
by the washing machines. Luckily for me, there is an android app 
just to do this, it's called [nRF Connect](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp&hl=en), so a quick scan told me, 
there was only one service that was advertised by the washing 
machines. The name of the service was called "Nordic UART Service" 
and it had two characteristics. "TX" and "RX". Huh, that looks like 
UART. So, I can send stuff to the machine, and it will give me back 
stuff. Hmm... But I have no clue the app sent and what the machine 
sent back.

Hmm... okay! So now I know how my phone talks with the washing machines. 
It'd be nice if knew exactly what it sent and received tho. I looked 
around the internet on how I could sniff Bluetooth LE traffic, and one 
convenient way was to enable Bluetooth HCI logs in my Android phone's 
developer options. But sadly, my phone simply refused to generate the logs. 
It wasn't anywhere in storage. I even generated a bug report 
(using `adb bugreport`) to see if the logs would show up in them, but sadly, 
my phone didn't generate them. I could have borrowed a friend's phone, but 
sadly I was too scared to ask lol. This is where I stopped exploring a few 
months back, until a few days ago, rediscovered my notes about these findings 
lol. But this time, I had a different idea. *drum rolls* STATIC ANALYSIS TIME!

That's right y'all, it's time to reverse engineer an Android app. Honestly, 
IDK why I didn't think of this a few months back, because it seems so obvious, 
but I guess I've never reversed an Android app before. Well, I kind of did a 
few years ago, like sometime in 2013-2014, but a lot of things have changed 
since then. For example, dalvikVM got replaced completely with ART. Either way, 
I got myself a copy of `JADX` and prayed to the dev gods that the Android apps 
didn't use any UI frameworks like React Native or Flutter because I have 
literally zero clue how to reverse them lol. Plopped the APK right into JADX 
and BAM! Out came somewhat readable Java code. LETS GOOO!!! Now to spend a few 
hours comprehending the code.

Now, this is when I realize, that whatever entity got commissioned to design 
this part of the washing machine (by "this part", I mean the Android app and the 
little microcontroller and whatnot in the washing machine that spoke back to 
the app) did a terrible terrible job. Like I am an undergrad noob, and even I 
would have seen the shortcomings of what they designed... and yes, I'm 
intentionally not naming the company.

Honestly, the hardest part of reversing is to figure out where to start. One 
could sit through and read the whole thing, but that would mean investing a 
metric crap-ton of hours on something. Luckily for me, the APK didn't strip all 
debug symbols! The class names and function names were intact, great! So, me 
being a reversing noob, started to look for `MainActivity.java`, but I didn't find 
it. Hmm... time to go back to the fundamentals. Looking at `AndroidManifest.xml`, 
we find that `SplashActivity` looks like our entry point. 

Looking through `SplashActivity.java` makes it obvious that it simply starts 
`HomeActivity` after two seconds. Ok, cool! `HomeActivity.java` has a bunch of 
interesting functions. A massive chunk of it deals with scanning QR codes lol. 
Then there is an interesting `connectDevice()` function (again, debug symbols saving 
me a lot of time and hair-pulling), which in turn calls `checkForNewDevice()`
after a few checks. `checkForNewDevice()` simply sets up a Bluetooth LE scan with 
a callback. The callback literally matches the advertised device name to a 
string (this string is simply the machine ID prefixed with a constant string), 
and if it found it, calls a function called `foundDevice()`, which launches the 
`ConnectMachineActivity`.

<p align="center">
  <img class="no-border" src="/washingmachine-reverse-map.png" />
</p>

Cool cool, so all of this tells me, I somehow need to get a `BluetoothDevice` 
object for the washing machine! Yea, I can do that. What next tho? Time to go 
look at `ConnectMachineActivity.java`. `ConnectMachineActivity` happens to be 
huge, so I again will try to rely on function names. Again I'll save you the 
sad lonely weekend of reversing and describe to you what the code actually does. 
And I swear to god, it's downright atrocious. The code first checks if you have 
enough balance and whatnot, then discovers the services, and checks for the 
UART service(the UUID for which is hardcoded, which is cool, I could check my 
work) to which it then connects. Then it discovers the TX and RX characteristics 
and sets up notifications on them so that the app gets to know if their values 
change. And then it simply sends a predetermined string. Wut? LIKE IT LITERALLY 
SENDS THE SAME STRING EVERY TIME.

<p align="center">
  <img class="no-border" src="/washingmachine-flaw.png" />
</p>

And that's it, well, I'm simplifying a lot of 
stuff here, because all this code was mixed in with telemetry code... Now if you 
don't see the problem here, it's the fact that the machine doesn't even verify the 
connection attempt. I might just as well simply write my own app that simply 
connects and sends the same string. A security nerd might tell you that the system 
is prone to a reply attack, oof, fancy words amirite? But the fact that someone 
got paid to design this still boggles my mind. To their credit, the design patterns 
in the code were neat, using an `MVVM` architecture, neatly separating the UART 
code into a service(not the BLE service, I'm talking about Services in Android 
apps here), were my expectations rather low? Maybe, IDK. Either way. Time to put 
this newly gained knowledge to test. 

I wrote a tiny Android app by basically yoinking the reversed `UARTService` code 
and simply sending the string. And wouldn't you know it, the washing machine 
turned on! HAHA! Sweet sweet reward for all this work. Either way, it was a fun 
challenge. 

<p align="center">
  <img class="no-border" style="height: 400px;" src="/washingmachine-getpwnd.png" />
</p>

Now that we did all this, I think it's also worth going all the way to think about 
ways to fix this because come on, it probably isn't that hard. 

* Solution 1: Connect the washing machines to the internet, and send the order ID 
  to the machine. Then the washing machine can verify that the order is valid. Well, 
  this has an obvious disadvantage of needing to connect the washing machines to the 
  internet.

* Solution 2: Digitally signed payload with an expiration. This probably is what they 
  should have really implemented. The idea is, instead of sending this constant string 
  as the payload, generate an expiration time (eg. the UNIX timestamp of one minute 
  from now), send it to an API to digitally sign it using the companies identity, and 
  send that as a payload. This doesn't even require the washing machines to be connected 
  to the internet. It was as easy as that, ugh. Whatever.

Well, that's all from me for now! Either way, was a fun way to spend a weekend. ?????????. 

<p align="center">
  <img class="no-border" style="height: 200px;" src="/catsneep.png" />
</p>