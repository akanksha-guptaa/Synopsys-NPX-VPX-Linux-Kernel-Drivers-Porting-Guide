# Synopsys NPX/VPX Linux Kernel Drivers Porting Guide

## Build Environment and requirements:

Buildroot environment is used to build required toolchain and Linux system image for supported platform.
These instructions are written based on ARC HS host processor, with specific instructions for adapting to ARM host where required.

Linux Desktop/Serve Host Environment:
  * Linux Ubuntu 20.04 distro or equivalent

Information about packages necessary for buildroot can be found at: https://buildroot.org/downloads/manual/manual.html#requirement

To setup buildroot environment, download buildroot sources and switch to the stable 2023.11 tag:
```
 git clone https://github.com/buildroot/buildroot.git
 cd ./buildroot
 git checkout 2023.11
```

## Driver code

The Synopsys NPX/VPX Linux drivers are available as part of the MetaWare MX toolchain in the form of diff patches for the Linux 5.15 and 6.6 kernel source trees as well as from linux kernel source repository on github:

https://github.com/foss-for-synopsys-dwc-arc-processors/snps-accel-linux

### Get Linux kernel source and apply the patch with the drivers
```
 git clone https://github.com/torvalds/linux.git
 cd linux
 git checkout v6.6
 cp ~/0001-snps-accel-6.6.patch ./
 patch -p1 -i 0001-snps-accel-6.6.patch
```
Get Linux kernel tree with the drivers:
```
 git clone https://github.com/foss-for-synopsys-dwc-arc-processors/snps-accel-linux.git
```
The default branch contains only README with brief instruction. Please select the required branch manually to access the certain kernel source tree with the NPX/VPX drivers. 

Stable branches:
* snps_accel-v5.15 - the Linux 5.15 kernel source trees with the drivers 
* snps_accel-v6.6 - the Linux 6.6 kernel source trees with the drivers 

Switch to the required branch
```
 git switch snps_accel-v5.15
```
### Files Hierarchy

The NPX/VPX drivers location in the Linux device tree
```
[Linux kernel source tree top directory]
|-- drivers
    |-- misc
        |-- snps_accel                         -> ARCSync and accelerator drivers source dir
            |-- Kconfig
            |-- Makefile
            |-- snps_accel_drv.c               -> accel helper driver API 
            |-- snps_accel_drv.h               -> accel helper driver internal structures
            |-- snps_accel_mem.c               -> accel helper driver memory management
            |-- snps_accel_mem.h               -> accel helper driver mm internal header
            |-- snps_arcsync.c                 -> ARCSync control driver
    |-- remoteproc
        |-- snps_accel
            |-- accel_rproc.c                  -> VPX/NPX driver in rempteproc framework
            |-- accel_rproc.h                  -> VPX/NPX rproc internal header
            |-- npx_config.c                   -> NPX Cluster Network setup
|-- include
    |-- linux
        |-- snps_arcsync.h                     -> ARCSync driver header to share with drivers
    |-- uapi
        |-- misc
            |-- snps_accel.h                   -> accel helper driver header for user space applications
|-- arch
    |-- arc
        |-- boot
            |-- configs
                |--haps_hs_npp_defconfig      -> Example of the kernel defconfig for the NPU protoryping
                                                 platform with ARC host (NPP)
            |-- dtc
                |-- zebu_hs_npp.dts           -> Example of the DTS file for the NPP running on ZeBu
                |-- haps_hs_npp.dts           -> Example of the DTS file for the NPP running on HAPS
                |-- haps_hs_npx6_8k_vpx.dts   -> Example of extended configuration for NPP on HAPS
```

### Drivers description

The NPX/VPX drivers stack implements several drivers to cover all the requirements of userspace runtimes such as the Synopsys NN Runtime for NPX for privileged functions access. It includes the following drivers:

- **snps_accel_rproc** - the driver in the remoteproc framework to setup VPX and NPX processors, upload and start processors firmware. It allows to use common remoteproc controls in /sys/class/remoteproc

- **snsp_arcsync** - the platform driver to manage instances of ARCSync IP present in a given SoC. It provides functions to control NPX/VPX processors reset/start/stop/power_on/power_off and allows to receive notification via ARCSync interrupts to the host. This driver doesn't provide an API for user space clients.

- **snps_accel_app** - the platform driver to facilitate user space runtime access to the kernel-space objects such as accelerator shared memory region, ARCSync MMIO, notification interrupts and DMA buffers.

