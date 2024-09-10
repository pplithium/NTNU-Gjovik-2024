# NTNU-Gjovik-2024
This is material accompanying NTNU Gjovik Hackaton. It contains the hints about the flags, and 'Basic methods to hack an android app'.

**FLAGS, APKs and Keystore**

*Keystore*

`sample_keystore.jks` file can be used for repackaging attacks.

```
password: promon
keyname: promon
keypass: promon
```

*Signing an application with a keystore*

The apk can be signed using apksigner tool. It is located in $ANDROID_SDK_PATH/build-tools/30.0.3/apksigner -> 30.0.3 is the downloaded build tools version, and it can change. Any build-tools version is OK to use.

`apksigner sign --ks sample_keystore.jks`

then enter the password "promon".


*FLAG 1:*

Snake.apk

Hint: You're expected to increase the snake speed by hacking the game by hooking, or repackaging the app. 
After changing the speed of the snake, you need to catch an apple.

```
Points for the first flag:
1st: 1205
2nd: 1105
3rd: 1005
4th: 955
...
39th: 375
```

*FLAG 2:*

Glimpse.apk

Hint: You're expected to bypass the login process.

You can: 
- reverse engineer the app and find out about the login credentials
- use hooking to change the behaviour
- repackage the app and change the behaviour

Note: the app might contain some security checks

```
Points for the second flag:

1st: 1604
2nd: 1504
3rd: 1404
4th: 1354
...
39th: 774
```

*FLAG 3:*

SanitysEclipse.apk

Hint: You're expected to find the flag data. The data might be hidden in the Firebase cloud.

You can, and probably should: 
- reverse engineer the app and find out about the app, how it is retrieving the data.
- repackage the app and change the behaviour of certain methods, or add logging to find out about the secret flag data.

```
Points for the third flag:

1st: 2024
2nd: 1924
3rd: 1824
4th: 1774
...

39th: 1194
```

*Submission App*

You must use the SubmissionApp.apk to submit your flag responses. Upon launching the app, you need to enter your first name, last name, and a memorable number. After that, you can submit your flag responses from the main screen.

**Practical Information**

In this repo, you find app.apk, which is a simple app that asks for a password and verifies it both in Java and in native code. We will use this app for the demonstrations.

To be able to run the app, you need an Android device. For some demonstrations, root access is required. You can run them all on a physical device with root access. But probably the easiest thing to do is to install Android Studio from here: https://developer.android.com/studio and then install an emulator that has root access. You will need to setup a device with an emulator without Google Play to be able to get root access. Also, you want to use an emulator that matches the architecture of your CPU for optimal performance. In the demonstrations, I will assume that we are using an ARM64 emulator so you might have to make slight modifications if your are running on a different platform.

### Required tools
jadx: https://github.com/skylot/jadx

**Reverse engineering**
Here, I show how you can reverse engineer the app to find out what the correct passwords are.

String Resources

You will need a tool called jadx, which you can download from here: https://github.com/skylot/jadx.

You can launch the tool by invoking jadx-gui or jadx-gui.bat.

You can then load the app into it and explore the resources of the application. Application's strings are usually under res/vaues/strings.xml.

Java code

You will need a tool called jadx, which you can download from here: https://github.com/skylot/jadx.

You can launch the tool by invoking jadx-gui or jadx-gui.bat.

You can then load the app into it and explore the Java code.

The interesting code is in the class no.promon.ntnu.MainActivity where we find the checkPasswordJava method that contains the password.

**Repackaging**
Here, I show how to modify the app on disk in different ways to accept any password.

Java code
To repackage the Java code of the app, you need to download apktool from here: https://bitbucket.org/iBotPeaches/apktool/downloads.

To make the Java check always succeed, perform the following steps:

Find the location you want to modify using jadx, which we have already done. It's the checkPasswordJava method in the no.promon.ntnu.MainActivity class. Our goal is to always make it return true.

Decompile the code of the app with apktool: 
`java -jar apktool.jar d --no-res app.apk`
Open the smali code of the MainActivity class:

`app/smali/no/promon/ntnu/MainActivity.smali.`

If you want to learn more about the Dalvik bytecode, you can check here: https://source.android.com/docs/core/runtime/dalvik-bytecode

Find the code of the checkPasswordJava method and change it to:

    .method private checkPasswordJava(Ljava/lang/String;)Z
        .locals 1
        const/4 v0, 0x1
        return v0
    .end method

