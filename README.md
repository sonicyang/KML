# Introduction

## TL; DR How to apply

This patch is based on Linux 4.4.12 with PREEMPT_RT.
You need to prepare that before applying.

```shell-session
# 0. Clone this repository
git clone git@github.com:sonicyang/KML.git

# 1. Clone Linux 4.4.12
git clone -b v4.4.12 --depth 1 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git

# You will get a copy of Linux 4.4.12 source in directory linux

# 2. Download PREEMPT_RT patch
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/4.4/older/patch-4.4.12-rt18.patch.xz

# You will get a file patch-4.4.12-rt18.patch.xz

# 3. Apply the PREEMPT_RT patch
cd linux
xzcat ../patch-4.4.12-rt18.patch.xz | patch -p1

# You will have a modified Linux 4.4.12 source with PREEMPT_RT, now.

# 4. Commit the the changes at once (Otherwise, git am will fail)
git add .
git commit -m "Apply PREEMPT_RT"

# Now you should have a clean git repository without unstaged changes

# 5. Apply these 2 patches
cp ../KML/*.patch .
git am *.patch

# You are good to go
# Compile the kernel then you should be able to use KML
```

## KML
Kernel-Mode Linux is a technology which enables us to execute user programs in kernel mode. In Kernel Mode Linux, user programs can be executed as user processes that have the privilege level of kernel mode. The benefit of executing user programs in kernel mode is that the user programs can access kernel address space directly. For example, user programs can invoke system calls very fast because it is unnecessary to switch between a kernel mode and user-mode by using costly software interruptions or context switches. In addition, user programs are executed as ordinary processes (except for their privilege level, of course), so scheduling and paging are performed as usual, unlike kernel modules. With this technology, we can implement the real time task in user space with advantages of flexibility and low latency.

KML supports generic x86_64 and ARM architecture family. At present, we evaluate KML on ARM platform along with real time enhancements, such as PREEMPT_RT and tickless kernel. The major differential of KML on ARM is that user processes specified by KML are created within SYSTEM_MODE by setting the CPSR register as SYSTEM mode and authorize this program the access of the kernel address by set_fs(KERNEL_DS). However, on ARM platform, not all system calls can be invoked directly. If the schedule related functions like schedule used in KML, the whole system will freeze or go into segfault. This reason of this restriction is not clarified and needs further investigation.
Limitation of Resource Access

With KML, though the user programs can invoke the kernel functions directly, they can't understand the symbols of these functions. One solution is that the user program invokes the kernel functions by addresses directly. Here is an example. If we want to invoke the kernel function, sys_clock_gettime, in KML user program, we can first find out the address from kernel's ELF image, named after vmlinx, by the nm command and invoke this address by casting it to a function. e.g.

Declaration:
```
long sys_clock_gettime(clockid_t which_clock, struct timespec __user *tp);
```

Invoke sys_clock_gettime in KML by address:
```
((long (*)(clockid_t which_clock, struct timespec *tp))0x8007fa20)(CLOCK_MONOTONIC, &time2);

```

It is very inconvenient for this solution because the user program needs to modify this address whenever the kernel is updated.

##vDSO
The vDSO, virtual dynamic shared object, is a small shared library that the kernel automatically maps into the address space of all user-space applications. User programs usually do not need to concern themselves with these details as the vDSO is most commonly called by the C library. This technique provides a good interface for the KML user program to access the kernel resource through symbols. When kernel is updated, what we need to do is rebuild the user program by relinking it with the new vDSO library.
Known Issues:
 - Scheduling specific operations provided by Linux kernel such as schedule can not be invoked in KML user program, otherwise system will freeze or get into segmentation faults.
 - On AM572x, powered by Cortex A15 core, LPAE (i.e. CONFIG_ARM_LPAE) should be disabled.

##Instructions:

KML now is evaluated on AM572x and i.MX6Q SDB with Linux4.4.12. To enable it, get the kernel source with 4.4.12 version and patch it by the following patch files.
 - 0001-Integrate-KML.patch
 - 0002-Use-VDSO-as-the-wrapper-for-KML-resource.patch

Then say Y in Kernel Mode Linux and VDSO field of kernel configuration, and N for ARM LPAE.
 - CONFIG_KERNEL_MODE_LINUX=y
 - CONFIG_KML_CHECK_CHROOT=y
 - CONFIG_VDSO=y
 - CONFIG_ARM_LPAE=N

In addition to the Linux kernel, the KML user program needs to be linked with vdso.so produced in kernel building (located in arch/arm/vdso/vdso.so). Simply supply vdso.so together with other object files while linking. If the program is linked with vdso correctly, the source of the symbols provided by vDSO should be LINUX not GLIBC. Here is an example for clock_gettime in vDSO.
```
$ arm-linux-gnueabihf-nm  build/user/drivers/mctest | grep clock_gettime
         U clock_gettime@@LINUX_2.6
```

The KML user program should be put under the "/trusted" directory.