### Enabling drivers in the kernel config

To enable accelerator drivers, select options (these options will appear if the patch with drivers correctly applied) do 'make menuconfig' and select options:
```
SNPS_ACCEL_APP=y
SNPS_ACCEL_RPROC=y
```
It is possible to build the drivers as kernel modules. The NPX/VPX drivers require several extra nodes in the kernel Device Tree Source file 
with the description of accelerator resources. The drivers DTS nodes and their configuration are described in detail below.

### Building the linux image

If you have a kernel config for your SoC chip/board, a correct Device Tree Source file, a file with prepared rootfs and a cross toolchain, you can build a linux image from a linux source tree:
```
 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j8 CONFIG_INITRAMFS_SOURCE=../rootfs.cpio
```
Another way is to use build environments such as Buildroot or YOCTO, to simplify and automate the process of building a complete Linux system for an embedded system using cross-compilation.

## Preparing linux image with NPX/VPX driver using the buildroot environment:

The following steps describe how to add Synopsys accelerator drivers to a system Linux image.

In the buildroot environment run ```make menuconfig``` and choose desired Linux kernel version (6.6 used in example below) :
```  
 make menuconfig
```
Set the following options:
```
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.6"
```
If you have a patch with the driver, put the diff patch file to the *buildroot/linux/6.6* directory to allow buildroot to apply this patch before building.

If you have using the Linux kernel tree with integrated NPX/VPX drivers from github you can create local.mk file in your buildroot directory and specify path for the directory with the kernel code:
```
 echo "LINUX_OVERRIDE_SRCDIR=path/to/linux/kernel/src" > local.mk
```
Set the directory containing the accelerators firmware image(S) and extra host apps that should be copied to the root file system of the image:
```
BR2_ROOTFS_OVERLAY="$(O)/override"
```
Set post-build script:

One additional step is needed to copy the driver public UAPI header to the toolchain sysroot directory to be able to use application helper driver API header in user-space applications. To do this, create a post-build script in buildroot directory, for example:
```
 echo '#!/bin/sh' > board/synopsys/npp/post-build.sh
 echo 'cp $BUILD_DIR/linux-6.6/include/uapi/misc/snps_accel.h  $STAGING_DIR/usr/include/misc' >> board/synopsys/npp/post-build.sh
 chmod +x board/synopsys/npp/post-build.sh
```
Then set ROOTFS_POST_BUILD_SCRIPT option:
```
BR2_ROOTFS_POST_BUILD_SCRIPT="board/synopsys/npp/post-build.sh"
```
Ensure that you have a Linux config file for your host CPU (SoC) and this
config is specified in the buildroot config and the correct Device Tree Source 
file is used:
```
BR2_LINUX_KERNEL_DEFCONFIG="npu_platform_host_cpu_config"
```
If a previous build exists, clear the Linux kernel source directory in order to apply patches 
in the next build:
```
 make linux-dirclean
```
Run ```linux menuconfig``` and enable accelerator drivers:
```
 make linux-menuconfig
```
To enable accel drivers select options:
```
SNPS_ACCEL_APP=y
SNPS_ACCEL_RPROC=y
```
After completed all above changes build the linux image:
```
  make
```

## Device Tree Source (DTS) Bindings

To bring-up the NPX/VPX drivers, a kernel DTS file should be updated correctly with
the proper accelerator resources description.

### Structure of special DTS accelerator nodes

Each accelerator chip (SoC) with NPX and VPX cores must have their own snps_accel node with the ```"snps,accel"``` compatible label.
For each node the **snps_accel_app** driver creates an accelerator instance.

Each accelerator node may have one or several remoteproc nodes with a description of the core or cores and the firmware that will run on these cores. Each remoteproc node must have compatible label ```"snps,npx-rproc"``` or ```"snps,vpx-rproc"```.
Based on the properties in the remoteproc node the **snps_accel_rproc** driver configures NPX/VPX groups, cores and starts execution of the firmware.

Each accelerator node may have one or several application nodes with a description of resources that will be accessible to the user space driver and user space clients. Each application node must have a compatible label ```"snps,accel-app"```. For each node with this label the **snps_accel_app** driver creates an application device instance and creates character device with index in ```/dev/snps/arcnetN/appM```, where _M_ - device index, _N_ - index of ARCnet (ARCSync).

