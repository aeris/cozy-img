DEBIAN_REPOSITORY := http://localhost:3142/http.debian.net/debian/
DEBIAN_RELEASE := jessie
IMG_SIZE ?= 10G
IMG_VERSION := 4.0.1
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
	echo 'Acquire::http::Proxy "http://localhost:3142";' > /etc/apt/apt.conf.d/00proxy
	sed -i "s/^# PassThroughPattern: \.\*/PassThroughPattern: \.\*/" /etc/apt-cacher-ng/acng.conf
	service apt-cacher-ng restart

	apt install -y debootstrap rsync apt-cacher-ng parted secure-delete pv pxz wget kpartx apt-transport-https linux-image-amd64 linux-headers-amd64 virtualbox expect systemd-container gnupg2 qemu-utils

        $(MAKE) resources

sync:
        $(RSYNC) --delete --exclude-from .exclude . cozy-dev.local:img/virtualbox/
desync: sync
        $(RSYNC) cozy-dev.local:img/virtualbox/ .

build/ build/root/ build/cozy/:
	mkdir -p "$@"

resources/cozy.gpg:
	wget https://debian.cozycloud.cc/cozy.gpg.key -O $@
	gpg2 --import $@
resources/nodesource.gpg:
	wget https://deb.nodesource.com/gpgkey/nodesource.gpg.key -O $@
	gpg2 --import $@

overlay/etc/apt/trusted.gpg.d/cozy.gpg: resources/cozy.gpg
	gpg2 --export --export-options export-minimal --no-armor 0xBCB2ED5D2765B235 > $@
overlay/etc/apt/trusted.gpg.d/nodesource.gpg: resources/nodesource.gpg
	gpg2 --export --export-options export-minimal --no-armor 0x1655A0AB68576280 > $@
resources: overlay/etc/apt/trusted.gpg.d/cozy.gpg overlay/etc/apt/trusted.gpg.d/nodesource.gpg

build/rootfs/: | build/
	$(eval PACKAGES := $(shell egrep -v '^(#|//|$$)' resources/packages | tr "\n" ,))
	rm -rf $@
	debootstrap --variant=minbase --components=main --include=$(PACKAGES) $(DEBIAN_RELEASE) $@ $(DEBIAN_REPOSITORY)
build/stamp.rootfs: resources/packages
	rm -rf build/rootfs/
	$(MAKE) build/rootfs/
	touch $@
rootfs:
	rm -rf build/stamp.rootfs
	$(MAKE) build/stamp.rootfs

SETUP_RESOURCES := resources/configure resources/configure.packages resources/cozy.debconf
SETUP_DEPS := $(SETUP_RESOURCES) resources/configure.expect
setup: resources
	$(MAKE) overlay
	cp $(SETUP_RESOURCES) build/rootfs/var/tmp/
	systemd-nspawn -M cozy-virtualbox -D build/rootfs/ /bin/bash -c "echo root:root | /usr/sbin/chpasswd"
	resources/configure.expect
build/stamp.setup: build/stamp.rootfs $(SETUP_DEPS) | build/rootfs/
	$(MAKE) setup
	touch $@

overlay: | build/rootfs/
	$(RSYNC) --usermap=1000:root --groupmap=1000:root overlay/ build/rootfs/

	find build/rootfs/var/log/ -type f -delete
	rm -f build/rootfs/root/.bash_history
ifndef DEV
	rm -rf build/rootfs/{etc/cozy/certs/cozy.local.*,etc/ssh/ssh_host_*_key{,.pub},root/.ssh/,etc/default/keyboard}
	touch build/rootfs/var/tmp/firstboot
endif
build/stamp.overlay: build/stamp.setup | build/rootfs/
	$(MAKE) overlay
	touch $@

rootfs_sync: build/stamp.setup
	$(RSYNC) --delete \
	--exclude /dev/ --exclude /sys/ --exclude /proc/ \
	--exclude /var/log/ \
	--exclude /tmp/ --exclude /var/tmp/ \
	--exclude /run/ \
	build/rootfs/ cozy.local:/

# MBR
PART_SECTOR := 2048

define format
	/sbin/parted -a optimal --script $(1) \
		mklabel msdos \
		mkpart primary ext4 $(PART_SECTOR)s 100% \
		align-check optimal 1 \
		print
	sync
	sleep 1
	mkfs.ext4 -O ^metadata_csum -F $(1)$(2)1
	sync
endef

