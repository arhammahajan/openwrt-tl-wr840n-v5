# Building a lean version of OpenWrt for the TP-Link TL-WR840n with ipv6 and WPA3-SAE support

## Introduction
The last officially supported version of OpenWrt for this device was 18.06.9. The problem is the limited flash (4MB) and ram (64MB) for this device. So while the image works well on the device, there is no flash space left for the overlay partition (used to store configuration), and hence it mounts on the ram. So, on power cycling the device any configuration changes are wiped off. You could get around this limitation in one of two ways. The first is, removing unncecessary packages while building to free up space. I was able to free up enough space to get the overlay partition to mount on one of the storage blocks. Else, you could simply change the config files according to your setup and include them in the build process. This way even if there is no space left for the overlay partition, but when the device power cycles, it would fall back to the default modified config which you had embedded into the firmware itself.

However, with a few tweaks it is also possible to build 19.07.10 for this device. While the image is generally stable, it comes with its own caveats. To get the image to build for the target (tl-wr840n-v5), I had to remove luci, pppoe support, dhcp server, dnsmasq, opkg among a few others. Since, I am using this only as an access point the dns and dhcp servers were of no use in my config. The lack of the luci web interface does make the device very hard to configure (editing the conf files through ssh being the only way). The removal of opkg also means that there is no package manager to install new packages. Again this should not be an issue, because even in the stripped down state, there is not much headroom to install additional packages. I have also replaced the default wpad-basic package with the wpad-wolfssl package to get WPA3-SAE support on my device. This change however has resulted in there being no space left for the overlay partition. That too is not much of an issue for me as I build the image with my own custom config. So even when the device reboots, it falls back to my desire config instead of the default one.

## Setting up a VM
I would not suggest anyone to do this process on their host machine. Instead spin up a Debian/Ubuntu virtual machine to have a clean and isolated build environment. You could get by just fine with 4vCPUs, 2GB ram, and 20GB of hard disk space.