For example, for the configuration with two different firmware for NPX cores and one firmware for VPX core and separate Linux host applications for NPX and VPX the configuration of nodes in DTS will have the following high-level structure.
```
snps_accel ---
             |---remoteproc_npx0
             |---remoteproc_npx1
             |---remoteproc_vpx
             |---app_npx
             |---app_vpx
```
In addition to the accelerator node ```snps_accel``` the accelerator configuration in Device Tree should contain two extra types of nodes: ARCSync IP (control unit) node and NPX configuration node.

The ARCSync node must contain compatible label ```"snps,arcsync"```. If system has several ARCSync IP units, ARCSync node for each control unit should be added. For each node with the ```"snps,arcsync"``` compatible label the snps_arcsync driver creates an instance.
The snps_arcsync driver provides internal API for the snps_accel_app and snps_accel_rproc driver to control NPX and VPX cores through the ARCSync unit. Each remoteproc and application node must contain property ```snps,arcsync-ctrl``` with the reference to the ARCsync node.

The NPX config node contains the address of the NPX AXI configuration interface MMIO and a set of Cluster Network parameters which are used by the snps_accel_rproc driver to setup NPX Cluster Network. The first remoteproc node for NPX processors should contain property ```snps,npu-cfg``` with the reference to the NPX config node to perform initial Cluster Network setup.

## Device Tree Source (DTS) properties description and usage notes

### Top level snps_accel node properties:
```
...
#address-cells = <2>;
#size-cells = <2>;
...
snsp_accel@0 {
	compatible = "snps,accel", "simple-bus";
	#address-cells = <1>;
	#size-cells = <1>;
	reg = <0x0 0x0000000 0x1 0x00000000>;
	range = <0x10000000 0x0 0x10000000 0x20000000>;
	...
}
...
```
**compatible**

The top-level node with the accelerator description should contain compatible property with the ```"snps,accel"``` and ```"simple-bus"``` labels.

```"snps,accel"``` – compatible string for driver matching

```"simple_bus"``` – bus functionality to manage child nodes

**#address-cells and #size-cells**

Within an accelerator node (snps_accel) and in child nodes the device address (DA) must be used to describe shared memory regions.
The accelerator has shared memory in 32-bit address space.
```
#address-cells = <1>;
#size-cells = <1>;
```
Set address and size cells number to one for convenience. This should be enough for a 32-bit space.

**reg**

The ```reg``` property is used to describe location of the entire accelerator shared memory in pairs of ```<address>``` and ```<size>```. This property must contain the device address (DA), not the CPU address. The drivers don't map this entire shared memory region. This value is used by the 
remoteproc and accel_app drivers to check that their shared memory regions, specified in the remoteproc and application child nodes are in the correct range.

**ranges**

The ```ranges``` property describes shared memory DA->PA (device address to CPU address) translations. ```ranges``` is a list of address translations. Each entry in the ranges table is a tuple containing the child address, the parent address, and the size of the region 
in the child address space. The size of each field is determined by taking the child's ```#address-cells``` value, the parent's ```#address-cells value```, and the child's ```#size-cells``` value.

The remoteproc driver reads the ```ranges``` property and calculates the offset between the device and the CPU address. This offset is needed to correctly locate the firmware loadable sections linked with the device address.

_Usage examples reg and ranges:_

The system has 64Mb shared memory region between the accelerator and the host CPU. The accelerator and the host CPU view this memory at address `0xC0000000`. This is
situation with 1-to-1 translation, no offset. The description of the shared memory
region should be the following:
```
reg = <0x0 0xC0000000 0x0 0x4000000>;
ranges = <0xC0000000 0x0 0xC0000000 0x4000000>;
```
The system has 64Mb shared memory region between the accelerator and the host CPU. The accelerator address `0xD0000000`, but the host CPU has shared memory at `0xC0000000`. This is situation with `-0x10000000` device to CPU translation offset. 
The description of the shared memory region in this case should be the following:
```
reg = <0x0 0xD0000000 0x0 0x4000000>;
ranges = <0xD0000000 0x0 0xC0000000 0x4000000>;
```
The default dts files contain the following description:
```
reg = <0x0 0x00000000 0x0 0x8000000>;
ranges = <0x00000000 0x0 0x00000000 0x8000000>;
```
This default config tells the accelerator and remoteproc drivers consider the first 2Gb of CPU address space as a shared memory with 1-to-1 memory translation and allows drivers to use this region for the firmware code and runtime data.