build/cozy.img: build/stamp.overlay | build/root/
	truncate -s $(IMG_SIZE) $@

	$(eval LOOP_DEVICE := $(shell losetup -f))
	losetup $(LOOP_DEVICE) $@
	$(call format,$(LOOP_DEVICE),p)
	losetup -d $(LOOP_DEVICE)

	$(eval LOOP_DEVICE := $(shell losetup -f))
	$(eval MAPPER_DEVICE := $(shell echo $(LOOP_DEVICE) | sed 's@^/dev/@/dev/mapper/@'))

	kpartx -as build/cozy.img
	mount $(MAPPER_DEVICE)p1 build/root/

	$(RSYNC) --delete build/rootfs/ build/root/

	sfill -zllf build/root/
	echo "(hd0) /dev/disk/by-uuid/$$(blkid -s UUID -o value $(MAPPER_DEVICE)p1)" > build/root/boot/grub/device.map
	for i in dev proc sys; do mount -o bind /$$i/ build/root/$$i/; done
	chroot build/root/ update-grub
	chroot build/root/ grub-install --no-floppy --recheck $(LOOP_DEVICE)
	for i in dev proc sys; do umount build/root/$$i/; done

	umount build/root/
	kpartx -d $(LOOP_DEVICE)
	losetup -d $(LOOP_DEVICE)
img: build/cozy.img
img_sync:
	$(eval LOOP_DEVICE := $(shell losetup -f))
	$(eval MAPPER_DEVICE := $(shell echo $(LOOP_DEVICE) | sed 's@^/dev/@/dev/mapper/@'))
	kpartx -as build/cozy.img
	mount $(MAPPER_DEVICE)p1 build/root/
	$(RSYNC) --delete --exclude /boot/ build/rootfs/ build/root/
	umount build/root/
	kpartx -d $(LOOP_DEVICE)
        losetup -d $(LOOP_DEVICE)
build/cozy/cozy.vdi: build/cozy.img | build/cozy/
        rm -f $@
        vboxmanage convertdd $^ $@ --format VDI --variant Standard --uuid 35c5efaa-bb9a-4546-afea-47f6143782cc
vdi: build/cozy/cozy.vdi
vdi_mount: build/cozy/cozy.vdi | build/root/
        qemu-nbd -c /dev/nbd0 build/cozy/cozy.vdi
	kpartx -as /dev/nbd0
	mount /dev/mapper/nbd0p1 build/root/
vdi_umount:
	-umount build/root/
	-kpartx -d /dev/nbd0
        -qemu-nbd -d /dev/nbd0

build/cozy/cozy.vbox: resources/cozy.vbox | build/cozy/
        cp $^ $@
vbox: build/cozy/cozy.vbox

build/cozy-virtualbox_$(IMG_VERSION).tar.xz: build/cozy/cozy.vbox build/cozy/cozy.vdi
        tar -C build -I "pxz -9e -k -T8" -cvf $@ cozy
%.sha512: %
        sha512sum -b $< | sed s@build/@@ > $@
xz: build/cozy-virtualbox_$(IMG_VERSION).tar.xz.sha512
%.asc: %
        gpg2 -u $(GPG_KEY) -b $<
gpg: build/cozy-virtualbox_$(IMG_VERSION).tar.xz.sha512.asc

chroot: | build/rootfs/
        echo 'Acquire::http::Proxy "http://localhost:3142";' > build/rootfs/etc/apt/apt.conf.d/00proxy

	-LANG=en_US.UTF-8 LANGUAGE=en_US.UTF-8 systemd-nspawn -M cozy-virtualbox -D build/rootfs/ -b

	rm build/rootfs/etc/apt/apt.conf.d/00proxy
	find build/rootfs/var/{cache/apt,log} -type f -delete

virtualbox:
	vboxmanage list runningvms | grep '^"cozy"' && vboxmanage controlvm cozy poweroff || true
	rsync -hP --inplace cozy-dev.local:img/virtualbox/build/cozy/cozy.vdi build/
	vboxmanage startvm cozy

umount:
	-[ -d build/rootfs/dev/ ] && umount build/rootfs/dev/ || true
	-[ -d build/rootfs/sys/ ] && umount build/rootfs/sys/ || true
	-[ -d build/rootfs/proc/ ] && umount build/rootfs/proc/ || true

	-[ -d build/root/dev/ ] && umount build/root/dev/ || true
	-[ -d build/root/sys/ ] && umount build/root/sys/ || true
	-[ -d build/root/proc/ ] && umount build/root/proc/ || true

	-[ -d build/root/ ] && umount build/root/ || true

	-[ -f build/cozy.img ] && kpartx -d build/cozy.img || true
	-[ -f /dev/loop0 ] && kpartx -d /dev/loop0 || true

	-losetup -D

clean:
	$(MAKE) -s umount
	rm -rf build/
