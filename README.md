# Install required packages
sudo apt-get update 
sudo apt-get install libncurses5-dev 
sudo apt-get install build-essential bc 
sudo apt-get install lbzip2
sudo apt-get install qemu-user-static

# Create build folder
mkdir $HOME/jetson_nano 
cd $HOME/jetson_nano 

# Download the following files in the jetson_nano folder:

# L4T Jetson Driver Package
https://developer.nvidia.com/embedded/dlc/r32-3-1_Release_v1.0/t210ref_release_aarch64/Tegra210_Linux_R32.3.1_aarch64.tbz2

# L4T Sample Root File System
https://developer.nvidia.com/embedded/dlc/r32-3-1_Release_v1.0/t210ref_release_aarch64/Tegra_Linux_Sample-Root-Filesystem_R32.3.1_aarch64.tbz2

# L4T Sources:
https://developer.nvidia.com/embedded/dlc/r32-3-1_Release_v1.0/Sources/T210/public_sources.tbz2

# GCC Tool Chain for 64-bit BSP
https://developer.nvidia.com/embedded/dlc/l4t-gcc-7-3-1-toolchain-64-bit

# Extract files
sudo tar xpf Tegra210_Linux_R32.3.1_aarch64.tbz2 
cd Linux_for_Tegra/rootfs/ 
sudo tar xpf ../../Tegra_Linux_Sample-Root-Filesystem_R32.3.1_aarch64.tbz2 
cd ../../ 
tar -xvf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
sudo tar -xjf public_sources.tbz2
tar -xjf Linux_for_Tegra/source/public/kernel_src.tbz2

# Apply PREEMPT-RT patches
cd kernel/kernel-4.9/ 
./scripts/rt-patch.sh apply-patches 

# Compile kernel
TEGRA_KERNEL_OUT=jetson_nano_kernel 
mkdir $TEGRA_KERNEL_OUT 
export CROSS_COMPILE=$HOME/jetson_nano/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
make ARCH=arm64 O=$TEGRA_KERNEL_OUT tegra_defconfig 
make ARCH=arm64 O=$TEGRA_KERNEL_OUT menuconfig 

# This option should already be selected:
Kernel Features -> Preemption  Model: Fully Preemptible Kernel (RT)

# You can modify other options for your kernel, like the timer frequency (or anything you need):
Kernel Features -> Timer frequency: 1000 HZ 

# After saving the configuration and exiting, start the kernel compilation
make ARCH=arm64 O=$TEGRA_KERNEL_OUT -j4 

# Copy results
sudo cp jetson_nano_kernel/arch/arm64/boot/Image $HOME/jetson_nano/Linux_for_Tegra/kernel/Image 
sudo cp -r jetson_nano_kernel/arch/arm64/boot/dts/* $HOME/jetson_nano/Linux_for_Tegra/kernel/dtb/ 
sudo make ARCH=arm64 O=$TEGRA_KERNEL_OUT modules_install INSTALL_MOD_PATH=$HOME/jetson_nano/Linux_for_Tegra/rootfs/ 
cd $HOME/jetson_nano/Linux_for_Tegra/rootfs/ 
sudo tar --owner root --group root -cjf kernel_supplements.tbz2 lib/modules 
sudo mv kernel_supplements.tbz2  ../kernel/ 

# Apply binaries
cd .. 
sudo ./apply_binaries.sh

# Generate Jetson Nano image
cd tools
sudo ./jetson-disk-image-creator.sh -o jetson_nano.img -s 14G -b jetson-nano -r 100