### remoteproc driver node properties:
```
	remoteproc_npx0: remoteproc_npx@0x8000000 {
		compatible = "snps,npx-rproc";
		reg = <0x10000000 0x1000000>;
		firmware-name = "npx-app.elf";
		snps,npu-cfg = <&npu_cfg0>;
		snps,arcsync-ctrl = <&arcsync0>;
		snps,arcsync-cluster-id = <0x0>;
		snps,arcsync-core-id = <0x1>;
		snps,auto-boot;
	};
```

**compatible**

Each remoteproc node for the accelerator must contain `"snps,npx-rproc"` or `"snps,vpx-rproc"` compatible label. Use the `"snps,vpx-rproc"` compatible label if remoteproc driver starts VPX cores. Use the `"snps,npx-rproc"` compatible label for NPX cores. For NPX cores the driver (if this is described in DTS) performs additional NPX groups and NPX Cluster Network setup.

**reg**

The reg property describes shared memory regions for firmware loadable sections. The remoteproc framework requires static memory definition for a firmware code. Memory for all loadable sections that remoteproc driver loads must be specified in the reg property. The driver maps regions specified in the reg property before elf sections load. If multiple shared memory regions are needed for firmware, they can be added as a comma-separated array:
```
reg = <0x10000000 0x100000>,
      <0x10200000 0x100000>,
      <0x10300000 0x100000>;
```

**firmware-name**

The firmware-name contains firmware file name in `/lib/firmware` directory. The driver
looks for the firmware file with the name specified in this property. The firmware name can be changed lately by writing to the remoterpoc sysfs entry
_firmware_:
```
	$ echo new_firmware.elf > /sys/class/remoteproc/remoteproc0/firmware
```
**snps,npu-cfg**

The `snps,npu-cfg` property contains a reference (phandle) to a NPX config node. This property tells the driver to read the NPU Cluster Network description from the NPX config node and perform the Cluster Network setup. The NPU Cluster Network setup should be done once. If the accelerator description contains several remoteproc nodes for NPX cores only one node, the first one should contain the `snps,npu-cfg` property with the NPX config node reference.

**snps,arcsync-ctrl**

The `snps,acrsync-ctrl` property contains a reference (phandle) to a ARCSync node. This property allows the remoteproc driver to get ARCSync device reference from the arcsync driver and to address certain ARCSync control unit. If several ARCSync presence in the system different accelerators may reference to a different ARCSync by this property.

**snps,arcsync-cluster-id**

This property allows driver to know the accelerator cluster identifier (cluster number) visible to the ARCSync interface. Each processor (cluster) in the system controlled by the ARCSync unit has a unique cluster identifier. The driver uses this identifier to calculate core id and to send control commands to the cores though the ARCSync driver. This property is not an array, only one cluster number can be specified.

For example, the NPP development platform has the following cluster identifiers: _NPX - 0_, _VPX - 1_, _host CPU - 2_;

**snps,arcsync-core-id**

The `snps,arcsync-core-id` property contains one or more core numbers. These numbers are core numbers inside the cluster. For example, for NPX cluster that has one L2 and four L1 cores numbers for `snps,arcsync-core-id` will be: _L2 - 0_, _L1<sub>0</sub> - 1_, _L1<sub>1</sub> - 2_,
_L1<sub>2</sub> - 3_, _L1<sub>3</sub> - 4_.
The remoteproc driver uses `cluster-id` and `core-id` to send control commands to the certain core through the arcsync driver functions calls. The arcsync driver calculates effective core-id base on cluster-id and core-id received from the remotproc driver.

If the firmware is single core only one number should be specified. Several core numbers can be specified only for the firmware with SMP support, as the driver initializes all cores with one reset vector.

**snps,auto-boot**

This property tells the driver to upload firmware and start cores during the driver start. Without this option the firmware can be started lately by writing to the remoteproc sysfs entry _state_:
``` 
 echo start > /sys/class/remoteproc/remoteproc0/state
```
### Application helper driver node properties:
```
	app_npx0: app_npx@20000000 {
		compatible = "snps,accel-app";
		reg = <0x20000000 0x10000000>;
		snps,arcsync-ctrl = <&arcsync0>;
		interrupts = <23>;
	};
```

**compatible**

