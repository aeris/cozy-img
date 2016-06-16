DEBIAN_REPOSITORY := http://localhost:3142/http.debian.net/debian/
DEBIAN_RELEASE := jessie
SDCARD_DEV := /dev/sdd
MAKE_OPTIONS := -j8
IMG_SIZE ?= 2G
RSYNC := rsync -ahxAHPX --numeric-ids --inplace

.DEFAULT_GOAL := xz
MAKEFLAGS += --no-builtin-rules
unexport MAKEFLAGS

CROSS_COMPILE_ARMHF := ARCH=arm CROSS_COMPILE="ccache arm-linux-gnueabihf-"
CROSS_COMPILE_ARM64 := ARCH=arm64 CROSS_COMPILE="ccache aarch64-linux-gnu-"
COMPILE				:= CC="ccache gcc" CXX="ccache g++"

SUNXI_PACK_TOOLS := build/sunxi-pack-tools/bin

.PHONY: build chroot clean desync dev distclean flash format img img_compress img_flash linux linux_flash mr-proper overlay_sync resources rootfs_flash rootfs_sync sync tor_ipset tor_keyring uboot uboot_flash overlay

sync:
	$(RSYNC) --delete . cozy-dev.local:img/pine64/ --exclude-from .exclude
desync: sync
	$(RSYNC) --delete cozy-dev.local:img/pine64/ .

dev:
	echo "APT::Install-Recommends \"0\";\nAPT::Install-Suggests \"0\";" > /etc/apt/apt.conf.d/60recommends
	apt install -y apt-cacher-ng
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > /etc/apt/apt.conf.d/00proxy

	apt install -y build-essential pkg-config libqt4-dev libncurses5-dev u-boot-tools device-tree-compiler build-essential wget qemu-user-static debootstrap binfmt-support rsync ccache curl apt-cacher-ng parted secure-delete pv pxz dos2unix mtools bc dosfstools git-core kpartx

	echo "deb http://emdebian.org/tools/debian/ jessie main" > /etc/apt/sources.list.d/embedian.list
	curl http://emdebian.org/tools/debian/emdebian-toolchain-archive.key | apt-key add -
	dpkg --add-architecture armhf
	dpkg --add-architecture arm64
	apt update
	apt install -y crossbuild-essential-armhf crossbuild-essential-arm64

	$(MAKE) resources

resources/ build/ build/sd/boot/ build/sd/root/:
	mkdir -p $@

resources/sys_config.fex: | resources/
	wget https://github.com/longsleep/build-pine64-image/raw/master/blobs/sys_config.fex -O $@
	unix2dos $@
resources/scp.bin: | resources/
	wget https://github.com/longsleep/build-pine64-image/raw/master/blobs/scp.bin -O $@
resources/boot0.bin: | resources/
	wget https://github.com/longsleep/build-pine64-image/raw/master/blobs/boot0.bin -O $@
resources: resources/sys_config.fex resources/scp.bin resources/boot0.bin \
	resources/u-boot/ resources/arm-trusted-firmware/ resources/sunxi-pack-tools/ resources/linux/

resources/linux/: | resources/
	git clone --bare --branch a64-v5 --single-branch --depth 1 https://github.com/apritzel/linux.git $@
build/linux/: | resources/linux/ build/
	git clone resources/linux/ $@
build/linux/.config: | build/linux/
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARM64) -C build/linux/ defconfig
build/linux/arch/arm64/boot/Image: build/linux/.config
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARM64) -C build/linux/ Image modules
kernel-image: build/linux/arch/arm64/boot/Image
build/linux/arch/arm64/boot/dts/allwinner/sun50i-a64-pine64.dtb: build/linux/.config
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARM64) -C build/linux/ dtbs
kernel-dtbs: build/linux/arch/arm64/boot/dts/allwinner/sun50i-a64-pine64.dtb
kernel: kernel-image kernel-dtbs

resources/sunxi-pack-tools/: | resources/
	git clone --bare --single-branch --depth 1 https://github.com/longsleep/sunxi-pack-tools.git $@
build/sunxi-pack-tools/: | resources/sunxi-pack-tools/
	git clone resources/sunxi-pack-tools/ $@