You could refer to this guide by OpenWrt for a simplified virtual machine setup: [https://openwrt.org/docs/guide-developer/buildserver_virtualbox]()

## Pre-requisites and dependencies
For this section, my recommendation is to refer to the OpenWrt official guide as that is quite informative in and of itself.
[https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem]()

The only problem however, is that building this version of OpenWrt requires python2.7. In newer versions of certain OS' that may not exisite in package manager repositories. So you will have to build that from source too. Since I was using Ubuntu 24, I referred to this guide: [https://github.com/ctch3ng/Installing-Python-2.7-and-pip-on-Ubuntu-24.04-Noble-LTS](). While the general process remains the same, you could adapt it for the os you are using as the build environment.

## Using the Image Builder
Keep in mind, that we are not building these images from source. Instead, we are using the imagebuilder package supplied by OpenWrt to bundle the linux base with packages and config of our own choice. The process to build images from source, is different from this point on. Refer to the official OpenWrt guide for the same.

### Step-1
Firstly, you would need to obtain the imagebuilder package for the router. This device is based on the `mt7628` mips based soc. 
```sh
wget https://archive.openwrt.org/releases/19.07.10/targets/ramips/mt76x8/openwrt-imagebuilder-19.07.10-ramips-mt76x8.Linux-x86_64.tar.xz
```
### Step-2
```sh
tar -xf openwrt-imagebuilder-19.07.10-ramips-mt76x8.Linux-x86_64.tar.xz && cd openwrt-imagebuilder-19.07.10-ramips-mt76x8.Linux-x86_64.tar.xz
```

### Step-3
Now we need to add the custom config which will be used during building. First, we need to obtain the default config. For this, I would suggest flashing the device with the 18.06.9 firmware and then obtaining the config. Once obtained, you could make the necessary changes by referring to the docs. Alternatively, you could also make the changes in the luci web interface and then copy over the config, however this method won't work if you remove the luci package from the firmware altogether. 
```sh
mkdir files && cd files
```
```sh
scp -O -oHostKeyAlgorithms=+ssh-rsa -r root:192.168.1.1:/etc .
```
The firmware is available in [https://archive.openwrt.org/releases/18.06.9/targets/ramips/mt76x8/]()

### Step-4
Now, we can actually build the image. The below command has been adapted to my configuration and will build a very stripped down version of OpenWrt. If you are unsure of anything, please refer to the official guide. [https://openwrt.org/docs/guide-user/additional-software/imagebuilder]()
```sh
make FORCE=1 image PROFILE="tl-wr840n-v5" PACKAGES="-wpad-basic -luci-ssl -ppp -ppp-mod-pppoe -dnsmasq -odhcpd -odhcpd-ipv6only -opkg -luci-app-firewall -luci-proto-ppp -luci-app-opkg -uclient-fetch -usign -jshn -jsonfilter -kmod-ipt-offload -kmod-nf-flow -kmod-nf-nat -kmod-nft-nat -kmod-ipt-nat wpad-wolfssl" FILES="files"
```

If after running the above command, you still get an error regarding missing dependencies, given that you had followed the above steps correctly, you should run the following, in the root directory of the imagebuilder. Post this, the build should start.
```sh
mkdir -p staging_dir/host && touch staging_dir/host/.prereq-build
```

If the build process exits with an error specifying 'images too big' it means that the selected configuration of packages is making the image bigger than physially possible for the device. In such a case you would have to remove some packages and try again. Finally the image is placed in `./bin/targets/ramips/mt76x8`.

### Step-5
Now we need to flash the image onto the device. You could either use the luci web interface for that if that is an option, or you could flash it through the command line. First copy over the image onto the /tmp folder of the router, then ssh into it and finally apply the upgrade. Post this, the router shall restart.
```sh
scp -O -oHostKeyAlgorithms=+ssh-rsa ./openwrt-19.07.10-ramips-mt76x8-tl-wr840n-v5-squashfs-sysupgrade.bin root@192.168.1.1:/tmp
ssh -oHostKeyAlgorithms=+ssh-rsa root@192.168.1.1
sysupgrade /tmp/openwrt-19.07.10-ramips-mt76x8-tl-wr840n-v5-squashfs-sysupgrade.bin 
```

### Additional notes
1. in my current setup, since the overlay partition is living on the ram, I rebuild the image everytime I make config changes and then apply that to the router.
2. do not flash the image without the dhcp server unless you have already configured the lan interface. if you do so, the router will not be able to assign ip addresses to your devices. to overcome this, you will need to obtain the ip manually
3. i tried flashing higher versions of openwrt, but with each new revision, the kernel gets bigger, making it even harder to get the image to build within size limits. imo, 19.07.10 is the last version, that can be practially used on this device
4. for anyone willing, you could swap out the existing flash chip for a W25Q128 (16mb flash). this would significantly improve the usability. however, the setup becomes even more challenging requiring modifications to the bootloader and image headers.
5. you could also use hostapd-wolfssl instead of wpad-wolfssl, as its a lighter package. But for 802.11r fast roaming support on SAE, i would suggest to go ahead with the wpad-wolfssl package only
6. v4 and v5 versions of this device are the best candidates. with the v6, the ram has been further reduced. for anyone interested, i have added links for unofficial v6 support as well in the references
7. in case you brick your device (flashing red led), you will need to restore it via tftp
8. you can ssh into router by `ssh -oHostKeyAlgorithms=+ssh-rsa root@192.168.1.1`

## References
[https://community.onion.io/topic/4992/official-openwrt-package-repos-are-no-longer-valid]()
[https://openwrt.org/docs/guide-user/additional-software/imagebuilder]()
[https://github.com/ctch3ng/Installing-Python-2.7-and-pip-on-Ubuntu-24.04-Noble-LTS]()
[https://github.com/shivammaggu/TP-Link-TL-WR840N-V5-OpenWRT]()
[https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem]()
[https://www.home-assistant.io/integrations/luci/]()
[https://archive.openwrt.org/releases/]()
[https://robu.in/product/w25q128-large-capacity-flash-storage-module/]()
[https://github.com/IcedShake/openwrt-19.07-tl-wr840n-v6.x]()
[https://github.com/geekobiloba/tl-wr840nv6.x_openwrt_builder]()
[https://github.com/hoppingninja666/openwrt-wr840n-v620]()
[https://openwrt.org/toh/hwdata/tp-link/tp-link_tl-wr840n_v6]()
[https://openwrt.org/toh/hwdata/tp-link/tp-link_tl-wr840n_v5]()
[https://www.tp-link.com/nordic/support/download/tl-wr840n/v5/]()
[https://static.tp-link.com/upload/firmware/2021/202111/20211112/TL-WR840N(EU)_V5_211109.zip]()