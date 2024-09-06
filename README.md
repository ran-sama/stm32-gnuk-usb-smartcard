# stm32-gnuk-usb-smartcard
The Gnuk smartcard based on the STM32F1x and Geehy APM32.

## Introduction

Secure mail exchange via GPG, code signing and verified Github commits are popular use cases of asymmetric cryptography utilizing cryptosystems such as RSA4096 or modern Ed/cv25519.

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/windows_test.png)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/raspi_test.png)  
Before you do anything: Remove the covers of the sticks, because their pin-out is often wrong. Check with your multimeter for voltages. The PCB is often correct but the cases are deceptive.  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/ali_haul.jpg)  

Above is a typical 10 buck haul from Ali with discounts on first purchased per store. To cover your losses you should at least source 1 device only per store. You will get scammed either way, but all of these worked. The actual issue at hand is soldering quality, residues and trace contaminants on the PCBs as well as missing solder on connectors. You get exactly the "quality" you paid for.

## Dependencies

For enabling the smartcard functionality of gpg:
```
sudo apt install scdaemon
```

For building the Gnuk and uploading the .bin-artifact to STM32 on GNU/Linux:
```
sudo apt install cmake gcc-arm-none-eabi libnewlib-arm-none-eabi build-essential
sudo apt install openocd python3-usb
```

For Windows users there are confirmed working OpenOCD builds (2023-07-12):
```
https://gnutoolchains.com/arm-eabi/openocd/
https://web.archive.org/web/20240801214427/https://sysprogs.com/getfile/2060/openocd-20230712.7z
```

In case of missing drivers or libusb errors in OpenOCD (2.1 tested):
```
https://visualgdb.com/UsbDriverTool/
https://web.archive.org/web/20240801214520/https://sysprogs.com/getfile/1372/UsbDriverTool-2.1.exe
```

It doesn't matter what you use, each according to their comfort. What works for you, just works. I ran a hybrid of *NIX and Windows for the task, because there are too many tutorials covering monocultures of only one OS already.

## Building on *NIX

I will not lose many words about building, the src repo and devs website (NIIBE Yutaka) are here:
```
https://salsa.debian.org/gnuk-team/gnuk/gnuk
https://www.gniibe.org/tag/gnuk.html
```
For some reason, many professional Gnuk that are sold rebranded as Nitrokey Start et al. are using the 1.2.19 release, and ignore the 2.x branches as they may or may not be stable (yet), being ongoing in their development with exciting features such as Curve448. If you have enough boards it could be interesting though to explore.

As NIIBE Yutaka wrote in his post, I am not using ```--enable-hid-card-change``` as he reported that his users didn't seem to care for it. Also Nitrokey Start seems to have it commented out. You are now aware that you could use it, should you need it.

For the ST-Link-v2 rip-off from China we use:
```
git clone --recurse-submodules https://salsa.debian.org/gnuk-team/gnuk/gnuk --branch "release/1.2.19"
cd gnuk/src
export kdf_do=optional
./configure --enable-factory-reset --target=ST_DONGLE --vidpid=234b:0000 --enable-certdo
make
cp build/gnuk.bin /home/ran/gnukst.bin
```

For the STM32 "bluepill" the changes are trivial as expected:
```
git clone --recurse-submodules https://salsa.debian.org/gnuk-team/gnuk/gnuk --branch "release/1.2.19"
cd gnuk/src
export kdf_do=optional
./configure --enable-factory-reset --target=BLUE_PILL --vidpid=234b:0000 --enable-certdo
make
cp build/gnuk.bin /home/ran/gnukblue.bin
```

## Pre-built binaries
They are here for myself and archival reasons. I strongly advise you though to build your own: This is a security device and flashing binaries by random people of the internet is not encouraged. Though I promise they will work and were not tampered with, you cannot know that and being lazy is no excuse.

## Pin-out on the PCBs, always double-check with a multimeter

The good boards have them marked:
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/pinout_JTAG_orig.jpg)

