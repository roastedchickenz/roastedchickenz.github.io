---
title: Playing with the Diva (android) - Part 2
date: 2023-01-07 23:00:00 +7
categories: [portable-computer]
tags: [ctf, android, diva-android]
---

Welcome to part 2 of my Diva-android posts.  
Lets jump right into it 😉️

## 5. Insecure Data Storage - Part 3

![Insecure Data Storage - Part 3](/public/diva-android-part2/insecure-datastorage3.png)

I entered "thisisauser" and "notauser" as test data.  
Once again, open <code>/sources/jakhar/aseem/diva/InsecureDataStorage3Activity.java</code> to see the source code we decompiled. 

Scanning through the source code, we see that the developer uses temp file this time.  
The name of the temp file is prefixed with "uinfo" and suffixed with "tmp" as shown by the line:
```java
File uinfo = File.createTempFile("uinfo", "tmp", ddir);
```

The file is also located on "ddir", which if we see the source code defined as:
```java
File ddir = new File(getApplicationInfo().dataDir);
```
It's defined as the directory of the app's package. So this should be pretty simple to find.  
All we need to do is go to the package's directory.  
So I ran this on my host machine:  
```bash
adb shell
cd /data/data/jakhar.aseem.diva/
ls -la uinfo*
cat uinfo<randomnuber>tmp
```

![Insecure Data Storage - Part 3 done](/public/diva-android-part2/insecure-datastorage3-done.png)

Indeed it shows us the credentials I entered thisisauser:notauser  

## 6. Insecure Data Storage - Part 4

