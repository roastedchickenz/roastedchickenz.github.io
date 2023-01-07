---
title: Playing with the Diva (android) - Part 1
date: 2022-12-24 10:00:00 +7
categories: [portable-computer]
tags: [ctf, android, diva-android]
---

Today I'm going to share my experience playing [diva-android](https://github.com/payatu/diva-android) for the first time.  
Diva-android is an intentionally vulnerable android app in an attempt to educate pentester on the basic of android pentesting and developer on secure coding. Diva has a total of 13 challenges. Lets just jump into it.

## Installation

You can build from source by following the instruction [here](https://payatu.com/damn-insecure-and-vulnerable-app/) but I'm lazy so I'm using the apk version found [here](https://github.com/0xArab/diva-apk-file).  

I use [Android-x86](https://www.android-x86.org/) to run android on my PC.  Even if I have a spare android device, I'd personally still use android-x86 because it's by default rooted already and we have a terminal built-in :D  
I ran android-x86 using qemu and virt-manager. 

After android-x86 up and running, install the DivaApplication.apk to android-x86.  
So on your host machine run:

```bash
sudo apt install adb # if you haven't already
adb devices # this will return nothing
adb connect <your-android-x86 IP address>:5555  # you'd need to run this everytime you boot the android-x86
adb devices # verify that your android-x86 is connected
adb install DivaApplication.apk
```
You can get android-x86's IP here

![get android-x86's IP](/public/diva-android/ipaddress.png)

Installation complete!

![terminal + diva](/public/diva-android/after-installation.png)


## 1. Insecure Logging

![insecure logging](/public/diva-android/insecure-logging.png)

Let's try entering some numbers and check where is this credit card being stored.  
I entered "1234567890" because why not?

Since it's about logging, we can try something called [logcat](https://developer.android.com/studio/command-line/logcat).  

On your host machine, run:
```bash
adb shell ps | grep diva # note the PID 
adb shell logcat --pid=<PID given above>
```
You might need to re-enter the credit card number just to make things easier to find. 

![credit card number logged](/public/diva-android/insecure-logging-done.png)

We can see the credit card number logged just like that, now imagine if this is your pin / CVV.  
I think this is equivalent to using GET request for username and password 😅️ 

## 2. Hardcoding Issues - Part 1

![Hardcoding issues part 1](/public/diva-android/hardcoding-issues-1.png)

To find out a hardcoding issues, most of the time we'd need the code (duh).  
Since this is an open-source project, we could just go to [diva-android's repo](https://github.com/payatu/diva-android) and search through the source code. But if we're pentesting a real application, chances are we don't have access to the source code so what should we do?

We could decompile the application using [jadx](https://github.com/skylot/jadx), even then it's not a guarantee since a lot of application would obfuscate their code but hey, it's a start!

Installing jadx is very simple. Download the zip, unzip it, feed jadx the program you want to decompile and you're good to go
```bash
wget https://github.com/skylot/jadx/releases/download/v1.4.5/jadx-1.4.5.zip
unzip jadx-1.4.5.zip
cd bin
./jadx DivaApplication.apk
```

If everything went well, we should get 2 directory as an output called "resources" and "sources".  
Anyway, the thing we're interested in is at <code>/sources/jakhar/aseem/diva/HardcodeActivity.java</code>  
It's not hard to spot the password after seeing the source code 😉️

![hardcoding source code](/public/diva-android/hardcode-sourcecode.png)

Entering the password to diva will give us a notification Access granted as expected after reading the source code. 

![vendorsecretkey](/public/diva-android/vendorsecretkey.png)

Btw, if you don't want to decompile the application for some reason, like I said we could check the source code from [github](https://github.com/payatu/diva-android/tree/master/app/src/main/java/jakhar/aseem/diva) 

## 3. Insecure Data Storage - Part 1

![Insecure Data Storage - Part 1](/public/diva-android/insecure-data-storage1.png)

I entered "not a user" and "myverysecurepassword" as a test data.  

We have decompiled the application so don't let it go to waste and go to <code>/sources/jakhar/aseem/diva/InsecureDataStorage1Activity.java</code>

Reading through the source code, it looks like the developer uses [Shared Preferences](https://developer.android.com/training/data-storage/shared-preferences). Shared preferences is basically a file that store small key-values data about the user.  

A little googling/duck duck going/searxing/binging/whatever search engine you use [shows](https://stackoverflow.com/questions/6146106/where-are-shared-preferences-stored) that shared preferences usually stored in <code>/data/data/YOUR_PACKAGE_NAME/shared_prefs/YOUR_PREFS_NAME.xml</code>

To access the file, I simply ran:
```bash
adb shell
su
cd /data/data/jakhar.aseem.diva/shared_prefs
cat jakhar.aseem.diva_preferences.xml
```

And this is the output that I get
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="password">myverysecurepassword</string>
    <string name="user">not a user</string>
</map>

```

## 4. Insecure Data Storage - Part 2

![Insecure Data Storage - Part 2](/public/diva-android/insecure-data-storage2.png)

I entered "a real user" and "thisisapassword" as a test data.  

Again, open the source code at <code>/sources/jakhar/aseem/diva/InsecureDataStorage2Activity.java</code> 

Very quickly, we could see that the developer uses sqlite databases with the databases name "ids2" as shown by this line.
```java
this.mDB = openOrCreateDatabase("ids2", 0, null);
``` 

I could probably find out where the usual location like the previous challenges but I tried something different for this one. I simply ran:

```bash
adb shell
su
find / -path /proc -prune -o -print | grep ids2
```
Am I overengineering this? probably, but it works and that's all I care about.  
Anyway, it's located in <code>/data/data/jakhar.aseem.diva/databases/</code>

To retrieve this file, on my host machine I ran:
```bash
adb root
adb pull /data/data/jakhar.aseem.diva/databases/ids2 ~/Desktop/
```

Opening this file with a text editor will give us random character, so we need to go to [sqlite viewer](https://sqliteviewer.app) and open our file.

![user password on sqlite](/public/diva-android/insecure-data-storage2-done.png)

And there it is my credentials. 

## End
I think I'm gonna end it here. Even though this is an outdated project (as of this writing, the last commit made was 2016) it's still quite fun especially since I'm still a beginner to android pentesting. 

Thank you for reading 😉️




















