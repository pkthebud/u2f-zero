

U2F Zero
========

![](http://i.imgur.com/ZSk2AW3.jpg)

* [Overview](#overview)
* [Hardware](#hardware)
* [Firmware](#firmware)
  * [Program flow](#program-flow)
  * [Code organization](#code-organization)
* [Building one yourself](#build-a-u2f-zero-token-yourself)
  * [Parts](#parts)
  * [Flashing](#flashing)


Overview
=======

U2F Zero is an affordable and physically secure two factor authentication token that implements
the [U2F protocol](https://fidoalliance.org/specifications/overview/).  It's completely open source 
and designed for other people to be able to build easily.  It consists of a small PCB and 8 surface mount parts.

Hardware
========

The device uses Silicon Labs' [EFM8UB1 microcontroller](http://www.digikey.com/product-detail/en/silicon-labs/EFM8UB10F16G-C-QFN20/336-3410-5-ND/5592438)
to provide a USB interface and implement U2F.
It uses [Atmel's ATECC508A](http://www.digikey.com/product-detail/en/ATECC508A-MAHDA-T/ATECC508A-MAHDA-TCT-ND/5213071)
chip for true random number generation, hardware accelerated ECC key generation and signatures, atomic counters,
and tamper resistant storage for private keys.  EFM8UB1 and ATECC508A interface via a I2C command set.

The device also has an RGB LED for status indication and a button to receive user input.

USB pins are exposed copper zones on the PCB.  A 2mm thick PCB is recommended for best fit but 1.6 mm will work as well.

## Random number generation

The ATECC508A has a tamper resistent, [cryptographically secure 
random number generator](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator)
(CSPRNG) that implements [CTR_DRBG](https://en.wikipedia.org/wiki/NIST_SP_800-90A).
It's used internally for key generation and signing but it's also exposed to the user because
a good source of entropy can be useful.

Generate random numbers from the device:

```bash
cd tools/u2f_zero_client
./client.py rng     # output randomness at about 1400 bytes/s
```

Update the seed with user supplied data:

```bash
cd tools/u2f_zero_client
cat /dev/random | ./client.py seed     # update seed at about 410 bytes/s
```


Firmware
========

### Program flow

The program generally follows this execution flow:

Main loop:

* Check if USB is busy and schedule a read if USB is free
* If USB interrupted with a read, pass the newly read message to HID layer

HID layer:

* Read a HID packet passed to it
* Implement HID commands and sequencing as described in the
[U2F HID layer spec](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-u2f-hid-protocol.html)
* If the HID message contains a U2F packet, buffer it or pass complete U2F packet to the U2F layer

U2F layer:

* Read a U2F packet
* Implement authenticate and register commands as described in 
[U2F raw message spec](https://fidoalliance.org/specs/fido-u2f-v1.0-nfc-bt-amendment-20150514/fido-u2f-raw-message-formats.html)
* Handle any key generation, signatures, and atomic counting through I2C commands with ATECC508A

I2C layer:

* Receives a command and empty buffer for ATECC508A response
* Wake the ATECC508A from suspension
* Send the formatted command with CRC16.
* Receive ATECC508A response and check for errors and verify received CRC16.
* Put ATECC508A device into suspension
* I2C I/O and CRC calculations are interrupt based and done byte by byte.

### Code organization

The HID and U2F layers are written to not be device specific and can
easily be ported elsewhere.

EFM8UB1 USB driver:

* descriptors.c
* descriptors.h
* callback.c

EFM8UB1 I2C driver:

* Interrupts.c
* i2c.c
* i2c.h

ATECC508A I2C layer:

* atecc508a.c
* atecc508a.h

HID layer:
* u2f_hid.c
* u2f_hid.h

U2F layer:

* u2f.c
* u2f.h
* u2f-atecc.c   // device specific implementation


Build a U2F Zero token yourself
===============================

What's the point of an open source project if you can't build it yourself?

## Parts

You need these 8 surface mount parts which can all be purchased from Digikey (totals ~$3 per board):

* Microcontroller [EFM8UB10F16G-C-QFN20](http://www.digikey.com/product-detail/en/silicon-labs/EFM8UB10F16G-C-QFN20/336-3410-5-ND/5592438)
* Secure element [ATECC508A](http://www.digikey.com/product-detail/en/ATECC508A-MAHDA-T/ATECC508A-MAHDA-TCT-ND/5213071)
* [RGB LED](http://www.digikey.com/product-detail/en/LTST-C19HE1WT/160-2162-1-ND/4866310) for status indication.
* 100 Ohm current limiting [resistor](http://www.digikey.com/product-detail/en/CRCW0603100RFKEA/541-100HCT-ND/1179695).
* [Zener diode](http://www.digikey.com/product-detail/en/DF5A5.6JE,LM/DF5A5.6JELMCT-ND/5403466) for ESD protection.
* [Push button](http://www.digikey.com/product-detail/en/e-switch/TL3305AF260QG/EG5353CT-ND/5816198) for user input.
* [4.7 uF bypass capacitor](http://www.digikey.com/product-detail/en/CL10B475KQ8NQNC/1276-2087-1-ND/3890173) and [0.1 uF bypass capacitor](http://www.digikey.com/product-detail/en/CL05A104MP5NNNC/1276-1443-1-ND/3889529).

You need to get a PCB.  You can order a [pack of 10 from Dirty PCB's](http://dirtypcbs.com/view.php?share=18423&accesskey=4c0387e07ca9d1a2fdd583278a350fff)
or other PCB fab for about $14-20.  If you're
interested in doing this I would recommend getting a ten pack and then 10 of each part from Digikey for the 
price breaks.  It will be about $34 or $3.4 per U2F token.

You can check the Kicad schematic and layout in hardware/ for soldering information or follow this image:

![](http://i.imgur.com/0CIbSwG.jpg)

## Flashing

#### Prerequisites and dependencies

You need to install [Simplicity Studio](http://www.silabs.com/products/mcu/Pages/simplicity-studio.aspx)
to build the project.

You also need python and the python hidapi module for initial set up of device.

You can install hidapi with pip:

```bash
sudo pip install hidapi
```

You will need openssl installed and openssl libraries for creating and signing an attestation certificate.  
If you do not wish to do this, it is not needed.

You will also need a [programmer](http://www.digikey.com/catalog/en/partgroup/usb-debug-adapter-debugadptr1-usb/20059)
for Silicon Labs C2CK/C2D wire devices.

#### Opening the project

* Open Simplicity Studio
* Click File -> Import
* General -> Existing Projects into Workspace
* Select root directory and choose the `firmware/` directory
* Finish

#### Setup device

You will need to make two builds.  First build is the setup build.  It does not implement U2F
but rather just has everything to write the configuration for the ATECC508A and lock it.  It also
has to generate a key used for attestation and signing during U2F registrations.  You can ask me to
sign your key using the master U2F Zero signing key or you can sign it yourself.  Both will work.

First open "app.h" and uncomment "ATECC_SETUP_DEVICE".  Now build and program the device.

Now to check the device works, lock it, and get the public key used for attestation.

```bash
cd tools/u2f_zero_client
./client.py configure pubkey.hex
```

The ECC public key X,Y values will be stored in hex in pubkey.hex if setup is successful.

Now to create an attestation certificate from the public key.  You have a number of different options.
You can give the public key to me to create and sign it (with proof of U2F Zero key ownership), or you
can sign it yourself.

To sign it yourself:

```bash
# create your own "certificate authority" key signing pair and certificate

# generate EC private key
openssl ecparam -genkey -name prime256v1 -out key.pem

# generate a "signing request"
openssl req -new -key key.pem -out key.pem.csr

# self sign the request
openssl x509 -req -in key.pem.csr -signkey key.pem -out cert.pem
```

Now to sign the public key received from the device at setup.

```bash
pub=$(cat tools/hid_config/pubkey.hex)
cd tools/gencert

# edit signcert.c to add the certificates fields you want

make
./hex2pubkey $pub pubkey.pem

# sign public key and save certificate in DER
./signcert ca/key.pem cert.der
```

Now we are ready to make the final device build.  We will need to copy the DER certificate
into the build.

```
cd tools/

# convert bytes to C string
cat gencert/ca/cert.der | gencert/cbytes.py > pubkey.c.txt

# copy to source code
insert_key/insert.py ../firmware/src/u2f-atecc.c pubkey.c.txt ../firmware/src/u2f-atecc.c

# or you can copy it in manually
```

Uncomment "ATECC_SETUP_DEVICE" in `app.h`. Build and program the device.

#### Programming

You need three jumper cables and a Silicon Labs C2CK/C2D wire capable programmer.  Most development boards
or this [debugger](http://www.digikey.com/catalog/en/partgroup/usb-debug-adapter-debugadptr1-usb/20059) will work.

Connect the C2CK and C2D wires from programmer to the respective C2CK and C2D pins on the board.

<pic of programmer cable>
<pic of pics>

Connect ground of programmer cable to sit under the ground leg of the push button or other USB ground.

<pic of ground connection>

You should be able to detect the chip from Simplicity Studio and program it if everything is soldered correctly.

### U2F Zero "Installation"

It is an HID device so no drivers are needed.  It should work out of the box with OS X or Windows.  But it may only be accessible to root user at first on Linux.  Just add the following file `70-u2f.rules` to `/etc/udev/rules.d/` and add the following entry:

```
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="8acf", TAG+="uaccess"
```

Then restart udev

```bash
sudo udevadm trigger
```
