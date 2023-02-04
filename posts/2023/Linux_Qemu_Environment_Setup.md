# Linux Qemu Environment Setup
<sub>Date: 04/02/2023</sub>

We need to build our environment (kernel image). We want to build two different kernels: one with address sanitizer and one without (at least according to one of the links). Download a kernel from kernel.org (here I choose [linux-4.10.6](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.10.6.tar.xz)). We can get the kernel by browsing to the following:
- [https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.69.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.69.tar.xz)
We can change the parameters within this url to request v4.x and linux-4.10.6. 

Another method of obtaining the linux kernel is through Github ([github](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)). This [github post](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md)on how to setup syzkaller mentions how to use the Github method.   

**NOTE**: In order to compile Linux 4.10.6, I needed to use Ubuntu 18.06 instead of 20.04.3. If you try to build the kernel in the wrong environment, you'll run into errors. 

## Building the kernel
Once downloaded use commands to tar and build using:
```bash
tar xf linux-4.10.6.tar.xz
cd linux-4.10.6

make defconfig
make kvmconfig
```

We now need to make changes to the created ```.config``` file. 
```bash
#open .config file and add the following lines
CONFIG_DEBUG_INFO=y
CONFIG_KASAN=y # for address sanitizer
```

We will eventually want to build two instances of our kernel. One with address sanitizer disabled and one where it is enabled.
```bash
nproc # Display number of processes available  
make -j 8 # 8 = the number of processes available, when prompted for other configuration options, choose default options for now
```

After running these commands we'll have a bzImage we can use (location: /linux-4.10.6/arch/x86/boot/**bzImage**). 

##  Creating the image
We still need to create our linux image.  

Download debootstrap:
```bash
sudo apt-get install debootstrap
```

Install necessary utilities for Debian Image:
```bash
mkdir qemu
sudo debootstrap --include=openssh-server,curl,tar,gcc,\
libc6-dev,time,strace,sudo,less,psmisc,\
selinux-utils,policycoreutils,checkpolicy,selinux-policy-default \
stretch qemu
```

Create debian image:
```bash
set -eux
 
# Set some defaults and enable promptless ssh to the machine for root.
sudo sed -i '/^root/ { s/:x:/::/ }' qemu/etc/passwd
echo 'T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100' | sudo tee -a qemu/etc/inittab
printf '\nauto eth0\niface eth0 inet dhcp\n' | sudo tee -a qemu/etc/network/interfaces
echo '/dev/root / ext4 defaults 0 0' | sudo tee -a qemu/etc/fstab
echo 'debugfs /sys/kernel/debug debugfs defaults 0 0' | sudo tee -a qemu/etc/fstab
echo 'securityfs /sys/kernel/security securityfs defaults 0 0' | sudo tee -a qemu/etc/fstab

# these two cause issues on the 4.10.6 build - they're listed within syskaller - by not enabling these, our millage may vary 
# echo 'configfs /sys/kernel/config/ configfs defaults 0 0' | sudo tee -a qemu/etc/fstab
# echo 'binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc defaults 0 0' | sudo tee -a qemu/etc/fstab

echo "kernel.printk = 7 4 1 3" | sudo tee -a qemu/etc/sysctl.conf
echo 'debug.exception-trace = 0' | sudo tee -a qemu/etc/sysctl.conf
echo "net.core.bpf_jit_enable = 1" | sudo tee -a qemu/etc/sysctl.conf
echo "net.core.bpf_jit_harden = 2" | sudo tee -a qemu/etc/sysctl.conf
echo "net.ipv4.ping_group_range = 0 65535" | sudo tee -a qemu/etc/sysctl.conf
echo -en "127.0.0.1\tlocalhost\n" | sudo tee qemu/etc/hosts

echo "nameserver 8.8.8.8" | sudo tee -a qemu/etc/resolve.conf
echo "ubuntu" | sudo tee qemu/etc/hostname
sudo mkdir -p qemu/root/.ssh/
rm -rf ssh
mkdir -p ssh
ssh-keygen -f ssh/id_rsa -t rsa -N ''
cat ssh/id_rsa.pub | sudo tee qemu/root/.ssh/authorized_keys

# enables us to ssh in as Root
echo 'PermitRootLogin yes' | sudo tee -a qemu/etc/ssh/sshd_config
 
# Build a disk image
dd if=/dev/zero of=qemu.img bs=1M seek=2047 count=1
sudo mkfs.ext4 -F qemu.img
sudo mkdir -p /mnt/qemu
sudo mount -o loop qemu.img /mnt/qemu
sudo cp -a qemu/. /mnt/qemu/.
sudo umount /mnt/qemu

# NOTE
# These are added as a quality of life - change the user and group to your own
user=$(id -u -n)
group=$(id -g -n)

echo $user:$group

sudo chown $user:$group qemu.img
sudo chmod g+w qemu.img
sudo chown $user:$group ssh
sudo chown $user:$group ssh/*
```