The bad boards have microscopic solder beads at sub-millimetre size that are hard to connect to with DuPont connectors:
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/pinout_JTAG_clone.jpg)

As with the programmers, measure first and avoid surprises in polarity.


## Flashing with OpenOCD via SWD on ST_DONGLE

Running the JTAG protocol on top, the Serial Wire Debug (SWD) provides an elegant electrical alternative JTAG interface over 2-pins (SWDIO/SWCLK). The wire protocol is bi-directional and SWD is a recognized ARM CPU standard and defined in the ARM Debug Interface documentation.

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/SWD_JTAG.jpg)

Let's initialize a few files on the go first.  

Create a ```openocd.cfg```:
```
telnet_port 4444
source [find interface/stlink-v2.cfg]
source [find target/stm32f1x.cfg]
set WORKAREASIZE 0x10000
```
Also create a ```openocd2.cfg``` for Chinese clones:
```
telnet_port 4444
source [find interface/stlink-v2.cfg]
source [find target/apm32f1x.cfg]
set WORKAREASIZE 0x10000
```

Create a copy of ```stm32f1x.cfg``` and call it ```apm32f1x.cfg```, then change the value in line 44 defining the ```_CPUTAPID```:
```
   } {
      # this is the SW-DP tap id not the jtag tap id
      set _CPUTAPID 0x1ba01477
   }
}
```
Into the value for the Chinese Geehy APM32 arm clone, should you need it, you have it ready:
```
   } {
      # this is the SW-DP tap id not the jtag tap id
      set _CPUTAPID 0x2ba01477
   }
}
```
Now we both support STM32 and APM32 and need not to care what they shipped us!

Launch OpenOCD on *NIX or Windows, it both just works:
```
# for real STM32
openocd -f openocd.cfg
# for the APM32 clones:
openocd -f openocd2.cfg
```
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/OpenOCD_cli.jpg)
Terrible camera pic, was lazy. If your pins make correct contact you should be greeted by a welcoming output and 3 ports are allocated for your pleasure.

Now we flash in telnet, I recommend PuTTY on Windows and in this example we flash an ST-Link v2 clone to Gnuk with the .bin located in C:\ drive:
```
reset halt
stm32f1x unlock 0
reset halt
stm32f1x mass_erase 0
flash write_bank 0 C:\gnukst.bin 0
stm32f1x lock 0
reset halt
```
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/telnet_console.png)
The telnet console in putty for ```localhost 4444```.

It will now be ready to be plugged into USB on its own without your JTAG interface! It works out of the box on both Windows and *NIX as card reader and smartcard in one device.  

## Flashing with STM32 Flash loader via UART on BLUE_PILL

I am sorry, but I couldn't bother. Use a CH34x or CP210x USB to UART adapter and the v2.8.0 of:
```
https://www.st.com/en/development-tools/flasher-stm32.html#get-software
```
Set the jumpers for flashing to:
```
BOOT0=1
BOOT1=0
```
Set both to OFF afterwards successfully flashing.

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/lazy_UART_bluepill.jpg)

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/blue_flash_tool.png)

Observe that 128KB was selected, though these only "officially" support 64KB. If the upload was successfuly you got a good batch. Many companies don't truly care to have different line-ups and just pretend there is a real difference. We can totally live with that.

## Factory resetting and loading it with your own keys

