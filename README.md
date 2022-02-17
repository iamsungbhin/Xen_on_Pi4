The scripts in this repository provide an easy way to set up Xen on raspberry pi 4 with:   
* U-Boot as the bootloader,   
* Xen as the hypervisor,   
* Rpi Linux as dom0.

## References
* [Xen Wiki](https://wiki.xenproject.org/wiki/Main_Page)
* [imagebuilder](https://gitlab.com/ViryaOS/imagebuilder)
* [Xen on Raspberry Pi 4 Adventures](https://xenproject.org/2020/09/29/xen-on-raspberry-pi-4-adventures/)
* [TFTP -Community Help Wiki](https://help.ubuntu.com/community/TFTP)

## Prerequisite
```
$ sudo apt install vim git bc bison u-boot-tools gcc-aarch64-linux-gnu \
flex libssl-dev make libncurses5-dev -y
```

## 0. TFTP server 세팅& Imagebuilder 다운로드

- TFTP server 세팅

```bash
$ sudo apt install tftpd-hpa
$ sudo mkdir /tftpboot
$ sudo vi /etc/default/tftpd-hpa
```

- tftpd-hpa Configuration File 수정
    
    ```
    TFTP_DIRECTORY="/tftpboot"
    TFTP_OPTIONS="--secure --create"
    ```
    

```bash
$ sudo chown -R tftp /tftpboot
$ sudo service tftpd-hpa restart
```

---

- Imagebuilder

```bash
$ git clone https://gitlab.com/viryaOS/imagebuilder.git/
$ cd imagebuilder
$ mkdir rpi
```

## 1. 라즈베리 파이 OS sd카드 마운트

```bash
$ mkdir /mnt/boot /mnt/rootfs
$ lsblk # usb 연결 확인
$ sudo mount /dev/sdX1 /mnt/boot
$ sudo mount /dev/sdX2 /mnt/rootfs
$ sudo vi /mnt/boot/config.txt
```

- config.txt에 다음 문장 추가

```
kernel=u-boot.bin
enable_uart=1
arm_64bit=1
```

## 2. boot.scr 파일 생성

```bash
$ sudo vi boot.source
```

- 다음 문장 추가

```
setenv serverip 192.168.0.xxx  // TFTP 서버의 IP주소

setenv ipaddr 192.168.0.xxx    // 라즈베리 파이 IP 주소

tftpb 0xC00000 boot2.scr

source 0xC00000
```

- boot.scr 생성

```
$ mkimage -T script -A arm64 -C none -a 0x2400000 -e 0x2400000 -d boot.source boot.scr
$ sudo cp boot.scr /mnt/boot/
```

## 3. U-boot.bin 파일 생성

```bash
$ git clone git://git.denx.de/u-boot.git
$ cd u-boot
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-gnu-
$ make rpi_4_defconfig
$ make all
$ sudo cp u-boot.bin /mnt/boot/
```

## 4. Xen 파일 생성

※ Xen 버전과 Buildroot로 생성하는 Xen 패키지 버전 일치해야함

※ Xen 버전은 4.14.2 이상 필수

```bash
$ git clone -b RELEASE-4.14.3 git://xenbits.xen.org/xen.git
$ cd xen
$ make -C xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
$ make -C xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

- Xen config 세팅
    
    Debugging Options - - - >
    
    [ * ] Early printk (Early printk via 8250 UART) - - - >
    
    (0xfe215040) Early printk, physical base address of debug UART
    
    (2) Early printk, left-shift to apply to the register offsets within the 8250 UART
    

```bash
$ make dist-xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j4
$ sudo cp xen/xen ~/imagebuilder/rpi/
```

## 5. 리눅스 이미지 파일 생성

```bash
$ gie clone -b rpi-5.10.y https://github.com/raspberrypi/linux
$ cd linux
$ KERNEL=kernel8
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig xen.config
$ sudo vi .config
```

- 리눅스 config 파일 수정
    
    CONFIG_XEN_BLKDEV_FRONTEND=y
    
    CONFIG_XEN_BLKDEV_BACKEND=y
    
    CONFIG_XEN_DEV_EVTCHN=y
    
    CONFIG_XENFS=y
    
    CONFIG_XEN_GNTDEV=y
    
    CONFIG_XEN_GRANT_DEV_ALLOC=y
    

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image dtbs modules 
$ sudo cp arch/arm64/boot/Image ~/imagebuilder/rpi
$ sudo cp arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb ~/imagebuilder/rpi
```

## 6. rootfs.cpio 파일 생성

```bash
$ git clone https://git.buildroot.net/buildroot
$ cd buildroot
$ make raspberrypi4_64_defconfig
$ make menuconfig
```

* buildroot 세팅
    + Toolchain
        - C library -> glibc
        - Kernel Headers -> Linux 5.10.x kernel headers
        - GCC compiler Version -> gcc 10.x
    + System configuration
        - Init system -> systemd
        - /bin/sh -> bash
    + Kernel
        - Kerenl version -> Custom Git repository
        - URL of custom repository -> https://github.com/raspberrypi/linux
        - Custom repository version -> rpi-5.10.y
    + Target packages
        - System tools
          > -*- util-linux - - - >   
          > [ * ] basic set   
          > [ * ] xen   
          > [ ] Xen hypervisor   
          > [ * ] Xen tools
    + Filesystem Images
        - [ * ] cpio the root filesystem
        - [ * ] ext2/3/4 root filesystem
          > ext2/3/4 variant (ext4)
        - (rootfs) filesystem label
        - (200M) exact size
    
```bash
$ make -j4
. . .
$ cd ~/linux
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	INSTALL_MOD_PATH=/home/name/buildroot/output/target \
	modules_install
$ cd ~/buildroot
$ sudo rm -rf output/target/lib/modules/5.10.x-v8
$ make -j4
$ sudo cp output/images/rootfs.cpio.gz ~/imagebuilder/rpi
```

## 7. boot2.scr 생성

```bash
$ cd imagebuilder
$ sudo vi config
```

* config 파일 수정
  ```
  DEVICE_TREE="rpi/bcm2711-rpi-4-b.dtb"
  XEN="rpi/xen"
  XEN_CMD="console=dtuart dtuart=serial0 sync_console dom0_mem=1G dom0_max_vcpus=2”
  DOM0_KERNEL="rpi/Image"
  DOM0_RAMDISK="rpi/rootfs.cpio"
  NUM_DOMUS=0
  ```  

```bash
$ ./scripts/uboot-script-gen -c config -d . -t tftp -o boot2
$ sudo cp boot2.scr /tftpboot
$ sudo cp -r rpi /tftpboot/
```

## 8. 부팅 후 Xen 초기 테스트

```jsx
# xl list
# xl info
# ls -l /dev/xen
# xenstore-ls
# dmesg | grep Xen
# xl dmesg
```
