# qemu-dev
I'm trying to make learn how to make (pseudo)hardware changes to qemu, then write drivers for the appropriate hardware. Plan to mostly work around PCIe, but may work on other hardware later.

## Setting up qemu
Most of the setup and development steps were taken from [here](https://blog.davidv.dev/posts/learning-pcie/) and slightly modified to run on Arch Linux.
1. Once in the right directory, run the following to compile qemu
	```
	./configure --target-list="x86_64-softmmu" --enable-debug
	make -j`nproc`
	```
2. Use the kernel you're running to quickly test if qemu is working. To do that, run
	```
	cp /boot/vmlinuz-linux .
	```
3. To quickly test if your qemu is working with your kernel, run the following. Note that you will get a kernel panic, as we haven't configured the storage yet.
	```
	./build/qemu-system-x86_64 -kernel vmlinuz-linux \
	-display none -m 256 \
	-chardev stdio,id=char0 -serial chardev:char0 \
	-append 'console=ttyS0 quiet panic=-1'
	```
4. We will now setup `initramfs`, the temporary rootfs that's loaded into memory during the linux kernel boot process.
	```
	mkdir initramfs
	cd initramfs
	```

5. Get static busybox from wherever you want. I used Arch Linux's official package, which would be located in `/usr/bin/busybox`. You can get it from [here](https://www.busybox.net/downloads/binaries/) as well. 
	```
	cp /usr/bin/busybox .
	```
6. Make a file named `init.sh` with the following contents, and make it executable:
	```
	cat <<EOF > init.sh
	#!/busybox sh
	/busybox mkdir /sys
	/busybox mkdir /proc
	/busybox mount -t proc null /proc
	/busybox mount -t sysfs null /sys
	/busybox mknod /dev/mem c 1 1
	/busybox lspci
	exec /busybox sh
	EOF

	chmod +x init.sh
	```
7. Now, you should basically have 2 files, `busybox` and `init.sh` inside the `initramfs/` directory.
8. Run the following to generate an initramfs:
	```
	find . -print0 | cpio --null -H newc -o | gzip -9 > ../initramfs.gz && cd ..
	```
9. Finally, run the following to boot up your qemu VM
	```
	./build/qemu-system-x86_64 -enable-kvm -kernel vmlinuz-linux -initrd initramfs.gz \
	-chardev stdio,id=char0 -serial chardev:char0 \
	-append 'quiet console=ttyS0,115200 rdinit=/init.sh' \
	-display none -m 256  -nodefaults
	```
We now have a temporary working qemu machine!!

## Creating a minimal PCIe device
TODO