A mandatory step and a lesson left to the inclined reader to figure out, though a starting point is provided hereby:
```
gpg --card-status
gpg --edit-card
gpg/card> admin
gpg/card> factory-reset
gpg/card> help
gpg/card> passwd
```
Default PINs:
```
User=123456
Admin=12345678
```
Easy guide, but RSA only:
```
https://raymii.org/s/articles/Nitrokey_Start_Getting_started_guide.html
```
My personal method for EC keys only, because RSA sucks:
```
$ gpg --quick-generate-key 'Ran Yakumo (Github) <your@email.tld>' ed25519 cert never
$ export KEYGR=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
$ gpg --quick-add-key $KEYGR ed25519 sign never
$ gpg --quick-add-key $KEYGR cv25519 encr never
$ gpg --quick-add-key $KEYGR ed25519 auth never
```
It is essential to back-up your keys before exporting them to the smartcard:
```
$ gpg --armor --output privkey.sec --export-secret-key $KEYGR
$ gpg --armor --output subkeys.sec --export-secret-subkeys $KEYGR
$ gpg --armor --output pubkey.asc --export $KEYGR
```
Export to smartcard:
```
$ gpg -K --with-keygrip
/home/ran/.gnupg/pubring.kbx
----------------------------
sec>  ed25519 2024-08-02 [C]
      AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
      Keygrip = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
      Card serial no. = XXXX XXXXXXXX
uid           [ultimate] Ran Yakumo (Github) <your@email.tld>
ssb>  ed25519 2024-08-02 [S]
      Keygrip = BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
ssb>  cv25519 2024-08-02 [E]
      Keygrip = CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
ssb>  ed25519 2024-08-02 [A]
      Keygrip = DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD
$ gpg --expert --edit-key $KEYGR
gpg> toggle
gpg> keytocard
Really move the primary key? (y/N) y
Please select where to store the key:
   (1) Signature key
Your selection? 1
gpg> save
$ gpg --expert --edit-key $KEYGR
gpg> toggle
gpg> key 1
gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
gpg> key 1
gpg> key 2
gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2
gpg> key 2
gpg> key 3
gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
gpg> save
$ gpg --edit-card
General key info..:
sub  ed25519/XXXXXXXXXXXXXXXX 2024-08-02 Ran Yakumo (Github) <your@email.tld>
sec>  ed25519/XXXXXXXXXXXXXXXX  created: 2024-08-02  expires: never
                                card-no: XXXX XXXXXXXX
ssb>  ed25519/XXXXXXXXXXXXXXXX  created: 2024-08-02  expires: never
                                card-no: XXXX XXXXXXXX
ssb>  cv25519/XXXXXXXXXXXXXXXX  created: 2024-08-02  expires: never
                                card-no: XXXX XXXXXXXX
ssb>  ed25519/XXXXXXXXXXXXXXXX  created: 2024-08-02  expires: never
                                card-no: XXXX XXXXXXXX

gpg/card> quit
```
After exporting the keys to the smartcard, local keys should only be stubs and not contain information. Back them up as well!
```
gpg --armor --output stubs.asc --export-secret-keys $KEYGR
```
Upload the contents of ```pubkey.asc``` to:  
```
https://github.com/settings/keys
```
Test your smartcard:
```
$ export GPG_TTY=$(tty)
$ echo "test" | gpg --clearsign
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

test
Please unlock the card

Number: XXXX XXXXXXXX
Holder: Ran Yakumo
Counter: 0
PIN:
-----BEGIN PGP SIGNATURE-----

iHUEARYIAB0WIQTg7M4rUe6PaMpPs99tMe7Ji+duHAUCZrupeQAKCRBtMe7Ji+du
HMcTAQCgUMWtTo9/+1tmukVwjYeADdBw9+0/27YzOlzX+o+trAEAnzyzL4RC1D8U
H7q71kd0ABWysFbnIkXVpx02jH7pHgQ=
=F9X7
-----END PGP SIGNATURE-----
```
Add the ```export GPG_TTY=$(tty)``` to your ```.bashrc``` and/or ```.profile```
```
git config --global gpg.program gpg
git config --global user.signingkey $KEYGR
git config --global commit.gpgsign true
```
All done!

## Unknown, APM32, STM32

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/chip_packages.jpg)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/terrible_okay_good.jpg)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/R10_USB_DP_pullup.png)  
Picture 1-2 (Dongle): Common types of MCUs, soldering quality.  
Picture 3 (Bluepill): Wrong resistor value preventing USB from working, bypass by calculating parallel resistance to lower to 1.5k Ohm.  

## License
Licensed under the WTFPL license.
