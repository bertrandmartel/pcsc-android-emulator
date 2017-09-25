# Smartcard Android Emulator Extension

## Download and run the emulator

* create an avd in Android Studio with system image sdk21 x86 named "smartcard"

* download and run the emulator : 

```bash
mkdir pcsc-emulator && cd pcsc-emulator
wget https://github.com/bertrandmartel/pcsc-android-emulator/releases/download/v1.0/emulator.tar.gz
tar -xvzf emulator.tar.gz && rm emulator.tar.gz
export ANDROID_SDK_ROOT=~/Android/Sdk #change this
export LD_LIBRARY_PATH=$PWD/pcsc-lite:$LD_LIBRARY_PATH
./emulator/emulator @smartcard -verbose -system system.img -pcsc
```

## Build instructions

Instructions for building and using the emulator extension for smartcard from [seek-for-android instructions](https://github.com/seek-for-android/pool/wiki/EmulatorExtension)

### Init

```bash
AOSP_DIR=seek-for-android
PCSC_DIR=~/pcsc-lite
```

### Build AOSP

building & patching AOSP

```bash
mkdir $AOSP_DIR && cd $AOSP_DIR
repo init -u https://github.com/seek-for-android/platform_manifest.git -b scapi-4.0.0
repo sync
. build/envsetup.sh 
lunch aosp_x86-eng
make update-api
make -j32
```

### Build pcsc

```bash
mkdir -p $PCSC_DIR && cd $PCSC_DIR
wget https://alioth.debian.org/frs/download.php/file/4225/pcsc-lite-1.8.22.tar.bz2
current=$(pwd)
mkdir -p 32lib
tar -C 32lib -xvjf pcsc-lite-1.8.22.tar.bz2
cd 32lib/pcsc-lite-1.8.22
sudo apt-get install libudev-dev:i386
CFLAGS="-m32" LDFLAGS="-m32" ./configure; make

cd $current
mkdir -p 64lib
tar -C 64lib -xvjf pcsc-lite-1.8.22.tar.bz2
cd 64lib/pcsc-lite-1.8.22
CFLAGS="-m64" LDFLAGS="-m64" ./configure; make
```

### Build emulator

build emulator with pcsc support

```bash
cd $AOSP_DIR
cd external/qemu
export PCSC_INCPATH=$PCSC_DIR/32lib/pcsc-lite-1.8.22/src/PCSC
export PCSC32_LIBPATH=$PCSC_DIR/32lib/pcsc-lite-1.8.22/src/.libs/
export PCSC64_LIBPATH=$PCSC_DIR/64lib/pcsc-lite-1.8.22/src/.libs/
./android-configure.sh
make -j32
```

### run emulator 

Prior to the following step, assure to create an avd in Android Studio with system-images 21 x86

In my case the avd is named "smartcard" : 

```bash
cd $AOSP_DIR
export ANDROID_SDK_ROOT=/path/to/Sdk
./external/qemu/objs/emulator @smartcard -verbose -pcsc -system out/target/product/generic_x86/system.img
```

If you have to name the reader : 

```bash
./external/qemu/objs/emulator @smartcard -verbose -pcsc "Gemalto Prox Dual USB PC Link Reader [Prox-DU Contact_10800061] 01 00" -system out/target/product/generic_x86/system.img
```

### Disable SIM pin code check

I had some troubles with the SIM pin code keyguard that won't return successfull pincode. Instead it will return "SIM operation failed" and exhaust the pin code retry.

To disable the pin code check (but still have the keyguard view), apply the patch :

```bash
cd $AOSP_DIR
cd frameworks/base
git apply /path/to/0001-disable-sim-pin-code-check-in-keyguard.patch
```

### Troubleshoot

#### no APDU access allowed

* Error encountered : 

```bash
09-10 00:48:49.396 2116-2116/fr.bmartel.smartcardapi E/MainActivity: Error occured:
     java.lang.SecurityException: Access Control Enforcer: no APDU access allowed!
         at org.simalliance.openmobileapi.service.SmartcardError.throwException(SmartcardError.java:134)
         at org.simalliance.openmobileapi.Session.openLogicalChannel(Session.java:339)
         at org.simalliance.openmobileapi.Session.openLogicalChannel(Session.java:378)
```

* Reason

This error occurs because the Access control enforcer refuses to send apdu for your app. You have to add an access rule to authorize your app to send apdu to a specific applet. A rule defines : 
* an applet ID
* a certificate sha1 hash
* a rule (0x01 for always, 0x00 for never, for more sophisticated rule see access rule spec)

To list, add and delete rules, clone [this fork of GlobalPlatformPro](https://github.com/bertrandmartel/GlobalPlatformPro/tree/access-control) on the `access-control` branch :

```bash
git git@github.com:bertrandmartel/GlobalPlatformPro.git
cd GlobalPlatformPro
git checkout access-control
ant -f build.xml
```

* list rules :

```bash
java -jar gp.jar -acr-list
```

* add a rule to authorize apdu for Android app signed with hash `1FA8CC6CE448894C7011E23BCF56DB9BD9097432` (debug keystore), with applet id `D2760001180002FF49502589C0019B01` with the rule `ALWAYS` (0x01) :

```bash
java -jar gp.jar -acr-add -app D2760001180002FF49502589C0019B01 -hash 1FA8CC6CE448894C7011E23BCF56DB9BD9097432 -rule 01
```

Note that this will issue an install for personalization which requires authentication (eg setting the right keys)

Note that you will have to restart the emulator to apply those rule (emulator will retrieve all rules at boot)

#### iccOpenLogicalChannel failed

* Error encountered : 

```bash
09-10 01:01:57.837 2121-2121/fr.bmartel.smartcardapi E/MainActivity: Error occured:
     java.io.IOException: iccOpenLogicalChannel failed
```

* Reason

smartcard has been disconnected

#### Secure Element is not presented

* Error encountered : 

```bash
09-10 01:05:31.855 2098-2098/fr.bmartel.smartcardapi E/MainActivity: Error occured:
	java.io.IOException: Secure Element is not presented.
```

* Reason

No SIM inserted