This makes the method always return true.

You then compile the app again:

`java -jar apktool.jar b app/ -o app-patched.apk.`

Finally, you have to zipalign: 

`zipalign -p -v 4 app-patched.apk app-aligned.apk`

And sign the app:

`apksigner sign --ks $KEYSTORE --ks-key-alias $KEYSTORE_ALIAS --ks-pass pass:$KEYSTORE_PASS --out app-signed.apk app-aligned.apk`

And then you can install the app to check that your modifications work:

`adb install app-signed.apk.`

The app should now accept any password.

**Hooking**
Here, I show how to modify the app in different ways to accept any password while it is running.

As a prerequisite to this, you have to install Frida on you computer: pip install frida-tools.

In addition, you have to install frida-server on your emulator:

Download the latest version of frida-server (frida-server-xxx-android-arm64.xz) from here: https://github.com/frida/frida/releases.
Unzip the file and push it to the emulator: adb push frida-server-xxx-android-arm64 /data/local/tmp/frida
Make the file executable and run the server:

`adb shell`
`su`
`chmod +x /data/local/tmp/frida`
`/data/local/tmp/frida`

### Java code

To make the java check always succeed, perform the following steps:

Find the location you want to modify using jadx.

In the class `no.promon.ntnu.MainActivity`, we find the `checkPasswordJava` method. Our goal is to always make it return true.

Create a new file java_hook.js with the following content:

    Java.perform(function()
    {
        var activity = Java.use("no.promon.ntnu.MainActivity");
        activity.checkPasswordJava.implementation = function (password)
        {
            console.log("Entered: " + password);
            return true;
        };
    });

Launch the app on the emulator.
Run Frida on your computer to inject the hook:

`frida -U -F -l java_hook.js`

The app should now accept any password for the Java check.

### Instrumentation

There are two ways that you can do the instrumentation of any code in an Android Application. 

1. We can repackage the application and add a call to this method. In case we expect and are interested in the return value, then we can log the return value with a normal android logger class.
2. We can instrument the method via calling it using Frida. In this case, we will focus on the instrumentation using Frida:

In order to instrument the code, you should firstly find the class, and then, simply call the method with a JS script:

    Java.perform(() => {
        const mainActivity = Java.use(
          'no.promon.ntnu.MainActivity'
        );
        var retVal = mainActivity.getPassword();
        console.log("retVal:" + retVal);
      });


This will print the return value for the method we have called.


### Launching Application and reverse engineering with frida

First, you need to install frida tools using python:

```
pip3 install frida-tools
```

You can launch the app by its package name

```

import frida, sys

app = "no.promon.example"

device = frida.get_usb_device()

def spawn_app():
    # launch the app
    pid = device.spawn([app])
    # attach frida to the app
    process = device.attach(pid)

```

You can run your script after you obtain a process instance:

```
def spawn_app(jscode):
    # launch the app
    pid = device.spawn([app])
    # attach frida to the app
    process = device.attach(pid)
    # run the script you have predefined
    script = process.create_script(jscode)
    script.load()
    device.resume(pid)

```

Example script definition:

```
jscode = """
Java.perform(function()
{
    let cls = Java.use("no.promon.example.MainActivity");
    cls.onCreate.implementation = function (bundle)
    {
        console.log("bundle: " + String(bundle));
    };
});
"""
```

This script will print the argument passed to onCreate method.

Let's run the script with entire code:


```
import frida, sys

app = "no.promon.example"

device = frida.get_usb_device()

jscode = """
Java.perform(function()
{
    let cls = Java.use("no.promon.example.MainActivity");
    cls.onCreate.implementation = function (bundle)
    {
        console.log("bundle: " + String(bundle));
    };
});

def on_message(message, data):
    print(message)

def spawn_app(jscode):
    # launch the app
    pid = device.spawn([app])
    # attach frida to the app
    process = device.attach(pid)
    # run the script you have predefined
    script = process.create_script(jscode)
    script.on("message", on_message)
    script.load()
    device.resume(pid)

try:
    spawn_app(jscode)
    sys.stdin.read()
except KeyboardInterrupt:

```

`spawn_app(jscode)` call runs the entire script.

You can run this command after you have activated frida server on the device with:

```
python3 hook.py
```

`hook.py` is your script's file name.