DEBIAN_REPOSITORY := http://localhost:3142/archive.raspbian.org/raspbian/
DEBIAN_RELEASE := jessie
SDCARD_DEV := /dev/sdd
IMG_SIZE ?= 2G
IMG_VERSION := 5.0.0
RSYNC := rsync -ahxAHX --info=progress2 --numeric-ids --inplace
GPG_KEY := 0xEFB74277ECE4E222

.DEFAULT_GOAL := gpg
MAKEFLAGS += --no-builtin-rules
unexport MAKEFLAGS
SHELL := /bin/bash

.PHONY: bootfs_flash chroot clean desync dev flash format img xz xz_flash img_flash img_sync mount overlay resources rootfs rootfs_flash rootfs_sync sync umount

dev:
	echo "APT::Install-Recommends \"0\";\nAPT::Install-Suggests \"0\";" > /etc/apt/apt.conf.d/60recommends
	apt install -y apt-cacher-ng
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > /etc/apt/apt.conf.d/00proxy
	curl -s https://archive.raspbian.org/raspbian.public.key | apt-key add -

	apt install -y dosfstools qemu-user-static debootstrap binfmt-support rsync apt-cacher-ng parted secure-delete pv pxz wget kpartx expect systemd-container

	$(MAKE) resources

sync:
	$(RSYNC) -P --delete --exclude-from .exclude . cozy-dev.local:img/raspi/
desync: sync
	$(RSYNC) cozy-dev.local:img/raspi/ .

build/ overlay/usr/local/bin/ overlay/etc/apt/trusted.gpg.d/ build/sd/boot/ build/sd/root/:
	mkdir -p "$@"

overlay/usr/local/bin/rpi-update: | overlay/usr/local/bin/
	wget https://raw.githubusercontent.com/Hexxeh/rpi-update/master/rpi-update -O "$@"
	chmod u+x "$@"
resources/cozy.gpg:
	wget https://debian.cozycloud.cc/cozy.gpg.key -O $@
	gpg2 --import $@
resources/nodesource.gpg:
	wget https://deb.nodesource.com/gpgkey/nodesource.gpg.key -O $@
	gpg2 --import $@

overlay/etc/apt/trusted.gpg.d/cozy.gpg: resources/cozy.gpg | overlay/etc/apt/trusted.gpg.d/
	gpg2 --export --export-options export-minimal --no-armor 0xBCB2ED5D2765B235 > $@
overlay/etc/apt/trusted.gpg.d/nodesource.gpg: resources/nodesource.gpg | overlay/etc/apt/trusted.gpg.d/
	gpg2 --export --export-options export-minimal --no-armor 0x1655A0AB68576280 > $@
resources: overlay/usr/local/bin/rpi-update \
			overlay/etc/apt/trusted.gpg.d/cozy.gpg overlay/etc/apt/trusted.gpg.d/nodesource.gpg

build/rootfs/: | build/
	$(eval PACKAGES := $(shell egrep -v '^(#|//|$$)' resources/packages | tr "\n" ,))
	rm -rf $@
	qemu-debootstrap --arch=armhf --variant=minbase --components=main --include=$(PACKAGES) --keyring /etc/apt/trusted.gpg $(DEBIAN_RELEASE) $@ $(DEBIAN_REPOSITORY)
build/stamp.rootfs: resources/packages
	rm -rf build/rootfs/
	$(MAKE) build/rootfs/
	touch $@
rootfs:
	rm -f build/stamp.rootfs
	$(MAKE) build/stamp.rootfs

SETUP_RESOURCES := resources/configure resources/configure.packages resources/cozy.debconf
SETUP_DEPS := $(SETUP_RESOURCES) resources/configure.expect
setup: resources
	$(MAKE) overlay
	cp $(SETUP_RESOURCES) build/rootfs/var/tmp/
	systemd-nspawn -M cozy-raspi --bind-ro=/usr/bin/qemu-arm-static -D build/rootfs/ /bin/bash -c "echo root:root | /usr/sbin/chpasswd"
	#systemd-nspawn -M cozy-raspi --bind-ro=/usr/bin/qemu-arm-static -D build/rootfs/ /bin/bash -c /var/tmp/configure
	resources/configure.expect
build/stamp.setup: build/stamp.rootfs $(SETUP_DEPS) | build/rootfs/
	$(MAKE) setup
	touch $@

overlay: | build/rootfs/
	$(RSYNC) --usermap=1000:root --groupmap=1000:root overlay/ build/rootfs/

	find build/rootfs/var/log/ -type f -delete
	rm -rf build/rootfs/root/.bash_history
ifndef DEV
	rm -rf build/rootfs/{etc/cozy/certs/cozy.local.*,etc/cozy/{couchdb.login,stack.token},etc/ssh/ssh_host_*_key{,.pub},root/.ssh/,etc/default/keyboard}
	touch build/rootfs/var/tmp/firstboot
endif
build/stamp.overlay: build/stamp.setup | build/rootfs/
	$(MAKE) overlay
	touch $@

bootfs_flash: build/stamp.overlay | build/sd/boot/
	mount $(SDCARD_DEV)1 build/sd/boot/
	$(RSYNC) --delete build/rootfs/boot/ build/sd/boot/
	umount build/sd/boot/
rootfs_flash: build/stamp.overlay | build/sd/root/
	mount $(SDCARD_DEV)2 build/sd/root/
	$(RSYNC) --delete --exclude '/boot*/' build/rootfs/ build/sd/root/
	mkdir -p build/sd/root/boot/
	umount build/sd/root/
flash: bootfs_flash rootfs_flash
rootfs_sync: build/stamp.setup
	$(RSYNC) --delete \
	--exclude /dev/ --exclude /sys/ --exclude /proc/ \
	--exclude /var/log/ \
	--exclude /tmp/ --exclude /var/tmp/ \
	--exclude /run/ \
	build/rootfs/ cozy.local:/

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
	mkfs.ext4 -O ^metadata_csum -F $(1)$(2)2
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
img_sync: | build/cozy.img
	$(call sync_img)
img_flash: | build/cozy.img
	pv $< | dd of=$(SDCARD_DEV) bs=4M
	sync
build/cozy-raspberry_$(IMG_VERSION).img.xz: build/cozy.img
	pxz -9e -kv -T8 $<
	# xz clone times from source, but badly, breaking make...
	touch $@
%.sha512: %
	sha512sum -b $< | sed s@build/@@ > $@
xz: build/cozy-raspberry_$(IMG_VERSION).img.xz.sha512
xz_flash: | build/cozy-raspberry_$(IMG_VERSION).img.xz
	xzcat $< | pv -s $(IMG_SIZE) | dd of=$(SDCARD_DEV) bs=4M
	sync
%.asc: %
	gpg2 -u $(GPG_KEY) -b $<
gpg: build/cozy-raspberry_$(IMG_VERSION).img.xz.sha512.asc


format:
	$(call format, $(SDCARD_DEV),)
chroot: | build/rootfs/
	echo 'Acquire::http { Proxy "http://localhost:3142"; }' > build/rootfs/etc/apt/apt.conf.d/00proxy

	-systemd-nspawn -M cozy-raspi --bind-ro=/usr/bin/qemu-arm-static -D build/rootfs/ -b

	rm -f build/rootfs/etc/apt/apt.conf.d/00proxy
	find build/rootfs/var/cache/apt build/rootfs/var/log/ build/rootfs/tmp -type f -delete
mount: build/cozy.img | build/sd/boot/ build/sd/root/
	$(call mount_img, $<)
umount:
	-$(call umount_img, $<)

clean:
	$(MAKE) -s umount
	rm -rf build/