$(SUNXI_PACK_TOOLS)/script $(SUNXI_PACK_TOOLS)/merge_uboot \
$(SUNXI_PACK_TOOLS)/update_uboot $(SUNXI_PACK_TOOLS)/update_uboot_fdt: | build/sunxi-pack-tools/
	$(MAKE) $(MAKE_OPTIONS) $(COMPILE) -C build/sunxi-pack-tools/
sunxi-pack-tools: build/sunxi-pack-tools/bin/update_uboot

build/sys_config.fex: resources/sys_config.fex
	cp $< $@
build/sys_config.bin: $(SUNXI_PACK_TOOLS)/script build/sys_config.fex
	$< $(word 2, $^)

resources/arm-trusted-firmware/: | resources/
	git clone --bare --branch allwinner --single-branch --depth 1 https://github.com/apritzel/arm-trusted-firmware.git $@
build/arm-trusted-firmware/: | resources/arm-trusted-firmware/ build/
	git clone resources/arm-trusted-firmware/ $@
build/arm-trusted-firmware/build/sun50iw1p1/release/bl31.bin: | build/arm-trusted-firmware/
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARM64) -C build/arm-trusted-firmware/ PLAT=sun50iw1p1 bl31
arm-trusted-firmware: build/arm-trusted-firmware/build/sun50iw1p1/release/bl31.bin

resources/u-boot/: | resources/
	git clone --bare --branch pine64-hacks --single-branch --depth 1 https://github.com/longsleep/u-boot-pine64.git $@
build/u-boot/: | resources/u-boot/ build/
	git clone resources/u-boot/ $@
build/u-boot/include/config.mk: | build/u-boot/
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARMHF) -C build/u-boot/ sun50iw1p1_config
build/u-boot/u-boot-sun50iw1p1.bin: build/u-boot/include/config.mk
	$(MAKE) $(MAKE_OPTIONS) $(CROSS_COMPILE_ARMHF) -C build/u-boot/
build/u-boot-sun50iw1p1.tmp1.bin: $(SUNXI_PACK_TOOLS)/merge_uboot \
	build/u-boot/u-boot-sun50iw1p1.bin \
	build/arm-trusted-firmware/build/sun50iw1p1/release/bl31.bin
	# m_ub ub.bin bl.bin secmonitor
	$< $(word 2, $^) $(word 3, $^) $@ secmonitor
build/u-boot-sun50iw1p1.tmp2.bin: $(SUNXI_PACK_TOOLS)/merge_uboot \
	build/u-boot-sun50iw1p1.tmp1.bin\
	resources/scp.bin
	# m_ub tmp1.bin scp.bin scp
	$< $(word 2, $^) $(word 3, $^) $@ scp
build/u-boot-sun50iw1p1.bin: $(SUNXI_PACK_TOOLS)/update_uboot_fdt \
	build/u-boot-sun50iw1p1.tmp2.bin \
	build/linux/arch/arm64/boot/dts/allwinner/sun50i-a64-pine64.dtb \
	$(SUNXI_PACK_TOOLS)/update_uboot build/sys_config.bin
	# u_ub_fdt tmp2.bin dtb ub.bin
	$< $(word 2, $^) $(word 3, $^) $@
	# u_ub ub.bin sc.bin
	$(word 4, $^) $@ $(word 5, $^)
u-boot: build/u-boot-sun50iw1p1.bin
u-boot_flash: build/u-boot-sun50iw1p1.bin
	pv $< | dd of=$(SDCARD_DEV) bs=1K seek=8
	sync

define overlay
	$(RSYNC) --usermap=1000:root --groupmap=1000:root overlay/ build/rootfs/
endef

build/stamp.overlay: build/stamp.rootfs | /build/rootfs/
	$(call overlay)
	touch $@
overlay: | build/rootfs/
	$(call overlay)

