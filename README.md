# stm32-gnuk-usb-smartcard
The Gnuk smartcard based on the STM32F1x and Geehy APM32.

## Introduction

Secure mail exchange via GPG, code signing and verified Github commits are popular use cases of asymmetric cryptography utilizing cryptosystems such as RSA4096 or modern Ed/cv25519.

![alt text[]()](https://raw.githubusercontent.com/ran-sama/stm32-gnuk-usb-smartcard/master/images/ali_haul.jpg)


## Dependencies

For enabling the smartcard functionality of gpg:
```
sudo apt install scdaemon
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

## License
Licensed under the WTFPL license.
