# build-FreeFileSync-on-raspberry-pi
FreeFileSync is a great open source file synchronization tool.
Building from source on linux is simple *if* all the necessary dependencies are installed.
These instruction capture the necessary steps for installing the necessary dependencies for Raspbian (Raspberry Pi OS) and compilation of FreeFileSync

This version of instruction apply to the following:

Item  | Release/Version
------------ | -------------
Rasbian (Raspberry Pi OS) | ```Release 3.6  Jan 2021```
FreeFileSync | ```v11.14```

## 1. Download and extract the FreeFilesSync source code

As of this writing, the latest version of FreeFileSync is 11.14 and it can be downloaded from: 

https://freefilesync.org/download/FreeFileSync_11.14_Source.zip

For some reason, wget **DID NOT** successfuly download the file on the first try (instead it downloads a portion and silently exits). Simply try the wget command a second time or you can manually download it through a browser.

Move the .zip file to the desired directory and uncompress
```unzip FreeFileSync_11.14_Source.zip```

## 2. Install available dependencies via apt-get
These instructions reflect building FreeFileSync using libgtk-3. This may lead to a non-optimal user experience- see:
https://freefilesync.org/forum/viewtopic.php?t=7660#p26057

The following dependencies need to be installed to make the code compile.
- libgtk-3-dev
- libxtst-dev

```
sudo apt-get update
sudo apt-get install libgtk-3-dev
sudo apt-get install libxtst-dev
```

## 3. Compile dependencies not available via apt-get

The following dependencies could not be installed via `apt-get` and need to be compiled from their source code.

### 3.1 gcc

FreeFileSync requires a C++ compiler that supports the C++20 standard.
The default version of gcc with Raspbian Jan2021 is 8.3.0 and does not have all the necessary support.

Follow the instruction at: https://www.raspberrypi.org/forums/viewtopic.php?t=239609 to build and install the gcc 11.2.0 with minor modifications. See [build_gcc.sh](build_gcc.sh) for the script with only C/C++ languages enabled. Before running, be sure to review and update the config in [build_gcc.sh](build_gcc.sh) according to your device (default is Raspberry Pi 4).

Note the build needs about 8 GB of free disk space and takes about **4 hours** on the **Raspberry Pi 4 (4 GB)**, **over 6 hours** on the **Raspberry Pi 3B+** and **over 50 hours** on the **Raspberry Pi Zero**.

```
sudo chmod +x build_gcc.sh
sudo bash build_gcc.sh
```

If you follow the steps correctly, you should see the new version: 
```
g++ --version
```

Expected result:
```
g++ (GCC) 11.2.0
```

### 3.2 openssl

Starting with FreeFileSync 11.14, use of Openssl 3.0 is supported so build OpenSSL 3.0 and make it available to use.
```
wget https://www.openssl.org/source/openssl-3.0.0.tar.gz
tar xvf openssl-3.0.0.tar.gz
cd openssl-3.0.0
mkdir build
cd build
../config
make
sudo make install
export LD_LIBRARY_PATH=/usr/local/lib
```

### 3.3 libssh2
The system-provided version of libssh2 1.8.0-2.1 does not work with a macro not found compilation error:
```LIBSSH2_ERROR_CHANNEL_WINDOW_FULL```

Had to build from source:
```
wget https://www.libssh2.org/download/libssh2-1.10.0.tar.gz
tar xvf libssh2-1.10.0.tar.gz
cd libssh2-1.10.0/
mkdir build
cd build/
../configure
make
sudo make install
```

### 3.4 libcurl
Could not get any package from `apt-get` working so had to build curl and libcurl from source.
```
wget https://curl.haxx.se/download/curl-7.79.1.tar.gz
tar xvf curl-7.79.1.tar.gz
cd curl-7.79.1/
mkdir build
cd build/
../configure --with-openssl
make
sudo make install
```

### 3.5 wxWidgets
The latest version compiles without problem:
```
wget https://github.com/wxWidgets/wxWidgets/releases/download/v3.1.5/wxWidgets-3.1.5.tar.bz
tar xvf wxWidgets-3.1.5.tar.bz2
cd wxWidgets-3.1.5/
mkdir gtk-build
cd gtk-build/
../configure --disable-shared --enable-unicode
make
sudo make install
```

## 4. Tweak FreeFileSync code

Even with the latest dependencies, some code tweaks are needed.

### 4.1 FreeFileSync/Source/afs/sftp.cpp

Add these constant definitions starting at line 25
```
#define MAX_SFTP_OUTGOING_SIZE 30000
#define MAX_SFTP_READ_SIZE 30000
```

### 4.2 Update makefile to use GTK3 instead of GTK2 
As mentioned previously, use of GTK3 can result in poor UI experience, see thread at:
https://freefilesync.org/forum/viewtopic.php?t=7660

On line 19:
```
change: cxxFlags  += `pkg-config --cflags gtk+-2.0`
to:     cxxFlags  += `pkg-config --cflags gtk+-3.0`
```

