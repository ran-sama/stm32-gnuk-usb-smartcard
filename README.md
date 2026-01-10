# stm32-gnuk-usb-smartcard
The Gnuk smartcard based on the STM32F1x and Geehy APM32.  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/windows_test.png)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/gnuk_st_dongle_v2.png)  

## Table of contents
* [Introduction](#introduction)  
* [Dependencies](#dependencies)  
* [Building on *NIX](#building-on-nix)  
* [Pre-built binaries](#pre-built-binaries)  
* [Pin-out on the PCBs, always double-check with a multimeter](#pin-out-on-the-pcbs-always-double-check-with-a-multimeter)  
* [Flashing with OpenOCD via SWD on ST_DONGLE](#flashing-with-openocd-via-swd-on-st_dongle)  
* [Flashing with STM32 Flash loader via UART on BLUE_PILL](#flashing-with-stm32-flash-loader-via-uart-on-blue_pill)  
* [Factory resetting and loading it with your own keys](#factory-resetting-and-loading-it-with-your-own-keys)  
* [Gnuk 2.2+](#gnuk-22)  
* [Restoring from backup](#restoring-from-backup)
* [Update on SSH authentication](#update-on-ssh-authentication)  
* [Statistics on success rate, changes](#statistics-on-success-rate-changes)  
* [MegaHunt, APM32, STM32](#megahunt-apm32-stm32)
  
## Introduction

Gnuk is actively developed by Niibe Yutaka (新部裕) who is a Debian developer going by the nickname "gniibe".  
This repository focuses on Gnuk 1.2.19 and Gnuk 2.2+ largely for specific features and compatbility:
```
v1.2.19:
RSA-2048
RSA-4096**
EdDSA (Ed25519)
ECDSA (NIST P-256, secp256k1)
ECDH (X25519, NIST P-256, secp256k1)
** 8 sec per operation, RSA-4096 keys must be generated on host and exported to card

v2.2+:
Ed25519/X25519 is a bit faster with safegcd256
New: Ed448/X448 support
Removal of RSA support
Removal of NIST P-256 support
Removal of experimental features, including pinpad supoort, debug with CDC
Smaller code

(Allegedly also SECG/Koblitz and brainpool, but I could never bother to test.)
```

Secure mail exchange via GPG, code signing and verified Github commits are popular use cases of a hardware smartcard. Whilst modern Ed/cv25519 or classic RSA-2048 keys are both safe up at least until 2030, the support Curve448 and oldfashioned RSA-4096 ensures long-term safety and relevancy of this security device.  
  
For security reasons keys should always be generated on the host machine (Linux has the best CSRNG) and exported to Gnuk. Provisioning them safely into the Gnuk is explained in detail near the end of this repo, other supported versions of Gnuk are featured in the compile section as note.  
  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/raspi_test.png)  
Before you do anything: Remove the covers of the sticks, because their pin-out is often wrong. Check with your multimeter for voltages. The PCB is often correct but the cases are deceptive.  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/ali_haul.jpg)  

Above is a typical 10 buck haul from Ali with discounts on first purchased per store. To cover your losses you should at least source 1 device only per store. You will get scammed either way, but all of these worked. The actual issue at hand is soldering quality, residues and trace contaminants on the PCBs as well as missing solder on connectors. You get exactly the "quality" you paid for.

## Dependencies

For enabling the smartcard functionality of gpg:
```
sudo apt install scdaemon pcscd -y
```
Only in case of problems when accessing the smartcard these settings have worked on my various distros:  
```
# read the docs at:
# https://blog.apdu.fr/posts/2023/11/pcsc-lite-and-polkit/
$ sudo nano /etc/default/pcscd
# edit this line
PCSCD_ARGS="--disable-polkit"
# save & close
sudo systemctl restart pcscd.service
```
and
```
# read the docs at:
# https://blog.apdu.fr/posts/2024/12/gnupg-and-pcsc-conflicts-episode-3/
$ nano ~/.gnupg/scdaemon.conf
# add line
disable-ccid
# save & close
$ gpgconf --kill all
$ gpg --card-status
```
However if you don't have issues ```disable-ccid``` and ```PCSCD_ARGS="--disable-polkit"``` may be detrimental. Try what works best for you and restart the gpg-agent, or better yet your system, to be sure if you need one, both or none of these settings. As you see below, there are simply too many daemons and services involved to troubleshoot the smart card ecosystem easily.  
  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/data_flow_diagram.png)  
  
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
https://www.fsij.org/doc-gnuk/index.html
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

Other supported options for ```--target=``` are ```BLUE_PILL``` in case of the STM32 "bluepill" and ```ST_NUCLEO_F103``` for the official dev board.  

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

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/src_trg_new.png)
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

Now we flash in telnet, I recommend PuTTY on Windows and in this example we flash an ST-Link v2 clone to Gnuk with the ```.bin``` located in E:\ drive:
```
reset halt
stm32f1x unlock 0
reset halt
stm32f1x mass_erase 0
flash write_bank 0 E:\gnukst.bin 0
stm32f1x lock 0
reset halt
```
Sometimes read permissions are limited, or you could run openocd as admin. If there is an issue with having the file on C:\ you can put it on an external drive E:\ to have no issues.  
  
Here is a picture of one device only having 64k, which would also render it mostly useless as programmer:  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/bad_chip_new.png)  
On a real chip there are no issues:  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/good_chip_new.png)  
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
$ gpg --quick-generate-key 'Ran Yakumo (Github) <your@email.tld>' ed25519 default never
$ export KEYGR=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
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

sec   ed25519 2025-04-10 [SC]
      AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
      Keygrip = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
uid           [ultimate] Ran Yakumo (Github) <your@email.tld>
ssb   cv25519 2025-04-10 [E]
      Keygrip = BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
ssb   ed25519 2025-04-10 [A]
      Keygrip = CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC

$ gpg --expert --edit-key $KEYGR

gpg> toggle
gpg> keytocard

Really move the primary key? (y/N) y
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

gpg> key 1
gpg> keytocard

Please select where to store the key:
   (2) Encryption key
Your selection? 2

gpg> key 1
gpg> key 2

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3

gpg> key 2
gpg> save

$ gpg --card-status

Signature key ....: XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
      created ....: 2025-04-10 19:31:08
Encryption key....: XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
      created ....: 2025-04-10 19:31:32
Authentication key: XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
      created ....: 2025-04-10 19:31:42
General key info..: pub  ed25519/XXXXXXXXXXXXXXXX 2025-04-10 Ran Yakumo (Github) <your@email.tld>
sec>  ed25519/XXXXXXXXXXXXXXXX  created: 2025-04-10  expires: never
                                card-no: XXXX XXXXXXXX
ssb>  cv25519/XXXXXXXXXXXXXXXX  created: 2025-04-10  expires: never
                                card-no: XXXX XXXXXXXX
ssb>  ed25519/XXXXXXXXXXXXXXXX  created: 2025-04-10  expires: never
                                card-no: XXXX XXXXXXXX
```
After exporting the keys to the smartcard, local keys should only be stubs and not contain information. Back them up as well!
```
gpg --armor --output stubs.asc --export-secret-keys $KEYGR
```
Upload the contents of ```pubkey.asc``` to:  
```
https://github.com/settings/keys
```
Learn about how to configure git:
```
https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key
```
To use Gnuk with SSH:  
```
https://docs.nitrokey.com/nitrokeys/features/openpgp-card/ssh/
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
Optional step to change the pinentry prompt to your preferred one:  
```
sudo apt install pinentry-tty -y
sudo update-alternatives --config pinentry
# set to pinentry-tty and manual mode
```
All done!

## Gnuk 2.2+

As of writing I am using this branch:  
```
https://salsa.debian.org/gnuk-team/gnuk/gnuk/-/tree/438d89db8dd927ebaa4e93c2149f8ef9879168de
```
Which includes these changes over plain 2.2:

```
Fix tests/card_test_kg_pko_dsc_*.py
Fix Ed448 key import/generation
Add Ed448/X448 tests
Fix keygen tests for NIST P256 and Curve25519
Fix the name of brainpool test
Fix pk_25519_with_libgcrypt.py
Rename key files in tests
Fix a udev rule for stlink, since GROUP+MODE is Debian specific
Update documentation for Gnuk 2
```
Of course you could also just build directly from master:
```
git clone --recurse-submodules https://salsa.debian.org/gnuk-team/gnuk/gnuk --branch "master"
cd gnuk/src
export kdf_do=optional
./configure --enable-factory-reset --target=ST_DONGLE --vidpid=234b:0000 --enable-certdo
make
cp build/gnuk.bin /home/ran/gnukst22.bin
```
Remember, Gnuk 2.2+ is modern ECC based. Debian 12/13 (bookworm/trixie) currently only support GnuPG 2.2 with Ed/X25519, if you need Ed/X448 you should use *shudders* Ubuntu Noble for GnuPG 2.4 support.  
  
First we start with enabling KDF-DO feature, so that private keys remain safer in the MCU flash, this also works on previous Gnuk and gpg versions:
```
gpg --edit-card
gpg> admin
gpg> factory-reset
gpg> kdf-setup
gpg> yes
gpg> y
gpg> quit
```
Important:  
KDF-DO can only be enabled on factory reset cards. Passwords can only be changed on loaded/initialized cards. This is not very intuitive, but if you know manageable. The security of KDF-DO is very sound, you get 100k SHA256 iterations over your password and a random salt:
```
https://dev.gnupg.org/T3152#97063
```
Since we lock the flash banks of the ARM32 here after programming, read-protection is on. Some other commercial producat allegedly forgot to do this step once, but not our Gnuk, which is safe!  

```
https://github.com/rot42/gnuk-extractor#how-to-protect-a-gnuk-token
```
Any attack is non-trivial: This repo recommends to lock flash ROM and use KDF-DO. This means the AES encrypted database cannot be dumped anymore over the 4 programming pins on the PCB. A thief with physical access and perhaps voltage glitchers or EMF probes (quite costly) would have to brute-force over the entire 94chars^(len(pwd+salt))*100000iter of unbroken SHA256. It would probably take over a year.  
  
By that time you can just use the backup to sign a rollover to new keys and revoke the old keys with the backup cert. Only countries without human rights the sole out of scope threat.  
  
It should look like this, note the default keys are still Ed/X25519 and not yet Ed/X448:  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/refs/heads/master/images/KDF_on.png)

The process is almost the same as with the other curve:  
```
gpg --quick-generate-key 'Ran Yakumo (Github) <your@email.tld>' ed448 default never
export KEYGR=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
gpg --quick-add-key $KEYGR cv448 encr never
gpg --quick-add-key $KEYGR ed448 auth never
gpg --list-keys
```
I was going to check if it was already ultimately trusted (yes) and exported them for emergency backup:  
```
gpg --expert --edit-key $KEYGR
gpg --armor --output privkey.sec --export-secret-key $KEYGR
gpg --armor --output subkeys.sec --export-secret-subkeys $KEYGR
gpg --armor --output pubkey.asc --export $KEYGR
```
And now the interactive part with gpg again:  
```
gpg --expert --edit-key $KEYGR
gpg> keytocard
Really move the primary key? (y/N) y
gpg> key 1
sec
ssb*
ssb
gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2
gpg> key 1
sec
ssb
ssb
gpg> key 2
sec
ssb
ssb*
gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
gpg> key 2
sec
ssb
ssb
Note: the local copy of the secret key will only be deleted with "save".
gpg> save
```
Alright, they are shadowed and we backup the stubs too:  

```
gpg --armor --output stubs.asc --export-secret-keys $KEYGR
gpg --card-status
```
Now we see that there is the primary [SC] and two sperate [E] and [A] subkeys:  

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/refs/heads/master/images/EdX448.png)

If you don't need Gnuk 1.2.19 with RSA anymore, go directly for Gnuk 2.2+ and enjoy the new modern codebase.


## Restoring from backup

Very tired I tried to use my Gnuk and entered my password 3 times wrong, making it even worse during an unblock attempt. I was fully locked out. Using the ```gpg --edit-card```, ```admin``` and ```factory-reset``` commands I turned my Gnuk into a blank state.  
  
Reminder:  
KDF-DO can only be enabled on factory reset cards. Passwords can only be changed on loaded/initialized cards. This is not very intuitive, but if you know manageable.  
  
Make sure your backups are on an external device, such as mine are. Of course my frustration continued as I tried to import the backup again, be careful the following command empties your entire keyrings:  
```
$ rm -rf .gnupg
$ gpg --import-options restore --import privkey.sec
$ gpg --list-keys --keyid-format=long
$ export KEYGR=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
$ gpg --edit-key $KEYGR
# trust, select (5), quit
$ gpg --expert --edit-key $KEYGR
# repeat all toggle and keytocard steps from above
$ gpg --edit-card
$ gpg --card-status
$ echo "test" | gpg --clearsign
```
  
The external keys are as follows:
```
[SC] signature and certificate (also for revocation)
[E] encryption key
[A] authentication key
```
On the card they are as follows:
```
sec (primary key, same as [SC])
ssb (sub key 1, same as [E])
ssb (sub key 2, same as [A])
```
  
Also as before you can confirm if the process worked if you inspect the 3 private keys in the ```.gnupg``` directory. They will internally have the ```shadowed``` property as before. Proving they are stubs now and the real keys are only on the Gnuk. Everything was fixed!  
  
Testing if your backup actually works, even at least once, is a good way to ensure confidence in your tools.  

## Update on SSH authentication

Export your ssh public key (marked [A] for Authentication, usually the third one) with:  
```
gpg --card-status
gpg --export-ssh-key 26BA26D09DAC8601 > gnuk.pub
cat gnuk.pub
```
Which returns in my case:  
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAqfdLhCvwA6iTaQxJhdYbliEjtc33fyD+UxMgTwt4jV openpgp:0x9DAC8601
```
Copy and paste *your own* SSH key as you did with your GPG key already into the list of your keys:  
```
https://github.com/settings/keys
```
  
Store the ```gnuk.pub``` it in your ```.ssh``` dir.

Edit your git settings with ```nano .gitconfig``` which tells it to sign with Gnuk and authenticate with Gnuk on git push:  

```
[user]
	name = ran-sama
	email = ran@example.tld
	signingkey = 20EC6E3E6F2D9DB2
[commit]
	gpgsign = true
[credential]
	helper = store
	credentialStore = gpg
[gpg]
	program = gpg
[url "ssh://git@github.com/"]
	pushInsteadOf = https://github.com/
```
Do ```cd .ssh``` and edit ```nano config``` to:

```
Match host * exec "gpg-connect-agent UPDATESTARTUPTTY /bye"

Host github.com
        User git
        Hostname github.com
        IdentitiesOnly yes
        IdentityFile /home/ran/.ssh/gnuk.pub
        PubkeyAuthentication unbound
        KexAlgorithms -sntrup761x25519-sha512@openssh.com
```
If you are interested why a key exchange algorithm was disable please read:  
```https://dev.gnupg.org/T5931```

Your ```gpg-agent.conf``` in ```.gnupg``` should look like this:  
```
enable-ssh-support
pinentry-program /usr/bin/pinentry-tty
```

Also make sure that ```scdaemon.conf``` has this commented out:
```
#disable-ccid
```
As as of writing I don't use ```disable-ccid``` and ```PCSCD_ARGS="--disable-polkit"``` in ```/etc/default/pcscd``` anymore to make it work.  

In your home dir the ```.bashrc``` should contain these:  
```
export GPG_TTY=$(tty)
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
```
I even forgot to add this to ```.gnupg\gpg.conf```, so it may not be required:
```
use-agent
```
  
## Statistics on success rate, changes

Surprisingly most hardware I received was usable:  
```
Total chinese ST-Link-v2s: 22 pcs
Used as programmers: 2 pcs
Flash below 128k and useless: 1 pc
Failed during flashing: 2 pcs
Observed firmwares: V2J34S7/V2J37S7
```

The new programmer has a different pinout but works well too, I wrapped all devices in polyimide for electrical insulation as small improvement. It is thin, orange-transparent and heat resistant too and more for soldering than insulation. You can use regular electrician's tape:  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/polyimide_1.jpg)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/polyimide_2.jpg)  

## MegaHunt, APM32, STM32

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/chip_packages.jpg)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/terrible_okay_good.jpg)  
![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/R10_USB_DP_pullup.png)  
Picture 1-2 (Dongle): Common types of MCUs, soldering quality.  
Picture 3 (Bluepill): Wrong resistor value preventing USB from working, bypass by calculating parallel resistance to lower to 1.5k Ohm.  

## License
Licensed under the WTFPL license.
