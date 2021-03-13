# Ethernet over AX.25

The eoax Linux kernel module creates a virtual Ethernet interface for every AX.25 one available. It encapsulates Ethernet frames in AX.25 UI frames, using Protocol Identifier (PID) 0x0D. It does the reverse job of the [BPQ Ethernet](http://www.linux-ax25.org/wiki/BPQ) module. The code can also serve as a basis for experimenting with new routing protocols for packet radio in Linux.

## Motivation and design

## Status

## Installation

The eoax module requires some changes to the kernel AX.25 layer in order to compile and run. These changes affect the way AX.25 handles outgoing and incoming UI frames with unknown PIDs, and are included as a patch to kernel sources.

The following is what I do to test the software. It is by no means a "proper" installation method, but rather a quick-and-dirty way to build and run the kernel module. Use at your own risk. The steps have been tested on Ubuntu 14.04.3 LTS and 15.10 and may require changes for other Linux distributions.

**Note that as time goes on the included kernel patches may need to be updated**

* Define environment variables:
  ```bash
  EOAX_SRC="~/eoax"
  KERNEL_SRC_DIR="~/RPiKernel"
  KERNEL_SRC="$KERNEL_SRC_DIR"/linux
  ```

* Download and install kernel build tools and sources:

  ```bash
  sudo apt update
  sudo apt install git bc bison flex libssl-dev
  sudo apt install libncurses5-dev
  sudo wget https://raw.githubusercontent.com/RPi-Distro/rpi-source/master/rpi-source -O /usr/local/bin/rpi-source && sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update
  ```
  It would be a good idea to reboot (especially if you decide to run `apt upgrade`) to ensure you are building for the same kernel as the one running
  ```bash
  sudo reboot
  ```
  Download the Kernel:
  ```bash
  mkdir -p "$KERNEL_SRC_DIR"
  rpi-source -d "$KERNEL_SRC_DIR"
  ```

* Download [eoax](https://github.com/Doom4535/eoax) if not already downloaded:
  ```bash
  [ ! -d "$KERNEL_SRC" ] && git clone git://github.com/Doom4535/eoax.git
  ```

* Apply the AX.25 patch (you may need to update this if there are patching issues):
  ```bash
  cd "$KERNEL_SRC"
  patch -p0 < "$EOAX_SRC"/patches/ax25_ui_type-4.2
  ```

* Compile and install the patched AX.25 module (we will build the entire kernel to obtain System.map, but only use the module):
  ```bash
  cd "$KERNEL_SRC"
  make oldconfig && make prepare
  make -j$(nproc)
  cd "$KERNEL_SRC"/net/ax25/
  make -j$(nproc) -C /lib/modules/$(uname -r)/build M=$(pwd) modules
  sudo make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install
  ```
  
* Instruct modprobe to use the patched version of AX.25:
  ```bash
  sudo mkdir /etc/depmod.d
  echo "override ax25 * extra" | sudo tee -a /etc/depmod.d/raspbian.conf
  sudo depmod
  ```

* Copy the new Module.symvers for AX.25 over to eoax sources, to get the correct symbol versions:
  ```bash
  cp Module.symvers "$EOAX_SRC"/drivers/net/hamradio/
  ```

* Compile and install the eoax module:
  ```bash
  cd "$EOAX_SRC"
  make -j$(nproc)
  sudo make install
  sudo depmod
  ```

## Usage

Load the module with `modprobe eoax`. If you have an AX.25 interface configured, you should now see an `eoax` one. If you don't, the `eoax` interface will show up, as soon as an AX.25 interface is configured. Look at `dmesg` to find out which `eoax` interface corresponds to which AX.25 one.