Each application node must contain `"snps,accel-app"` compatible label. For each node with this label the snps_accel_app driver creates an application device instance and creates a character device.

**reg**

The reg property describes a region in the accelerator shared memory reserved for the core firmware runtime (control structures, communication queues, buffers). The firmware and userspace host application share this region. The driver reads only one the first region from the `reg` property.

_Usage notes:_

The integrator is responsible for specifying the correct amount of shared memory for the firmware and host userspace application. The size must match the amount of available shared memory and the accelerator firmware and userspace application requirements. A userspace application is allowed to
mmap this region to its own address space to access it, and an incorrect value in this property can cause the userspace application to crash if it needs to access outside the allowed space.

**snps,arcsync-ctrl**

The `snps,acrsync-ctrl` property contains a reference (phandle) to an ARCSync node. This property allows the application driver to get ARCsync device reference from the arcsync driver and to address certain ARCSync control unit and to receive notifications about interrupts from this unit. If several ARCSync presence in the system different accelerators may reference to a different ARCSync by this property.

**interrupts**

The interrupts property contains interrupt specifier, that describe certain interrupt line. The interrupts property is the default device tree property. For a full technical description refer to the ePAPR v1.1 specification.
The specifier in the `interrupts` property must match with one of IRQs in the ARCSync node. The driver doesn't install its own ISR, but it uses this number to request notifications from the ARCsync driver. The driver reads only the first interrupt specifier and can work only with one interrupt.

### ARCSync driver node properties:
```
	arcsync0: arcsync@d4000000 {
		compatible = "snps,arcsync";
		reg = <0x0 0xd4000000 0x0 0x1000000>;
		snps,arcnet-id = <0x0>
		snps,host-cluster-id = <0x2>;
		interrupts = <23>, <22>;
	};
```

**compatible**

The ARCSync node must contain `"snps,arcsync"` compatible label. For each node with this label the snps_arcsync driver creates a device instance.

**reg**

This property contains the address and size of the ARCsync MMIO region.

**snps,arcnet-id**

The logical ARCSync (ARCNet) identifier. This option can be skipped. The default value - 0. The logical ARCSync identifier is used by the application driver to create character device in certain ARCNet: `/dev/snps/arcnetN/appM`, where _N_ - index of ARCnet (ARCSync), _M_ - device index.

**snps,host-cluster-id**

This property describes the host processor (cluster) identifier as it is visible to the ARCsync interface. This value is needed to the arcsync driver to calculate offset in the ARCSync MMIO to manage host interrupts (for interrupt ACK).

**interrupts**

The interrupts property contains interrupt specifiers, that describe certain ARCSync interrupt lines. The ARCSync control unit may have several interrupt lines connected to the host CPU. The ARCSync driver reads all interrupt specifiers from the interrupts properties and sets ISR for each interrupt.

### NPU config node properties:

The remoteproc driver for the NPX processor reads properties from the NPU config  node and performs groups setups and Cluster Network setup. 
The driver finds this node by the `phandle` reference in the `snps,npu-cfg` property in the remoteproc node.
```
	npu_cfg0: npu_cfg@d3000000 {
		reg = <0x0 0xd3000000 0x0 0xF4000>;
		snps,npu-slice-num = <4>;
		snps,npu-group-num = <1>;
		snps,cln-map-start = <0xE0000000>;
		snps,csm-size = <0x4000000>;
		snps,csm-banks-per-group = <8>;
		snps,stu-per-group = <2>;
		snps,cln-safety-lvl = <1>;
	};
```

**reg**

This property contains the address and size of the NPX AXI configuration interface MMIO.

**snps,npu-slice-num**

Number of L1 slices of the NPX processor. This is the main property for the NPX remoteproc driver, the driver counts all other CLN setup options based in this property.

**snps,npu-group-num**

Number of groups in the NPX processor. If number of groups is not specified, the driver counts number of groups as:
```
   1 group if num of slices <= 4,
   2 groups if 4 < num of slices <= 8,
   4 groups if num of slices > 8.
```

**snps,cln-map-start**

The start offset (address) of DMI mappings in the Cluster Network address space. The default value is `0xE0000000`.

**snps,csm-size**

The size of the NPX Cluster Shared Memory (CSM). This property can be skipped. The default value - 64 Mb (`0x4000000`).

**snps,csm-banks-per-group**

Number of L2 CSM banks per group. If this property is not specified, the default value is used. Default - 8.