build/rootfs/: | build/
	$(eval PACKAGES := $(shell egrep -v '^(#|//|$$)' packages | tr "\n" ,))
	rm -rf $@
	qemu-debootstrap --arch=arm64 --variant=minbase --components=main --include=$(PACKAGES) $(DEBIAN_RELEASE) $@ $(DEBIAN_REPOSITORY)

	$(MAKE) overlay
	for i in proc dev dev/pts sys; do mount -o bind /$$i $@/$$i; done
	cp configure.packages $@/tmp/packages
	[ -x $@/usr/bin/qemu-aarch64-static ] || cp /usr/bin/qemu-aarch64-static $@/usr/bin/qemu-aarch64-static
	chroot $@ /bin/bash < configure
	rm -f $@/usr/bin/qemu-aarch64-static
	rm -f $@/tmp/packages
	for i in proc dev/pts dev sys; do umount $@/$$i; done
	rm -f $@/etc/ssh/ssh_host_*_key*
	$(MAKE) overlay
	date -u '+%Y-%m-%d %H:%M:%S' > "$@/etc/fake-hwclock.data"
build/stamp.rootfs: packages configure configure.packages
	rm -r build/rootfs/
	$(MAKE) build/rootfs/
	touch $@

build/rootfs/boot/Image: build/linux/arch/arm64/boot/Image | build/rootfs/
	cp $< $@

ROOTFS_DEP := build/stamp.overlay build/rootfs/boot/Image
rootfs:
	rm -rf build/stamp.rootfs
	$(MAKE) build/stamp.rootfs
rootfs_flash: build/stamp.overlay | build/sd/root/
	mount $(SDCARD_DEV)2 build/sd/root/
	$(RSYNC) --delete --exclude '/boot*/' build/rootfs/ build/sd/root/
	mkdir -p build/sd/root/boot/
	umount build/sd/root/
bootfs_flash: build/stamp.overlay | build/sd/boot/
	mount $(SDCARD_DEV)1 build/sd/boot/
	$(RSYNC) --delete build/rootfs/boot/ build/sd/boot/
	umount build/sd/boot/
rootfs_sync: build/stamp.overlay
	$(RSYNC) --delete \
	--exclude /dev/ --exclude /sys/ --exclude /proc/ \
	--exclude /var/log/ \
	--exclude /tmp/ --exclude /var/tmp/ \
	--exclude /run/ \
	build/rootfs/ cozy.local:/

BS := $(shell echo $$((1*1024))) # 1KiB

BOOT0_OFFSET := $(shell echo 'ibase=16; 2000' | bc)
BOOT0_SKIP := $(shell echo $$(($(BOOT0_OFFSET) / $(BS))))

UBOOT_OFFSET := $(shell echo 'ibase=16; 12A6000' | bc)
UBOOT_SKIP := $(shell echo $$(($(UBOOT_OFFSET) / $(BS))))

# B/s
SECTOR_SIZE := 512
# MBR
PART_SECTOR := 40960
# 50MiB
BOOT_SIZE := $(shell echo $$((50*1024*1024)))
BOOT_SECTORS := $(shell echo $$(($(BOOT_SIZE) / $(SECTOR_SIZE))))
BOOT_LAST_SECTOR := $(shell echo $$(($(PART_SECTOR) + $(BOOT_SECTORS) - 1)))
ROOT_FIRST_SECTOR := $(shell echo $$(($(BOOT_LAST_SECTOR) + 1)))

BS := $(shell echo $$(($(BS)/1024)))k

define format
	/sbin/parted -a optimal --script $(1) \
	mklabel msdos \
		mkpart primary fat32 $(PART_SECTOR)s $(BOOT_LAST_SECTOR)s \
		mkpart primary ext4 $(ROOT_FIRST_SECTOR)s 100% \
	align-check optimal 1 \
	print
	sync
	sleep 1
	mkfs.vfat $(1)$(2)1
	mkfs.ext4 -F $(1)$(2)2
	sync
endef

define mount_img
	$(eval LOOP_DEVICE := $(shell losetup -f | sed 's#^/dev/#/dev/mapper/#'))
	kpartx -as build/cozy.img
	mount $(LOOP_DEVICE)p1 build/sd/boot/
	mount $(LOOP_DEVICE)p2 build/sd/root/
endef