**Note**: if you run the above script as root, then ```qemu.img``` and ```ssh folder``` are now owned by root instead of the current user. 

## Running Qemu

First install Qemu:
```bash
sudo apt-get install qemu-system-x86
```

Start up Qemu using the following: 
```bash
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel /path/to/the/bzImage-kernel-4.10.6 \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=/path/to/the/qemu.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-enable-kvm \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log
```

We can also SSH into the instance using the following: 
```bash
ssh -i ssh/id_rsa -o "StrictHostKeyChecking no" root@127.0.0.1 -p 10021
```

We can kill the Qemu instance by running the following:
```bash
pid=$(cat vm.pid)
kill -9 $pid
rm vm.pid
rm vm.log
```

## References
The following is a list resources that were useful:
- [https://dangokyo.me/2018/10/11/linux-kernel-exploitation-setting-up-debugging-environment/](https://dangokyo.me/2018/10/11/linux-kernel-exploitation-setting-up-debugging-environment/)
- [https://dangokyo.me/2018/03/08/qemu-escape-part-2-debugging-environment-set-up/](https://dangokyo.me/2018/03/08/qemu-escape-part-2-debugging-environment-set-up/)
- [https://vccolombo.github.io/cybersecurity/linux-kernel-qemu-stuck/](https://vccolombo.github.io/cybersecurity/linux-kernel-qemu-stuck/)
- [https://lists.ubuntu.com/archives/kernel-team/2016-May/077178.html](https://lists.ubuntu.com/archives/kernel-team/2016-May/077178.html)
- [https://www.kernel.org/](https://www.kernel.org/)
- [https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md)
- [https://unix.stackexchange.com/questions/46077/where-to-download-linux-kernel-source-code-of-a-specific-version](https://unix.stackexchange.com/questions/46077/where-to-download-linux-kernel-source-code-of-a-specific-version)
- [https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.10.6.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.10.6.tar.xz)
- [https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)
- [https://groups.google.com/g/syzkaller/c/CkcWOTqJvqk](https://groups.google.com/g/syzkaller/c/CkcWOTqJvqk)
- [https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh](https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh)
- [https://superuser.com/questions/599253/i-am-trying-to-ssh-into-a-server-and-it-hangs-at-login](https://superuser.com/questions/599253/i-am-trying-to-ssh-into-a-server-and-it-hangs-at-login)
- [https://unix.stackexchange.com/questions/597789/virtio-vs-e1000-vs-rtl8139-whats-the-difference](https://unix.stackexchange.com/questions/597789/virtio-vs-e1000-vs-rtl8139-whats-the-difference)
- [https://superuser.com/questions/1706286/qemu-configuration-to-host-a-linux-with-ssh-and-http-on-windows](https://superuser.com/questions/1706286/qemu-configuration-to-host-a-linux-with-ssh-and-http-on-windows)

## Questions
If you have any questions you can ask me on twitter [@lewisparsons123](https://twitter.com/lewisparsons123).