This one require an SD card on your android-x86.  
Unfortunately I haven't figured out how to emulate an SD card and connect it to android-x86.  
I tried several things like
1. [adding SD card controller](https://stackoverflow.com/questions/61453355/qemu-to-emulate-sd-bus-and-card)
2. [messing around woth /etc/vold.conf](https://www.android-x86.org/documentation/sd_card.html)
3. manually mounting the SD card to /mnt/sdcard

Maybe I did something wrong along those steps, if you know how to emulate an SD card with/without virt-manager please let me know.  

Anyway, we can read through the source code 
```java
package jakhar.aseem.diva;

import android.os.Bundle;
import android.os.Environment;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.EditText;
import android.widget.Toast;
import java.io.File;
import java.io.FileWriter;
/* loaded from: classes.dex */
public class InsecureDataStorage4Activity extends AppCompatActivity {
    /* JADX INFO: Access modifiers changed from: protected */
    @Override // android.support.v7.app.AppCompatActivity, android.support.v4.app.FragmentActivity, android.support.v4.app.BaseFragmentActivityDonut, android.app.Activity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_insecure_data_storage4);
    }

    public void saveCredentials(View view) {
        EditText usr = (EditText) findViewById(R.id.ids4Usr);
        EditText pwd = (EditText) findViewById(R.id.ids4Pwd);
        File sdir = Environment.getExternalStorageDirectory();
        try {
            File uinfo = new File(sdir.getAbsolutePath() + "/.uinfo.txt");
            uinfo.setReadable(true);
            uinfo.setWritable(true);
            FileWriter fw = new FileWriter(uinfo);
            fw.write(usr.getText().toString() + ":" + pwd.getText().toString() + "\n");
            fw.close();
            Toast.makeText(this, "3rd party credentials saved successfully!", 0).show();
        } catch (Exception e) {
            Toast.makeText(this, "File error occurred", 0).show();
            Log.d("Diva", "File error: " + e.getMessage());
        }
    }
}
```

If I'm reading the source code correctly, the developer decided to use an external storage device to store user's information and store them with a file called <code>.uinfo.txt</code>

So we could use something like:
```bash
adb shell
cd /mnt/sdcard # assuming that sdcard is mounted at /mnt
cat .uinfo.txt
```

## 7. Input Validation Issue - Part 1

![Input Validation Issue - Part 1](/public/diva-android-part2/input-validation1.png)

Reading the description, I can feel my SQLi sense is tingling.  
But first, lets try the username "admin" as suggested by the Hint.

![admin's creds](/public/diva-android-part2/input-validation-admin.png)

We get admin's username, password, and credit card back.  
Earlier, I mention SQLi, lets try the most basic SQLi tricks known to man!
```sql
' OR 1=1 --
```

![sqli attempt](/public/diva-android-part2/input-validation-sqli.png)

And we got them!

Reviewing the source code confirms this is vulnerable to SQLi
```java
Cursor cr = this.mDB.rawQuery("SELECT * FROM sqliuser WHERE user = '" + srchtxt.getText().toString() + "'", null);
```

## 8. Input Validation Issue - Part 2

![Input Validation Issue - Part 2](/public/diva-android-part2/input-validation2.png)
 
Reading the description, my gut feeling said that it's probably fetching local files.  
So I tried inputting these:
```bash
cat /proc/cpuinfo
cat /etc/passwd
cat /etc/shadow
```
and got none.  
Okay, maybe another SQLi?  
Tried them and no luck.

After half an hour or so figuring this out, I finally decided to adb into android

```bash
adb shell
su
cat /etc/passwd 
cat /etc/shadow
```
Both passwd and shadow file does not exists :)

But /proc/cpuinfo exists yet it doesn't return anything. Maybe it's permission issue?
So I tried fetching a file within the package directory. I tried fetching shared_prefs file from the previous challenge
```
file:///data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
```

and got this
![shared_prefs file](/public/diva-android-part2/input-validation2-shared_prefs.png)

Finally 😃️ It WAS fetching local files, should've tried harder at that

## 9. Access Control Issue - Part 1

![Access Control Issue - Part 1](/public/diva-android-part2/access-control-issue1.png)

Okay, so we're supposed to access the API credentials page without opening the app.  
Lets take a look at the source code

![source code](/public/diva-android-part2/access-control-issue1-sourcecode.png)

At first, I didn't encounter anything interesting, until I learn about [Intent](https://developer.android.com/reference/android/content/Intent)

Spesifically, the developer uses [setAction](https://developer.android.com/reference/android/content/Intent#setAction(java.lang.String)) method to do an action, which the developer passed "jakhar.aseem.diva.action.VIEW_CREDS" as the action.  

So we may need to execute setAction manually. I achieve this by executing

```bash
adb shell
su
am start -a jakhar.aseem.diva.action.VIEW_CREDS
```

![Access Control Issue - Part 1](/public/diva-android-part2/access-control-issue1-done.png)

Interestingly, this also works even when diva is not launched

## 10. Access Control Issue - Part 2

![Access Control Issue - Part 2](/public/diva-android-part2/access-control-issue2.png)

When we choose "Register now", it'll tell us to register. And if we choose "Already Registered", it'll bring us to the API credentials page. Great, now we just have to access the "Already Registered" page.  

Trying Intent the exact same way as previous challenge will bring us to the "Register Now" page. So we need to bypass it somewhow. Well, lets read the source code!  

It seems like the code that prevents us from sending an arbitrary Intent like before is this line of code
```java
boolean chk_pin = rbregnow.isChecked();
i.putExtra(getString(R.string.chk_pin), chk_pin);
```

We need to somehow bypass this check.  Looking through am help page, I encounter this
```text
[--ez <EXTRA_KEY> <EXTRA_BOOLEAN_VALUE> ...]
```
Nice!! so we pass an abitrary value to the app. So, I tried this command:
```
adb shell
su
am start -a jakhar.aseem.diva.action.VIEW_CREDS2 --ez chk_pin false
```
and.... it doesn't work. Why? Well, I'm also banging my head to the wall. I also check <code>R.java</code> and of course tried to decode hex to string and get nothing.  
But, checking <code>R.java</code> might not be a bad idea after all, the variable from <code>R.java</code> must exist somewhere right?  
So I search using <code>grep -R -i chk_pin</code> and lo and behold, I found this
```xml
resources/res/values/strings.xml:    <string name="chk_pin">check_pin</string>
```

So we now know why <code> am start -a jakhar.aseem.diva.action.VIEW_CREDS2 --ez chk_pin false </code> doesn't work, because it's called check_pin instead of chk_pin by the application.  

With that in mind, all we need to do is exeucte this
```bash
am start -a jakhar.aseem.diva.action.VIEW_CREDS2 --ez check_pin false
```

and we got it!

![access control issue part 2 done ](/public/diva-android-part2/access-control-issue2-done.png)

## 11. Access Control Issue - Part 3
![access control issue part ](/public/diva-android-part2/access-control-issue3.png)

This challenges feature a create pin function. Interesting. For testing purposes I entered "1234"
After that I can access private notes if I provide the pin "1234"

Looking at the source code for this, it looks like the developer uses shared_prefs again as shown by this line
```java
SharedPreferences spref = PreferenceManager.getDefaultSharedPreferences(this);
String pin = spref.getString(getString(R.string.pkey), "");
```
This has been shown to be NOT a secure way to store credentials, so we might need it later.  
Looking through the source code more thoroughly, I notice something different with this challenges compared to the last two Access control issue challenges. That is, there are no SetAction to be made, this is huge, since the last two challenges I depend on the action to be made and trigger it manually through the command line. Looks like I need some another way!

After giving up for possibly a billionth times I finally found a way. I simply need to access the file in the terminal! 
Instead of setAction, turns out the developer uses hardcoded URI on <code> NotesProvider.java </code> to access the notes shown by this line
```java
NotesProvider.java:    static final Uri CONTENT_URI = Uri.parse("content://jakhar.aseem.diva.provider.notesprovider/notes");
```
I use <code> adb shell content </code> to access the file. Also, While browsing the internet, it turns out that this is a quite obvious vulnerability shown by this [comment](https://stackoverflow.com/a/27993247)

With that in mind, all I need to do is:
```bash
adb shell
su
content query --uri content://jakhar.aseem.diva.provider.notesprovider/notes/
```

![access control issue part ](/public/diva-android-part2/access-control-issue3-done.png)

## 12. Hardcoding Issue - Part 2

![hardcoding issue part 2](/public/diva-android-part2/hardcoding-issue2.png)

This one is quite tough, I tried looking through logcat, decoding <code> R.java </code>, and bruteforce :D. Nothing works.  
What eventually works is looking through the source code again and reading them properly.  

The first and most obvious thing to do is go to <code> Hardcode2Activity.java </code>. Here, I find:

```java
private DivaJni djni;
this.djni = new DivaJni();
```
These lines tells us it loads another file called DivaJni. Opening that file, I find:

```java
private static final String soName = "divajni";

    public native int access(String str);
    static {
        System.loadLibrary(soName);
    }
```
These lines tells us that the application load a library called "divajni". A little searching bring me [here](https://stackoverflow.com/questions/27886092/system-loadlibrary-in-android).  
Accepted answer tells us the extension is different for different operating system. The extension mentioned is either <code> .a </code> or <code> .so </code>  
I try to find both with:
```bash
find . | grep divajni.a
find . | grep divajni.so
```
The first command doesn't output anything, the second command however, outputs many file called <code> libdivajni.so </code> for different architecture. I chose x86_64 since that's what I'm using.
This is a compiled shared library so I can't view what's inside easily. I tried putting  <code> libdivajni.so </code> to jadx again and doesn't seem to work.  
Then I tried the <code> strings </code> command and it outputs a lot of different dependencies libraries and the hardcoded password! 
```bash
$ strings libdivajni.so 

__cxa_finalize
__cxa_atexit
Java_jakhar_aseem_diva_DivaJni_access
Java_jakhar_aseem_diva_DivaJni_initiateLaunchSequence
strcpy
JNI_OnLoad
_edata
__bss_start
_end
libstdc++.so
libm.so
libc.so
libdl.so
libdivajni.so
<$!H
olsdfgad;lh
.dotdot
;*3$"
GCC: (GNU) 4.9 20140827 (prerelease)
gold 1.11
.shstrtab
.dynsym
.dynstr
.hash
.rela.dyn
.rela.plt
.text
.rodata
.eh_frame
.eh_frame_hdr
.fini_array
.init_array
.dynamic
.got
.got.plt
.data
.bss
.comment
.note.gnu.gold-version
```

It's hidden between those library I almost missed them. For the record, I tried <code> <$!H </code> first thinking that is the password. 

![hardcoding issue part 2 done](/public/diva-android-part2/hardcoding-issue2-done.png)
 

## 13. Input Validation Issue - Part 3

![Input validation Issue - part 3](/public/diva-android-part2/input-validation3.png)

Well, it asks to crash the app, the first thing that pop to my brain is inputting a big string. So, I input:
```text
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabbbbaaaggaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

Definitely didn't count the string length but it did crash the app!
To cross-check, I look through logcat and found this

![Input validation Issue - part 3 done](/public/diva-android-part2/input-validation3-done.png)
SIGSEGV - Signal segmentation fault. A classic memory corruption vulnerability


## Done

This has been quite fun! Definitely learn some new tricks 😃️