**snps,stu-per-group**

Number of STU channels per group. The default value is 2.

**snps,cln-safety-lvl**

This value tells the driver if the NPX processor has functional safety extension or not. The default value is 1. If L1/L2 cores of the NPX processor doesn't have safety registers (no FS extension) the option `snps,cln-safety-lvl` with value 0 should be used.

_Usage notes:_

All these options affect Cluster Network Configuration. In general setup default values should be used. The mandatory parameters are only **reg** and `snps,npu-slice-num`
```
	npu_cfg0: npu_cfg@d3000000 {
		reg = <0x0 0xd3000000 0x0 0xF4000>;
		snps,npu-slice-num = <4>;
	};
```
The driver implements the recommended CLN configuration. If this configuration is not suitable for the certain chip some changes in code may be needed. First of all, check the _remoteproc/snps/npx_config.c_ file. Good starting point is definitions in the beginning of this file.

For example, some Cluster Network properties like group connections are hardwired to certain ports according to the outgoing shuffle table. The driver implements the recommended table and this table coded in the `groups_map` array. Change values in  this array if you need to change connections map.

### Reserve host CPU physical address region for cluster shared memory

For some configurations, where shared memory is part of the system memory, it is necessary to exclude the accelerator shared memory region from the system memory pool, this can be done with help of the reserved-memory node:
```
reserved-memory {
	#address-cells = <2>;
	#size-cells = <2>;
	ranges;
	reserved: buffer@0 {
		compatible = "removed-dma-pool";
		no-map;
		reg = <0x0 0x10000000 0x0 0x20000000>;
	};
};
```
This description tells the kernel memory manager that it should not use the region with PA from `0x10000000` to `0x30000000` to allocate system memory.


### Use of Linux contiguous memory allocator (CMA)

The accelerator helper driver supports dma-bufs, it allocates contiguous buffers for dma-bufs. Default Linux physical memory allocator (buddy allocator) not allow to get huge contiguous buffers. Contiguous memory allocator (CMA) allows to overcome this limitation.
The VPX/NPX driver uses internal Linux kernel DMA framework to allocate buffers in system memory and if CMA is enabled for DMA buffers (the Linux configuration option `CONFIG_DMA_CMA` is set) then the driver requests buffers through the CMA.

To enable CMA set the kernel configuration option `CONFIG_DMA_CMA`

In .dts file reserve some memory for the CMA pool. Example of reserve memory node in dts:
```
reserved-memory {
	#address-cells = <2>;
	#size-cells = <2>;
	ranges;

	reserved: buffer@0 {
		compatible = "shared-dma-pool";
		reusable;
		reg = <0x0 0xC0000000 0x0 0x10000000>;
		linux,cma-default;
	};
};
```

### Device tree example 1

This example describes a simple configuration for the ARC host based NPP:
* the accelerator with 4 L1 slices
* NPU configs MMIO at `0xd3000000`
* ARCSync MMIO at `0xd4000000`
* ARCSync has two interrupt lines (23 and 22) connected to the host CPU
* shared memory region `0x10000000 - 0x30000000`
* 1-to-1 DA-PA translation
* the remoteproc driver starts one firmware `/lib/firmware/npx-app.elf` on the L2 core (core-id - 0) of the NPX cluster (cluster-id - 0)
* the firmware linked to the `0x10000000` DA-address and occupied no more than `0x1000000` bytes
* the application helper driver reserves a region `0x20000000-0x30000000` in the accelerator shared memory for access by the firmware runtime and the host user space driver. The application helper driver receives notifications from the interrupt line 23.
```
	reserved-memory {                     <--- reserve shared memory area if needed
		#address-cells = <2>;              to exclude from the system ram
		#size-cells = <2>;
		ranges;
		reserved: buffer@0 {
			compatible = "removed-dma-pool";
			no-map;
			reg = <0x0 0x10000000 0x0 0x20000000>;
		};
	};

	npu_cfg0: npu_cfg@d3000000 {                     <--- Cluster Network config node
		reg = <0x0 0xd3000000 0x0 0xF4000>;
		snps,npu-slice-num = <4>; 
	};

	arcsync0: arcsync@d4000000 {                     <--- ARCSync node
		compatible = "snps,arcsync";
		reg = <0x0 0xd4000000 0x0 0x1000000>;
		snps,host-cluster-id = <0x2>; 
		interrupts = <23>, <22>;
	};

	snsp_accel@0 {                                  <--- Top level snps_accel node
		compatible = "snps,accel", "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		reg = <0x0 0x1000000 0x0 0x20000000>;
		range = <0x10000000 0x0 0x10000000 0x20000000>;

		remoteproc_npx0: remoteproc_npx@0x8000000 {   <--- Remoteproc node
			compatible = "snps,npx-rproc";
			reg = <0x10000000 0x1000000>; 
			firmware-name = "npx-app.elf";
			snps,npu-cfg = <&npu_cfg0>;
			snps,arcsync-ctrl = <&arcsync0>;
			snps,arcsync-cluster-id = <0x0>;
			snps,arcsync-core-id = <0x0>;
			snps,auto-boot;
		};

		app_npx0: app_npx@0 {                         <--- Application node
			compatible = "snps,accel-app";
			reg = <0x20000000 0x10000000>
			snps,arcsync-ctrl = <&arcsync0>; 
			interrupts = <23>;
		};
	};
```

