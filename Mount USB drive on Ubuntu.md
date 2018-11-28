MOUNT USB DRIVE ON BOOT:
========================



## GET THE DRIVE UUID:
	oot@Pi2:~# sudo blkid
	/dev/mmcblk0p1: LABEL="boot" UUID="A75B-DC79" TYPE="vfat" PARTUUID="194bb9b0-01"
	/dev/mmcblk0p2: LABEL="rootfs" UUID="485ec5bf-9c78-45a6-9314-32be1d0dea38" TYPE=  "ext4" PARTUUID="194bb9b0-02"
	/dev/mmcblk0: PTUUID="194bb9b0" PTTYPE="dos"
	/dev/sda: UUID="31097a83-c739-4dfd-81b5-5079d93ebb0a" TYPE="ext4"
## EDIT THE FILE: /etc/fstab
	where: /mnt/usbstorage is the mount point
	UUID="31097a83-c739-4dfd-81b5-5079d93ebb0a"    /mnt/usbstorage    ext4   nofail,defaults    0   0
## TEST & MOUNT THE HDD
	sudo mount -a

