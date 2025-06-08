# 25-06-08-teste-blog

# Building Yocto + Mender for Rock Pi S: Complete OTA System Updates

## Project Overview
Creating a Yocto project for Rock Pi S with Mender.io integration for full Linux system OTA updates.

## Hardware & Environment Setup
- **Development Machine:** Lenovo laptop with AMD Ryzen 7 4000 series + AMD Radeon Graphics
- **Host OS:** Bluefin Project (Fedora 42 Silverblue with GNOME 42)
- **Build Environment:** Distrobox container for isolation
- **Target:** Rock Pi S (RK3308 SoC)
- **Storage Requirements:** 50GB+ free space
- **RAM:** 8GB+ recommended

## Why Distrobox for Yocto Builds?
Using Bluefin (immutable OS) provides excellent isolation between the base system and development environments. Distrobox allows us to create a traditional mutable Linux environment specifically for Yocto builds without affecting the host system.

## Step 1: Setting up Distrobox for Yocto Development

### 1.1 Choose Your Container Image
For this project, we'll use Ubuntu 24.04:
``` bash
distrobox create --name yocto-dev --image quay.io/toolbx/ubuntu-toolbox:24.04
distrobox enter yocto-dev
```

### 1.2 Install Yocto Dependencies Inside Container
``` bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
iputils-ping python3-git python3-jinja2 libegl1-mesa-dev libsdl1.2-dev pylint \
xterm python3-subunit mesa-common-dev zstd liblz4-tool file coreutils
```

### 1.3 Create Project Workspace
``` bash
mkdir -p ~/Code/yopime && cd ~/Code/yopime
```

*Project name "yopime" is a tribute to the indigenous Yopime people from the Guerrero region of Mexico.*

## Step 2: Yocto Project Setup

### 2.1 Clone Poky (Yocto Core)
``` bash
cd ~/Code/yopime
git clone git://git.yoctoproject.org/poky && cd poky && git checkout scarthgap && cd ..
```

### 2.2 Clone Required Meta-Layers
``` bash
cd ~/Code/yopime
git clone git://git.openembedded.org/meta-openembedded && cd meta-openembedded && git checkout scarthgap && cd ..
git clone https://github.com/JeffyCN/meta-rockchip.git && cd meta-rockchip && git checkout scarthgap && cd ..
git clone https://github.com/mendersoftware/meta-mender.git && cd meta-mender && git checkout scarthgap && cd ..
```

### 2.3 Create a Custom Meta-Layer
It's a best practice to create a custom layer for our project-specific changes. This keeps the project clean and manageable.
``` bash
# First, source the build environment to get access to helper scripts
source poky/oe-init-build-env build

# Now, create our custom layer
bitbake-layers create-layer ../meta-yopime
```

## Step 3: Build Configuration

### 3.1 Initialize Build Environment
If you haven't already, source the build environment script. This will move you into the `build` directory.
``` bash
cd ~/Code/yopime
source poky/oe-init-build-env build
```

### 3.2 Configure bblayers.conf
Add our new `meta-yopime` layer to `conf/bblayers.conf`. Replace the entire content of the file with the following:
``` bash
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /var/home/aviler/Code/yopime/poky/meta \
  /var/home/aviler/Code/yopime/poky/meta-poky \
  /var/home/aviler/Code/yopime/poky/meta-yocto-bsp \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-oe \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-python \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-networking \
  /var/home/aviler/Code/yopime/meta-rockchip \
  /var/home/aviler/Code/yopime/meta-mender/meta-mender-core \
  /var/home/aviler/Code/yopime/meta-yopime \
  "
```

### 3.3 Configure local.conf
Add the following lines to the end of your `conf/local.conf` file. The comments highlight common issues and their solutions.
``` bash
# Machine configuration (for Yocto build)
# Note: The machine name for Rock Pi S in meta-rockchip (scarthgap) is 'rockchip-rk3308-evb', not 'rock-pi-s'.
MACHINE = "rockchip-rk3308-evb"

# Mender configuration
INHERIT += "mender-full"
MENDER_FEATURES_ENABLE:append = " mender-uboot"
MENDER_ARTIFACT_NAME = "release-1"
# Device identifier for the Mender server. We use 'yopime' for our project.
MENDER_DEVICE_TYPE = "yopime"

# Mender storage configuration for a 32GB SD card
# Note: Yocto's parser requires comments to be on separate lines from variable assignments.
MENDER_STORAGE_TOTAL_SIZE_MB = "32768"
MENDER_BOOT_PART_SIZE_MB = "64"
MENDER_DATA_PART_SIZE_MB = "512"

# Network configuration
MENDER_SERVER_URL = "https://hosted.mender.io"

# Features: Enable systemd as the init manager
# Note: Explicitly set all three systemd variables to avoid common build errors with mender-systemd.
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"
INIT_MANAGER = "systemd"
VIRTUAL-RUNTIME_initscripts = ""

# Image features
IMAGE_FEATURES += "ssh-server-openssh"
```

### 3.4 Patching U-Boot for Mender Compatibility
During the build, you might encounter an error: `ERROR: Nothing PROVIDES 'u-boot'`. This happens because `meta-mender` depends on a recipe named `u-boot`, but `meta-rockchip` provides it as `u-boot-rockchip`. We fix this by creating a `.bbappend` file in our custom layer to tell Yocto that `u-boot-rockchip` also provides `u-boot`.

Create the required directory structure and the `.bbappend` file:
``` bash
mkdir -p ../meta-yopime/recipes-bsp/u-boot
touch ../meta-yopime/recipes-bsp/u-boot/u-boot-rockchip.bbappend
```

Now, add the following content to the newly created `u-boot-rockchip.bbappend` file:
```
# Let the build system know that this recipe provides the 'u-boot' dependency
# required by the meta-mender layer.
PROVIDES += "u-boot"
RPROVIDES_${PN} += "u-boot"
```

## Step 4: Build the Image
With all configurations and patches in place, you are ready to build the image.
``` bash
bitbake core-image-base
```
**Status:** ✅ Build successful!

## Next Steps
1. Flash the generated `.sdimg` to an SD card. The image will be located at `tmp/deploy/images/rockchip-rk3308-evb/core-image-base-rockchip-rk3308-evb.sdimg`.
2. Boot the device and accept it in the Mender UI.
3. Create a second release and deploy an OTA update.