### Device tree example 2

This example describes a simple configuration for the ARC host based NPP:
* the accelerator with 2 L1 slices
* NPU configs MMIO at `0xd3000000`
* ARCSync MMIO at `0xd4000000`
* ARCSync has one interrupt line connected to the host CPU - 24
* shared memory region `0x10000000 - 0x40000000`
* 1-to-1 DA-PA translation
* the remoteproc driver starts three different firmware:
  * firmware for VPX core (cluster-id 0x1, core-id 0x0) - _Deployment_vpx.elf_ has two loadable sections (base - `0x18000000` size - `0x01000000` and base - `0x30001000` size - `0x01000000`)
  * firmware for NPX L2 core (cluster-id 0x0, core-id 0x1) - _Deployment_l2.elf_ linked at base - `0x20000000 size -0x02000000`
  * firmware for NPX L1 core (cluster-id 0x0, core-id 0x1) - _Deployment_l1.elf_ linked at base - `0x10000000` size - `0x02000000`
* the application helper driver reserves a region `0x30000000-0x40000000` in the accelerator shared memory for access by the firmware runtime and the host user space driver. The application helper driver receives notifications from interrupt line 24.

```
	npu_cfg0: npu_cfg@d3000000 {
		reg = <0x0 0xd3000000 0x0 0xF4000>;
		snps,npu-slice-num = <2>;
	};

	arcsync0: arcsync@d4000000 {
		compatible = "snps,arcsync";
		reg = <0x0 0xd4000000 0x0 0x1000000>;
		snps,host-cluster-id = <0x2>;
		interrupts = <24>;
	};

	snps_accel@0 {
		compatible = "snps,accel", "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		reg = <0x0 0x1000000 0x0 0x30000000>;
		ranges = <0x10000000 0x0 0x10000000 0x30000000>;

		remoteproc_vpx0: remoteproc_vpx@0x18000000 {
			compatible = "snps,vpx-rproc";
			reg = <0x18000000 0x01000000>,
			      <0x30001000 0x01000000>;
			firmware-name = "Deployment_vpx.elf";
			snps,arcsync-ctrl = <&arcsync0>;
			snps,arcsync-core-id = <0x0>;
			snps,arcsync-cluster-id = <0x1>;
			snps,auto-boot;
		};

		remoteproc_npx0: remoteproc_npx@0x5000000 {
			compatible = "snps,npx-rproc";
			reg = <0x20000000 0x2000000>;
			firmware-name = "Deployment_l2.elf";
			snps,npu-cfg = <&npu_cfg0>;
			snps,arcsync-ctrl = <&arcsync0>;
			snps,arcsync-core-id = <0x0>;
			snps,arcsync-cluster-id = <0x0>;
			snps,auto-boot;
		};

		remoteproc_npx1: remoteproc_npx@0x8000000 {
			compatible = "snps,npx-rproc";
			reg = <0x10000000 0x2000000>;
			firmware-name = "Deployment_l1.elf";
			snps,arcsync-ctrl = <&arcsync0>;
			snps,arcsync-core-id = <0x1>;
			snps,arcsync-cluster-id = <0x0>;
			snps,auto-boot;
		};

		app_npx2: app_npx@2 {
			compatible = "snps,accel-app";
			reg = <0x30000000 0x10000000>;
			snps,arcsync-ctrl = <&arcsync0>;
			interrupts = <24>;
		};
	};
```
