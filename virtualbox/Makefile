DEBIAN_REPOSITORY := http://localhost:3142/http.debian.net/debian/
DEBIAN_RELEASE := jessie
IMG_SIZE ?= 4G
RSYNC := rsync -ahxAHPX --numeric-ids --inplace

.DEFAULT_GOAL := xz
MAKEFLAGS += --no-builtin-rules
unexport MAKEFLAGS

.PHONY: bootfs_flash chroot clean desync dev flash format img xz xz_flash img_flash img_sync mount overlay resources rootfs rootfs_flash rootfs_sync sync umount

dev:
	echo "APT::Install-Recommends \"0\";\nAPT::Install-Suggests \"0\";" > /etc/apt/apt.conf.d/60recommends
	apt install -y apt-cacher-ng
	echo 'Acquire::http::Proxy "http://localhost:3142";' > /etc/apt/apt.conf.d/00proxy
	sed -i "s/^# PassThroughPattern: \.\*/PassThroughPattern: \.\*/" /etc/apt-cacher-ng/acng.conf
	service apt-cacher-ng restart

	apt install -y debootstrap rsync apt-cacher-ng parted secure-delete pv pxz wget kpartx apt-transport-https
	apt install -y -t jessie-backports linux-image-amd64 linux-headers-amd64 virtualbox

	$(MAKE) resources

sync:
	$(RSYNC) --delete --exclude-from .exclude . cozy-dev.local:img/virtualbox/
desync: sync
	$(RSYNC) cozy-dev.local:img/virtualbox/ .

build/ build/root/ build/cozy/:
	mkdir -p "$@"

resources/cozy.gpg:
	wget https://debian.cozycloud.cc/cozy.gpg.key -O $@
	gpg --import $@
resources/nodesource.gpg:
	wget https://deb.nodesource.com/gpgkey/nodesource.gpg.key -O $@
	gpg --import $@

overlay/etc/apt/trusted.gpg.d/cozy.gpg: resources/cozy.gpg
	gpg2 --export --export-options export-minimal --no-armor 0xBCB2ED5D2765B235 > $@
overlay/etc/apt/trusted.gpg.d/nodesource.gpg: resources/nodesource.gpg
	gpg2 --export --export-options export-minimal --no-armor 0x1655A0AB68576280 > $@
resources: overlay/etc/apt/trusted.gpg.d/cozy.gpg overlay/etc/apt/trusted.gpg.d/nodesource.gpg

define overlay
	$(RSYNC) --usermap=1000:root --groupmap=1000:root overlay/ build/rootfs/
endef

build/stamp.overlay: build/stamp.rootfs | build/rootfs/
	$(call overlay)
	touch $@
overlay: | build/rootfs/
	$(call overlay)

setup: resources
	systemd-nspawn -D build/rootfs/ apt-get autoremove --purge -y systemd sysvinit-core+
	$(MAKE) overlay
	cp resources/cozy.debconf resources/configure.packages resources/configure build/rootfs/tmp/
	systemd-nspawn --setenv=RUNLEVEL=1 -D build/rootfs/ /tmp/configure
	$(MAKE) overlay
	systemd-nspawn -D build/rootfs/ apt-get autoremove --purge -y sysvinit-core systemd-sysv+
build/rootfs/: | build/
	$(eval PACKAGES := $(shell egrep -v '^(#|//|$$)' resources/packages | tr "\n" ,))
	rm -rf $@
	debootstrap --variant=minbase --components=main --include=$(PACKAGES) $(DEBIAN_RELEASE) $@ $(DEBIAN_REPOSITORY)
	$(MAKE) setup
build/stamp.rootfs: resources/packages resources/configure resources/configure.packages
	rm -rf build/rootfs/
	$(MAKE) build/rootfs/
	touch $@
rootfs:
	rm -rf build/stamp.rootfs
	$(MAKE) build/stamp.rootfs
rootfs_sync: build/stamp.overlay
	$(RSYNC) --delete \
	--exclude /dev/ --exclude /sys/ --exclude /proc/ \
	--exclude /var/log/ \
	--exclude /tmp/ --exclude /var/tmp/ \
	--exclude /run/ \
	build/rootfs/ cozy.local:/
flash: rootfs_flash

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
	mkfs.ext4 -F $(1)$(2)1
	sync
endef

define mount_img
	$(eval LOOP_DEVICE := $(shell losetup -f | sed 's#^/dev/#/dev/mapper/#'))
	kpartx -as build/cozy.img
	mount $(LOOP_DEVICE)p1 build/root/
endef

define umount_img
	sync
	umount build/root/
	[ -f build/cozy.img ] && kpartx -d build/cozy.img
	losetup -D
endef

define sync_img
	$(call mount_img)

	$(RSYNC) --delete build/rootfs/ build/root/
	#sfill -zllf build/root/

	echo "(hd0) /dev/disk/by-uuid/$$(blkid -s UUID -o value /dev/loop0p1)" > build/root/boot/grub/device.map
	for i in dev proc sys; do mount -o bind /$$i/ build/root/$$i/; done
	chroot build/root/ update-grub
	chroot build/root/ grub-install --no-floppy --recheck /dev/loop0
	for i in dev proc sys; do umount build/root/$$i/; done

	$(call umount_img)
endef

IMG_DEP := build/stamp.overlay | build/root/
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
build/cozy/cozy.vdi: build/cozy.img | build/cozy/
	rm -f $@
	vboxmanage convertdd $^ $@ --format VDI --variant Standard --uuid 35c5efaa-bb9a-4546-afea-47f6143782cc
vdi: build/cozy/cozy.vdi
build/cozy/cozy.ovf: resources/cozy.ovf | build/cozy/
	cp $^ $@
build/cozy-virtualbox.xz: build/cozy/cozy.ovf build/cozy/cozy.vdi
	tar -I "pxz -9e -k -T8" -cvf $@ build/cozy
xz: build/cozy-virtualbox.xz

chroot: | build/rootfs/
	echo 'Acquire::http::Proxy "http://localhost:3142";' > build/rootfs/etc/apt/apt.conf.d/00proxy

	-LANG=en_US.UTF-8 LANGUAGE=en_US.UTF-8 systemd-nspawn -D build/rootfs/

	rm build/rootfs/etc/apt/apt.conf.d/00proxy
	find build/rootfs/var/cache/apt/ -type f -delete
	find build/rootfs/var/log/ -type f -delete
umount:
	-for i in dev proc sys; do umount build/rootfs/$$i/; done
	-for i in dev proc sys; do umount build/root/$$i/; done
	-$(call umount_img)

clean:
	$(MAKE) -s umount
	rm -rf build
