![Build Status](https://api.travis-ci.org/gl-sergei/u2f-token.svg?branch=master)

# U2F-TOKEN firmware for STM32F103 and EFM32HG boards

Turns your cheap STM32F103 or EFM32HG board into U2F token.

## Supported boards

U2F-TOKEN is known to work on:

- [Tomu board](https://tomu.im/) (EFM32HG) is an excellent device to use for U2F
  as it can sit entirely inside of your USB port (and it is opensource)
- Blue pill (STM32F103) as well as Black pill
- Countless ST32F103 based Chinese St-Link V2 clones can be turned into U2F
  devices with U2F-TOKEN
- Variety of Maple Mini clones which can be found on Aliexpress

## Building and flashing

### Requirements

#### Build tools

Install and setup Command Line tools for Xcode on macOS.

Install build-essentials package on Debian/Ubuntu:

``` sh
sudo apt install build-essential
```

#### GNU Toolchain for ARM Embedded Processors

Installing on macOS with homebrew:

``` sh
brew tap osx-cross/arm
brew install arm-gcc-bin
```

Installing on Debian/Ubuntu:

``` sh
sudo apt-add-repository ppa:team-gcc-arm-embedded/ppa
sudo apt update
sudo apt install gcc-arm-embedded
```
For Kali Linux:
``` sh
apt install gcc-arm-none-eabi
```


#### OpenSSL

MacOS comes with openssl/libressl installed out of the box.

Installing on Debian/Ubuntu:

``` sh
sudo apt install openssl
```

#### asn1crypto

There is a tiny python script used to convert certificates generated by OpenSSL
from DER format into C-array. It depends on asn1crypto package.

To install with pip:

``` sh
pip install --user --upgrade asn1crypto
```

#### OpenOCD

Installing on macOS with homebrew:

``` sh
brew install open-ocd
```

Installing on Debian/Ubuntu:

``` sh
sudo apt install openocd
```


### Building

``` sh
git clone https://github.com/gl-sergei/u2f-token.git
cd u2f-token
git submodule update --init
cd src
```

``` sh
make TARGET=<target>
```

will produce firmware file `build/u2f.bin`.

Supported targets are:

- [TOMU](http://tomu.im/)
- [MAPLE_MINI](https://wiki.stm32duino.com/index.php?title=Maple_Mini) 
- [BLUE_PILL](https://wiki.stm32duino.com/index.php?title=Blue_Pill)
- [BLACK_PILL](https://wiki.stm32duino.com/index.php?title=Black_Pill)
- [ST_DONGLE](https://wiki.stm32duino.com/index.php?title=ST-LINK_clone)

Use BLUE_PILL or BLACK_PILL for generic STM32F103 board without push button.

If build was unsuccessful for whatever reason and you want to start from scratch
you may want to run

``` sh
make clean
```

to remove all object files, or

``` sh
make certclean
```

to remove generated certificates, or

``` sh
make distclean
```

to remove `board.h` symlink, or even all of the above.


### Readout protection

Make sure to enable readout protection if you are going to use your device as
2FA for your accounts. Build firmware with `ENFORCE_DEBUG_LOCK=1`:

``` sh
make clean
make TARGET=<target> ENFORCE_DEBUG_LOCK=1
```

### Injecting private key

Firmware generates EC private key on its first boot and erases it when it
enters the bootloader. You may want to backup your private key and make it
survive firmware upgrade. To achieve this, generate the key on your host machine
and inject it into the firmware binary.

Generate your private key:

``` sh
openssl ecparam -name prime256v1 -genkey -noout -outform der -out key.der
openssl rand 16 > aes_key.dat
```

You may want to encrypt your `key.der` and `aes_key.dat` and back them up.

Check device's authentication counter if you are going to perform the firmware
upgrade. You can see it in Yubikey demo site output. For the new device, you can
skip `ctr` parameter all together or set it to 1. Let's say the current counter
value is 1000.

Use this command to patch firmware binary:

``` sh
./inject_key.py --key key.der --wrapping-key aes_key.dat --ctr 1001
```

### Flashing

#### To STMF103 board using ST-LINK/V2 and OpenOCD

Start OpenOCD:

``` sh
openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg
```

On other terminal run:

``` sh
telnet localhost 4444
> reset halt
> stm32f1x unlock 0
> reset halt
> program build/u2f.elf verify reset
> shutdown
```

#### To EFM32HG (Tomu) board using DFU

Providing you have Toboot installed:

``` sh
dfu-util -v -d 1209:70b1 -D build/u2f.bin
```


## Security considerations


### Random number generator (RNG)

U2F-TOKEN is using [Neug][neug] to generate high quality random numbers. The
source of entropy is built-in Analog to Digital Converter (both STM32F103 and
EFM32HG have ones). 32-bit ADC output is passed through CRC32 35 times to get
1120 bit input for SHA256-based whitening element.

[neug]: https://www.gniibe.org/memo/development/gnuk/rng/neug.html "Neug"

I ran [this suite][rngtest] on 1.7M of raw RNG output from Tomu board.

```text
SUMMARY
-------
monobit_test                             0.119269669219     PASS
frequency_within_block_test              0.11518538339      PASS
runs_test                                0.0626194973829    PASS
longest_run_ones_in_a_block_test         0.585067135452     PASS
binary_matrix_rank_test                  0.862015222632     PASS
dft_test                                 0.965404804209     PASS
non_overlapping_template_matching_test   1.00000631693      PASS
overlapping_template_matching_test       0.35076924588      PASS
maurers_universal_test                   0.999954925686     PASS
linear_complexity_test                   0.906328320146     PASS
serial_test                              0.159775233458     PASS
approximate_entropy_test                 0.15960828003      PASS
cumulative_sums_test                     0.0774471722878    PASS
random_excursion_test                    0.251774950817     PASS
random_excursion_variant_test            0.0871834280054    PASS
```

[rngtest]: https://github.com/dj-on-github/sp800_22_tests "SP800-22 Rev 1a PRNG test suite"


### Device Key

Device key is a private key for ECDSA p256r1. Any 256 bits are valid key. It is
generated using RNG on first run and stored in device's flash memory. It should
not leave the device. You must protect your device's flash from readout to
protect device key (see below).


### Key handles

U2F protocol specifies two actions - register and authenticate.

Register takes appID and challenge from the caller and returns key handle,
public key corresponding to that key handle and attestation certificate. No one
usually verify the validity of attestation certificate.

Authenticate takes the key handle, appID and another challenge, then it signs
both with the private key corresponding to the key handle. Caller then able to
check if signature has been made by the same device using a public key from
register step. Device in turn has to make sure to refuse signing requests for
unknown key handles and don't mess private keys corresponding to different key
handles.

If embedded devices would have unlimited storage, the best would be to store all
pairs of key handle and private key on the device. But it is not the case, so
the private key actually computed based on key handle and device key.

Here is how it is done by U2F-TOKEN firmware:


#### Register request (`app_id` is given):

1. Generate random `nonce` (32 bytes)
2. Compute `HMAC_SHA265(app_id + nonce)` using device key. Result becomes a
   private key (32 bytes)
3. Compute `HMAC_SHA256(private_k + app_id) + nonce`. Result becomes a key
   handle (64 bytes)
4. Compute public key for the private key (it's a nice feature of ECC that
   public key can be easily computed for given private key, but it is not that
   easy to do vice versa) and hand it out to the caller along with the key
   handle


#### Authenticate request (`app_id` and key handle are given):

1. Extract `nonce` (last 32 bytes of key handle)
2. Compute `HMAC_SHA256(app_id + nonce)` using device key. This is a private key
3. Compute `HMAC_SHA256(private_k)` and compare it to the first 32 bytes of key
   handle, they should match. If they don't, return key not found error.


Key handle should look random for casual observer and it does. I ran the same
suite on 200K of key handle data generated on Tomu board:

``` text
SUMMARY
-------
monobit_test                             0.228183190471     PASS
frequency_within_block_test              0.226128174333     PASS
runs_test                                0.0567490131255    PASS
longest_run_ones_in_a_block_test         0.748631279961     PASS
binary_matrix_rank_test                  0.612219673314     PASS
dft_test                                 0.553339041027     PASS
non_overlapping_template_matching_test   1.00000008312      PASS
overlapping_template_matching_test       0.932591232105     PASS
maurers_universal_test                   0.999315334632     PASS
linear_complexity_test                   0.335268833847     PASS
serial_test                              0.201711285468     PASS
approximate_entropy_test                 0.20129897411      PASS
cumulative_sums_test                     0.11790756896      PASS
random_excursion_test                    0.199118786038     PASS
random_excursion_variant_test            0.157445636324     PASS
```


### Tamper resistance

STM32F103 and EFM32HG devices are not tamper resistant. Highly skilled hacker
with the right equipment will be able to read the contents of device's flash and
get the device key. Then he will be able to clone your device and use it as
second factor to log into your account providing that he knows your passwords as
well.

However, they are resistant enough for daily use as *second factor*
authentication device, because:

1. Firmware does not allow to read flash contents by USB, neither does it disclose
   device key
2. Both STM32F103 and EFM32HG have "flash readout protection" feature. Once
   enabled, this feature protecting flash from being read via debug interface.
   See [this post][p1] for EFM32, and [this question][q1] for STM32F103. See also
   `efm32 debuglock` and `stm32f1x lock` commands in [OpenOCD manual][openocd-flash].

[p1]: http://community.silabs.com/t5/32-bit-MCU/Read-Write-Protection-of-Flash-and-SRAM/td-p/106405 "readout protection"
[q1]: https://stackoverflow.com/q/32509747 "readout protection"
[openocd-flash]: http://openocd.org/doc/html/Flash-Commands.html "readout protection"

## License

Chopstx is a threading library for Cortex-M0 and Cortex-M3 processors written by
Niibe Yutaka.

ECC is taken from Gnuk project by Niibe Yutaka.

Some constants and memory layout structures were taken from EFM32 platform
libraries by Silicon Laboratories.

Copyright &copy; 2017 Sergei Glushchenko

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

As additional permission under GNU GPL version 3 section 7, you may
distribute non-source form of the Program without the copy of the
GNU GPL normally required by section 4, provided you inform the
recipients of GNU GPL by a written offer.
