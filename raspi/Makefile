DEBIAN_REPOSITORY := http://localhost:3142/archive.raspbian.org/raspbian/
DEBIAN_RELEASE := jessie
SDCARD_DEV := /dev/sdd
IMG_SIZE ?= 2G
RSYNC := rsync -ahxAHPX --numeric-ids --inplace

.DEFAULT_GOAL := xz
MAKEFLAGS += --no-builtin-rules
unexport MAKEFLAGS

.PHONY: bootfs_flash chroot clean desync dev flash format img xz xz_flash img_flash img_sync mount overlay resources rootfs rootfs_flash rootfs_sync sync umount

dev:
	echo "APT::Install-Recommends \"0\";\nAPT::Install-Suggests \"0\";" > /etc/apt/apt.conf.d/60recommends
	apt install -y apt-cacher-ng
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > /etc/apt/apt.conf.d/00proxy
	curl -s https://archive.raspbian.org/raspbian.public.key | apt-key add -

	apt install -y dosfstools qemu-user-static debootstrap binfmt-support rsync apt-cacher-ng parted secure-delete pv pxz wget kpartx

	$(MAKE) resources

sync:
	$(RSYNC) --exclude-from .exclude . cozy-dev.local:img/raspi/
desync: sync
	$(RSYNC) cozy-dev.local:img/raspi/ .

build/ overlay/usr/local/bin/ build/sd/boot/ build/sd/root/:
	mkdir -p "$@"

overlay/usr/local/bin/rpi-update: | overlay/usr/local/bin/
	wget https://raw.githubusercontent.com/Hexxeh/rpi-update/master/rpi-update -O "$@"
	chmod u+x "$@"
resources: overlay/usr/local/bin/rpi-update

define overlay
	$(RSYNC) --usermap=1000:root --groupmap=1000:root overlay/ build/rootfs/
endef

build/stamp.overlay: build/stamp.rootfs | build/rootfs/
	$(call overlay)
	touch $@
overlay: | build/rootfs/
	$(call overlay)

build/rootfs/: | build/
	$(eval PACKAGES := $(shell egrep -v '^(#|//|$$)' packages | tr "\n" ,))
	rm -rf $@
	qemu-debootstrap --arch=armhf --variant=minbase --components=main --include=$(PACKAGES) --keyring /etc/apt/trusted.gpg $(DEBIAN_RELEASE) $@ $(DEBIAN_REPOSITORY)

	$(MAKE) overlay
	for i in proc dev dev/pts sys; do mount -o bind /$$i/ $@/$$i/; done
	cp configure.packages $@/tmp/packages
	[ -x $@/usr/bin/qemu-arm-static ] || cp /usr/bin/qemu-arm-static $@/usr/bin/qemu-arm-static
	chroot $@ /bin/bash < configure
	rm -f $@/tmp/packages $@/usr/bin/qemu-arm-static
	for i in proc dev/pts dev sys; do umount $@/$$i/; done
	rm -f "$@/etc/ssh/ssh_host_*_key*"
	$(MAKE) overlay
	date -u '+%Y-%m-%d %H:%M:%S' > "$@/etc/fake-hwclock.data"
build/stamp.rootfs: packages configure configure.packages
	rm -r build/rootfs/
	$(MAKE) build/rootfs/
	touch $@
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
flash: bootfs_flash rootfs_flash

# B/s
SECTOR_SIZE := 512
# MBR
PART_SECTOR := 2048
# 64MiB
BOOT_SIZE := $(shell echo $$((64*1024*1024)))
BOOT_SECTORS := $(shell echo $$(($(BOOT_SIZE) / $(SECTOR_SIZE))))
BOOT_LAST_SECTOR := $(shell echo $$(($(PART_SECTOR) + $(BOOT_SECTORS) - 1)))
ROOT_FIRST_SECTOR := $(shell echo $$(($(BOOT_LAST_SECTOR) + 1)))

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
	umount build/sd/boot/ build/sd/root/
	[ -f build/cozy.img ] && kpartx -d build/cozy.img
	losetup -D
endef

define sync_img
	$(call mount_img)

	$(RSYNC) --delete build/rootfs/boot/ build/sd/boot/
	sfill -zllf build/sd/boot/

	mkdir -p build/sd/root/boot/
	$(RSYNC) --delete --exclude '/boot*/' build/rootfs/ build/sd/root/
	sfill -zllf build/sd/root/

	$(call umount_img)
endef

IMG_DEP := build/stamp.overlay | build/sd/boot/ build/sd/root/
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
	$(call format, $(SDCARD_DEV),)
chroot: | build/rootfs/
	for i in proc dev dev/pts sys; do mount -o bind /$$i build/rootfs/$$i; done

	[ -x build/rootfs/usr/bin/qemu-arm-static ] || cp /usr/bin/qemu-arm-static build/rootfs/usr/bin/qemu-arm-static
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > build/rootfs/etc/apt/apt.conf.d/00proxy

	-chroot build/rootfs/ /bin/bash

	rm -f build/rootfs/etc/apt/apt.conf.d/00proxy
	rm -f build/rootfs/usr/bin/qemu-arm-static
	find build/rootfs/var/cache/apt -type f -delete
	find build/rootfs/var/log/ -type f -delete
	for i in proc dev/pts dev sys; do umount build/rootfs/$$i/; done
mount: build/cozy.img | build/sd/boot/ build/sd/root/
	$(call mount_img, $<)
umount:
	-for i in proc dev/pts dev sys; do umount build/rootfs/$$i/; done
	-$(call umount_img, $<)

clean:
	$(MAKE) -s umount
	rm -rf build
