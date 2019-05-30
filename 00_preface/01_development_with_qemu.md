# development with qemu

Previously, instead of using real development machine, I test and run kernel
modules in [VirtualBox](https://www.virtualbox.org/), a powerful x86 and
AMD64/Intel64 virtualization product for enterprise as well as home use.
However, VirtualBox needs a full Linux distribution installation, and not
fast enought to reboot after the system crashes. [QEMU](http://www.qemu.org/)
is another vitualization tool which can OSes and programs made for one machine.
QEMU is not as convenient as VirtualBox, since users have to instruct QEMU
to behave correctlly by feeding correct command line options. But it is
more powerful I think, especially in kernel development.

# Install QEMU

The QEMU's offical page details all the needed dependencies and install
procedures. For most Linux distribution, QEMU is already included in the
package manager.

   apt-get install qemu

For other Linux distributions, check [HERE](http://www.qemu.org/download/).

The version of QEMU contained in distro's package repo is not always the
newest. Compile QEMU from souce to get the latest version is a good idea.
[QEMU's official page](http://www.qemu.org/download/#source) details how
to compile it yourself. DO NOT forget to add QEMU binary in your $PATH
environment.

# Compile Linux Kernel

1. Create a working directory

   ```bash
   mkdir Linux-device-driver
   cd Linux-device-driver
   ```

2. Download the Linux kernel.

   Download from [kernel.org](https://www.kernel.org/), Here we suppose to
   use Linux Kernel 4.9, a longterm version with EOF date of Jan, 2019.
   Here, we use linux-4.9.30 as an example.

   ```bash
   wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.30.tar.xz
   ```

3. Extract the Kernel.

   ```bash
   tar -xf linux-4.9.30.tar.xz
   cd linux-4.9.30
   ```

4. Configure the kernel.

   As [LFS](http://www.linuxfromscratch.org/lfs/view/stable/chapter08/kernel.html)
   says, "A good starting place for setting up the kernel
   configuration is to run make defconfig, This will set the base
   configuration to a good state that takes your current system architecture 
   into account." We first set the kernel to the default configuartion, then
   make some adjustment for a better suit of our needs.

   ```bash
   make defconfig
   make menuconfig
   ```

   Disable all sound card support:

   ```
   Device Drivers  --->
     < > Sound card support  ----
   ```

   Disable all wireless lan device support and USB network adapters:

   ```
   Device Drivers  --->
     [*] Network device support  --->
       [ ]   Wireless LAN  ----
       < >   USB Network Adapters  ----
   ```

   Disable all ethernet device support except intel e1000 device:

   ```
   Device Drivers  --->
     [*] Network device support  --->
       [*]   Ethernet driver support  --->
         [*]   Intel devices
           <*>     Intel(R) PRO/100+ support
   ```

   Disable IPv6 support:

   ```
    [*] Networking support  --->
      Networking options  --->
        < >   The IPv6 protocol  ----
   ```

   Enable parallel port support:
   ```
   Device Drivers  --->
     <*> Parallel port support  --->
     [*] Network device support  --->
       <*>   PC-style hardware

   Device Drivers  --->
     Character devices  --->
       <*> Parallel printer support
   ```

5. Compile the Kernel.

   ```bash
   make -j 4 bzImage
   ```

# Build initramfs

1. what is initramfs

   Except from [Linux kernel documentation](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git/tree/Documentation/filesystems/ramfs-rootfs-initramfs.txt?h=v4.9.30):

   > All 2.6 Linux kernels contain a gzipped "cpio" format archive, which is
   extracted into rootfs when the kernel boots up.  After extracting, the
   kernel checks to see if rootfs contains a file "init", and if so it executes
   it as PID 1. If found, this init process is responsible for bringing the
   system the rest of the way up, including locating and mounting the real root
   device (if any).  If rootfs does not contain an init program after the
   embedded cpio archive is extracted into it, the kernel will fall through to
   the older code to locate and mount a root partition, then exec some variant
   of /sbin/init out of that.

2. Create our initramfs image

   ```bash
   # enter the working directory we previously crated.
   cd Linux-device-driver

   # Create initramfs build directory
   mkdir initramfs

   # Create necessary directories
   mkdir -p initramfs/{bin,dev,etc,lib,lib64,mnt,proc,root,sbin,sys}

   # Copy necessary device files from host
   cp -a /dev/{null,console,tty,ttyS0} initramfs/dev/

   # Create necessary device files
   mknod initramfs/dev/parport0 c 99 0
   mknod initramfs/dev/lp0 c 6 0
   ```

3. Download busybox

   > BusyBox combines tiny versions of many common UNIX utilities into a single
   small executable. It provides replacements for most of the utilities you
   usually find in GNU fileutils, shellutils, etc. The utilities in BusyBox
   generally have fewer options than their full-featured GNU cousins; however,
   the options that are included provide the expected functionality and behave
   very much like their GNU counterparts. BusyBox provides a fairly complete
   environment for any small or embedded system.

   ```bash
   # Download pre-build busybox, you may build busybox manually according to
   # your own needs.
   wget https://www.busybox.net/downloads/binaries/1.27.1-i686/busybox -O initramfs/bin/busybox
   chmod +x initramfs/bin/busybox

   # Install busybox
   initramfs/bin/busybox --install initramfs/bin
   initramfs/bin/busybox --install initramfs/sbin
   ```

4. Compose init script

   Kernel executes init script as PID 1. this init process is responsible for
   bringing the system the rest of the way up, including locating and mounting
   the real root device (if any).

   ```bash
   touch initramfs/init
   chmod +x initramfs/init

   cat << EOF > initramfs/init
   #!/bin/busybox sh

   # Mount the /proc and /sys filesystems.
   mount -t proc none /proc
   mount -t sysfs none /sys

   # Boot real things.

   # NIC up
   ip link set eth0 up
   ip addr add 10.0.2.15/24 dev eth0
   ip link set lo up

   # wait for NIC ready
   sleep 0.5

   # make the new shell as a login shell with -l option
   # only login shell read /etc/profile
   setsid sh -c 'exec sh -l </dev/ttyS0 >/dev/ttyS0 2>&1'

   EOF
   ```

   Note how we initiate the shell, if we simply write `exec /bin/sh` here, we
   may encounter the **sh: can't access tty; job control turned off** error.
   Check [Busybox's FAQ](https://www.busybox.net/FAQ.html#job_control) for
   details.

5. Add some necessary files

   ```bash
   # name resolve
   cat << EOF > initramfs/etc/hosts
   127.0.0.1    localhost
   10.0.2.2     host_machine
   EOF

   # common alias
   cat << EOF > initramfs/etc/profile
   alias ll='ls -l'
   EOF

   # busybox saves password in /etc/passwd directly, no /etc/shadow is needed.
   cat << EOF > initramfs/etc/passwd
   root:x:0:0:root:/root:/bin/bash
   EOF

   # group file
   cat << EOF > initramfs/etc/group
   root:x:0:
   EOF
   ```

6. Create initramfs image.

   ```bash
   cd initramfs
   find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
   cd ..
   ```

# Run our kernel in QEMU

   ```bash
   qemu-system-x86_64 -enable-kvm \
                      -kernel linux-4.9.30/arch/x86_64/boot/bzImage \
                      -initrd initramfs.cpio.gz \
                      -append 'console=ttyS0' \
                      -nographic
   ```

   Press `<C-A> x` to terminate QEMU.

# if you get No such device tty2, tty3, tty4 run this on qemu console
 ln -sf /dev/null /dev/tty2
 ln -sf /dev/null /dev/tty3
 ln -sf /dev/null /dev/tty4
# Enjoy the happy life with QEMU

---

# Reference

1. [https://landley.net/writing/rootfs-howto.html][1]
2. [http://jootamam.net/howto-initramfs-image.htm][2]
3. [https://wiki.gentoo.org/wiki/Custom_Initramfs][3]
4. [https://busybox.net/FAQ.html][4]


[1]: https://landley.net/writing/rootfs-howto.html
[2]: http://jootamam.net/howto-initramfs-image.htm
[3]: https://wiki.gentoo.org/wiki/Custom_Initramfs
[4]: https://busybox.net/FAQ.html

---

### ¶ The end