define umount_img
	sync
	[ -d build/sd/boot/ ] && umount build/sd/boot/
	[ -d build/sd/root/ ] && umount build/sd/root/
	[ -f build/cozy.img ] && kpartx -d build/cozy.img
	losetup -D
endef

define flash_bootloader
	pv resources/boot0.bin | dd conv=notrunc bs=$(BS) seek=$(BOOT0_SKIP) of=$(1)
	pv build/u-boot-sun50iw1p1.bin | dd conv=notrunc bs=$(BS) seek=$(UBOOT_SKIP) of=$(1)
endef

define sync_img
	$(call flash_bootloader,build/cozy.img)

	$(call mount_img)

	$(RSYNC) --delete build/rootfs/boot/ build/sd/boot/
	sfill -zllf build/sd/boot/

	mkdir -p build/sd/root/boot/
	$(RSYNC) --delete --exclude '/boot*/' build/rootfs/ build/sd/root/
	sfill -zllf build/sd/root/

	$(call umount_img)
endef

define sync_sd
	pv resources/boot0.bin | dd conv=notrunc bs=$(BS) seek=$(BOOT0_SKIP) of=$(1)
	pv build/u-boot-sun50iw1p1.bin | dd conv=notrunc bs=$(BS) seek=$(UBOOT_SKIP) of=$(1)

	mount $(1)1 build/sd/boot/
	mount $(1)2 build/sd/root/

	$(RSYNC) --delete build/rootfs/boot/ build/sd/boot/

	mkdir -p build/sd/root/boot/
	$(RSYNC) --delete --exclude '/boot*/' build/rootfs/ build/sd/root/

	umount build/sd/boot/ build/sd/root/
endef

BOOT_DEP := resources/boot0.bin build/u-boot-sun50iw1p1.bin
bootloader_flash: $(BOOT_DEP)
	$(call flash_bootloader,$(SDCARD_DEV))
flash: bootloader_flash bootfs_flash rootfs_flash

IMG_DEP := $(BOOT_DEP) $(ROOTFS_DEP) | build/sd/boot/ build/sd/root/
build/cozy.img: $(IMG_DEP)
	truncate -s $(IMG_SIZE) $@

	$(eval LOOP_DEVICE := $(shell losetup -f))
	losetup $(LOOP_DEVICE) $@
	$(call format, $(LOOP_DEVICE),p)
	losetup -d $(LOOP_DEVICE)

	$(call sync_img)
img: build/cozy.img
img_sync: build/cozy.img
	$(call sync_img)
img_flash: build/cozy.img
	pv $< | dd of=$(SDCARD_DEV) bs=4M
	sync
build/cozy.img.xz: build/cozy.img
	pxz -9e -kv -T8 $<
	# xz clone times from source, but badly, breaking make...
	touch $@
xz: build/cozy.img.xz
xz_flash: build/cozy.img.xz
	xzcat $< | pv -s $(IMG_SIZE) | dd of=$(SDCARD_DEV) bs=4M
	sync

format:
	$(call format,$(SDCARD_DEV),)
chroot: | build/rootfs/
	for i in proc dev dev/pts sys; do mount -o bind /$$i build/rootfs/$$i; done

	[ -x build/rootfs/usr/bin/qemu-aarch64-static ] || cp /usr/bin/qemu-aarch64-static build/rootfs/usr/bin/qemu-aarch64-static
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > build/rootfs/etc/apt/apt.conf.d/00proxy

	-chroot build/rootfs/ /bin/bash

	rm -f build/rootfs/etc/apt/apt.conf.d/00proxy
	rm -f build/rootfs/usr/bin/qemu-aarch64-static
	find build/rootfs/var/cache/apt -type f -delete
	find build/rootfs/var/log/ -type f -delete
	for i in proc dev/pts dev sys; do [ -d build/rootfs/$$i/ ] && umount build/rootfs/$$i/; done
mount: build/cozy.img | build/sd/boot/ build/sd/root/
	$(call mount_img)
umount:
	-for i in proc dev/pts dev sys; do [ -d build/rootfs/$$i/ ] && umount build/rootfs/$$i/; done
	-$(call umount_img)

clean:
	$(MAKE) -s umount
	rm -rf build
