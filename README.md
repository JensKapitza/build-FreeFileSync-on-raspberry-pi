# build-FreeFileSync-on-raspberry-pi
FreeFileSync is a great open source file synchronization tool but despite it being open source, build instructions were hard to find. 

This repo records a way of building FreeFileSync on a Raspberry Pi running Raspbian (Raspberry Pi OS) Aug2020. 

The applicable version involved are:
Rasbian (Raspberry Pi OS) release: ```August 2020```
FreeFileSync version : ```v11.3```

## 1. Download and extract the source code

As of this writing, the latest version of FreeFileSync is 11.3 and it can be downloaded from: 

https://freefilesync.org/download/FreeFileSync_11.3_Source.zip

In the past, wget **DID NOT** work with this URL on the first try so you had to either manually download it, or try wget a second time to get the source code downloaded. Seems the issue is resolved but worth noting incase you experience an issue.

## 2. Install dependencies
The following dependencies need to be installed to make code compile.
- libgtk2.0-dev
- libxtst-dev

```
sudo apt-get update
sudo apt-get install libgtk2.0-dev
sudo apt-get install libxtst-dev
```

## 3. Compile dependencies

The following dependencies could not be installed via `apt-get` and need to be compiled from source code.

### 3.1 gcc

FreeFileSync requires a c++ compiler that supports c++2a.
The default version of gcc with Raspbian Aug2020 is 8.3.0 and does not work.

Follow the instruction at: https://www.raspberrypi.org/forums/viewtopic.php?t=239609 to build and install the gcc 10.1.0 with minor modifications. See [build_gcc.sh](build_gcc.sh) for the script with only c/c++ languages enabled. But first change the config in [build_gcc.sh](build_gcc.sh) according to your device (default: Raspberry Pi 4).

Note the build needs about 8 GB of free disk space and takes about **4 hours** on the **Raspberry Pi 4 (4 GB)**, **over 6 hours** on the **Raspberry Pi 3B+** and **over 50 hours** on the **Raspberry Pi Zero**.

```
sudo chmod +x build_gcc.sh
sudo bash build_gcc.sh
```

If you follow the steps correctly, you should see the new verison of g++ using "g++ -v": 
```
g++ --version
```

Expected result:
```
g++ (GCC) 10.1.0
```

### 3.2 openssl

Starting with FreeFileSync 10.22, the openssl version needs to be `0x1010105fL` or above, otherwise you get an error:
```
../../zen/open_ssl.cpp:21:38: error: static assertion failed: OpenSSL version too old
   21 | static_assert(OPENSSL_VERSION_NUMBER >= 0x1010105fL, "OpenSSL version too old");
```

It would seem openssl-1.1.1f should work, but got another compilation error:
```
../../zen/open_ssl.cpp:576:68: error: 'SSL_R_UNEXPECTED_EOF_WHILE_READING' was not declared in this scope
  576 |             if (sslError == SSL_ERROR_SSL && ERR_GET_REASON(ec) == SSL_R_UNEXPECTED_EOF_WHILE_READING) //EOF: only expected for HTTP/1.0
```

So, used the latest openssl 3 from github ('alpha8' as of this writing)
```
git clone git://git.openssl.org/openssl.git --depth 1
cd openssl
./config
make
sudo make install
```
### 3.3 libssh2
The system-provided version 1.8.0-2.1 does not work with macro not found:
```LIBSSH2_ERROR_CHANNEL_WINDOW_FULL```

Had to build from source:
```
wget https://www.libssh2.org/download/libssh2-1.9.0.tar.gz
tar xvf libssh2-1.9.0.tar.gz
cd libssh2-1.9.0/
mkdir build
cd build/
../configure
make
sudo make install
```

### 3.4 libcurl
Could not get any package from `apt-get` working so had to build from source.
```
wget https://curl.haxx.se/download/curl-7.73.0.zip
unzip curl-7.73.0.zip
cd curl-7.73.0/
mkdir build
cd build/
../configure
make
sudo make install
```

### 3.5 wxWidgets
The latest version compiles without problem:
```
wget https://github.com/wxWidgets/wxWidgets/releases/download/v3.1.4/wxWidgets-3.1.4.tar.bz2
tar xvf wxWidgets-3.1.4.tar.bz2
cd wxWidgets-3.1.4/
mkdir gtk-build
cd gtk-build/
../configure --disable-shared --enable-unicode
make
sudo make install
```