On line 21:
```
change: cxxFlags  += -isystem/usr/include/gtk-2.0
to:     cxxFlags  += -isystem/usr/include/gtk-3.0
```

### 4.3 [Optional] FreeFileSync/Source/Makefile

To make the exectuable easier to run, add after line 28:
```
LINKFLAGS += -Wl,-rpath -Wl,\$$ORIGIN
```

## 5. Compile

Run ```make``` in folder FreeFileSync_11.14_Source/FreeFileSync/Source. 

Assuming the command completed without fatal errors, the binary should be waiting for you in FreeFileSync_11.14_Source/FreeFileSync/Build/Bin. 

## 6. Run FreeFileSync
Go to the FreeFileSync_11.14_Source/FreeFileSync/Build/Bin directory and enter:
```
./FreeFileSync_armv7l
```

# Running FreeFileSync on another Raspberry Pi {UNTESTED/UNVERIFIED for v11.14}
You don't need to build anything again on the other Raspberry Pi hosts but you will need to copy over the various libraries and other dependencies so the executable can run.
Compilation made and tested on Raspberry Pi 4 w/4GB RAM using clean updated Raspberry Pi OS on 22.11.2020

## 1. Create zip file with executable and all dependencies
Once the executable binary has been created and verified working, copy all dependency libraries to the same folder as the binary, then copy the `Build/Resources` folder, then zip them all up in a file.

The shared libraries that need to be copied are:
- libssl.so.3
- libstdc++.so.6
- libcurl.so.4
- libgcc_s.so.1
- libcrypto.so.3
- libssh2.so.1


Then end zip file should look like this:
```
Archive:  FreeFileSync_11.4_armv7l.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2020-04-27 22:38   Bin/
  9074216  2020-04-27 22:35   Bin/FreeFileSync_armv7l
   560412  2020-04-17 13:40   Bin/libssl.so.3
 17574572  2020-04-10 15:12   Bin/libstdc++.so.6
   492832  2020-04-17 14:14   Bin/libcurl.so.4
  7543332  2020-04-10 15:10   Bin/libgcc_s.so.1
  3476740  2020-04-17 13:40   Bin/libcrypto.so.3
   961484  2020-04-17 14:07   Bin/libssh2.so.1
      349  2020-03-18 21:57   FreeFileSync.desktop
        0  2020-03-18 21:57   Resources/
      234  2020-03-18 21:57   Resources/Gtk3Styles.css
    12402  2020-03-18 21:57   Resources/RealTimeSync.png
   182060  2020-03-18 21:57   Resources/harp.wav
    87678  2020-03-18 21:57   Resources/bell2.wav
    13232  2020-03-18 21:57   Resources/FreeFileSync.png
   223687  2020-03-18 21:57   Resources/cacert.pem
      440  2020-03-18 21:57   Resources/Gtk2Styles.rc
    67340  2020-03-18 21:57   Resources/ding.wav
   143370  2020-03-18 21:57   Resources/bell.wav
   230274  2020-03-18 21:57   Resources/gong.wav
   388959  2020-03-18 21:57   Resources/Icons.zip
   522150  2020-03-18 21:57   Resources/Languages.zip
    55006  2020-03-18 21:57   Resources/notify.wav
```

Now the zip file should contain all the dependencies and the binary `Bin/FreeFileSync_armv7l` is able to run on a new raspberry pi assuming the target is running a current verions of Raspbian.


## 2. On target host, copy and extract the zip archive you created
extract to:
```
/home/pi/Desktop/FFS_11.4_ARM/
```

## 3. Copy all lib* files to /usr/lib

```
sudo cp /home/pi/Desktop/FFS_11.4_ARM/Bin/lib* /usr/lib
sudo ldconfig
```

## 4. Run FreeFileSync
Go to /home/pi/Desktop/FFS_11.4_ARM/Bin/
```
./FreeFileSync_armv7l
```

# Troubleshooting & Known Issues

## Can't add Google Drive Connection
When attempting to add a Google Drive Connection, a tab in the chromium browser opens with an authentication error:
> Error 400: invalid_request
> 
> Missing required parameter: client_id

## Error due to use of gcc 9.3
> *../../zen/legacy_compiler.h:10:14: fatal error: numbers: No such file or directory 10 | #include <numbers> //C++20*
 
For FreeFileSync 10.25 and later, you need gcc 10.1 and using gcc 9.3 will give you the above erro
  

## Error due to missing shared libraries
> *./FreeFileSync_armv7l: error while loading shared libraries: libssl.so.3: cannot open shared object file: No such file or directory*

```
sudo cp /home/pi/Desktop/FFS_11.3_ARM/Bin/lib* /usr/lib
sudo ldconfig
```

## Other issues could exist
Other issues could certainly exist as overall usage of FreeFileSync on Raspberry Pi is presumably small. It seems likely there could be issues with the more demanding or complex use-cases.