## 4. Tweak the code

Even with the latest dependencies, there are still some compilation errors which are fixed with some code tweaks.

### 4.1 FreeFileSync/Source/afs/sftp.cpp

Add these constant definitions starting at line 58
```
#define MAX_SFTP_OUTGOING_SIZE 30000
#define MAX_SFTP_READ_SIZE 30000
```
### 4.2 zen/open_ssl.cpp
Change some function definitions to avoid compliation error with function not found:
```
Replace: using EvpToBioFunc = int (*)(BIO* bio, EVP_PKEY* evp);
With: using EvpToBioFunc = int (*)(BIO* bio, const EVP_PKEY* evp);

---
Replace: int PEM_write_bio_PrivateKey2(BIO* bio, EVP_PKEY* key)
With: int PEM_write_bio_PrivateKey2(BIO* bio, const EVP_PKEY* key)
```
### 4.3 wx+/dc.h
Change code to avoid compliation error with missed variable lines 71-73:

before
```
#ifndef wxHAVE_DPI_INDEPENDENT_PIXELS
#error why is wxHAVE_DPI_INDEPENDENT_PIXELS not defined?
#endif
```
after
```
/* #ifndef wxHAVE_DPI_INDEPENDENT_PIXELS
#error why is wxHAVE_DPI_INDEPENDENT_PIXELS not defined?
#endif */
```
Or just delete this 3 lines
### 4.4 [Optional] FreeFileSync/Source/Makefile
To make the exectuable easier to run, add after line 28:
```
LINKFLAGS += -Wl,-rpath -Wl,\$$ORIGIN
```

## 5. Compile

Run ```make``` in folder FreeFileSync_11.3_Source/FreeFileSync/Source. 

Assuming the command completed without fatal errors, the binary should be waiting for you in FreeFileSync_11.3_Source/FreeFileSync/Build/Bin. 

## 6. Run FreeFileSync
Go to the FreeFileSync_11.3_Source/FreeFileSync/Build/Bin directory and enter:
```
./FreeFileSync_armv7l
```

# Run FreeFileSync on another Raspberry Pi
You don't need to build anything again on the other Raspberry Pi hosts but you will need to copy over the various libraries and other dependencies so the executable can run.
Compilation made and tested on Raspberri Pi 4 RAM 4GB with clean updated Raspberry PI OS on 22.11.2020.

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
Archive:  FreeFileSync_11.3_armv7l.zip
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
/home/pi/Desktop/FFS_11.3_ARM/
```

## 3. Copy all lib* files to /usr/lib

```
sudo cp /home/pi/Desktop/FFS_11.3_ARM/Bin/lib* /usr/lib
sudo ldconfig
```

## 4. Run FreeFileSync
Go to /home/pi/Desktop/FFS_11.3_ARM/Bin/
```
./FreeFileSync_armv7l
```

# Troubleshooting & Known Issues
## Error due to use of gcc 9.3
> *../../zen/legacy_compiler.h:10:14: fatal error: numbers: No such file or directory 10 | #include <numbers> //C++20*

For FreeFileSync 10.25 you need gcc 10.1 and using gcc 9.3 will give you this error.

## Error due to missing shared libraries
> *./FreeFileSync_armv7l: error while loading shared libraries: libssl.so.3: cannot open shared object file: No such file or directory*

```
sudo cp /home/pi/Desktop/FFS_11.3_ARM/Bin/lib* /usr/lib
sudo ldconfig
```
## SSL Error during startup when checking if new version is available
When starting up, FreeFileSync checks if a newer version is available. For some reason (perhaps related to use of openssl 3?), there's a problem with the underlying SSL session and the version information can't be retrieved.
A popup dialog reads: "Cannot find current FreeFileSync version number online. A newer version is likely available. Check manually now?"
and the error detail provided reads:
```
Error code 167772454: error:0A000126:SSL routines::unexpected eof while reading [SSL_read_ex] SSL_ERROR_SSL
```
The problem doesn't seem to have any other side effects.

## Other issues could exist
Other issues could certainly exist as overall usage of FreeFileSync on Raspberry Pi is presumably small. It seems particularly possible there could be issues with the more demanding or complex use-cases.
