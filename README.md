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
First, let's see what container images are available locally to avoid unnecessary downloads:

```bash
# List available images via Distrobox
distrobox-list

# Or check underlying podman images (Distrobox uses podman as backend)
podman images
```

For this project, we'll use Ubuntu 24.04 from the toolbox collection:
```bash
# Using Ubuntu 24.04 toolbox image (reusing existing image to save bandwidth)
distrobox create --name yocto-dev --image quay.io/toolbx/ubuntu-toolbox:24.04

# Enter the container
distrobox enter yocto-dev
```

**Note for readers:** You can choose any Ubuntu-based image from various providers (Docker Hub, Quay.io, etc.). We recommend using Ubuntu 24.04 as it has excellent compatibility with Yocto's latest dependencies. Check your Distrobox app for available images or browse online registries.

### 1.2 Install Yocto Dependencies Inside Container
Once inside the Ubuntu 24.04 container, some package names have changed. Let's install the correct dependencies:

```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
iputils-ping python3-git python3-jinja2 libegl1-mesa-dev libsdl1.2-dev pylint \
xterm python3-subunit mesa-common-dev zstd liblz4-tool file coreutils
```

**Note:** In Ubuntu 24.04, some package names differ from older versions:
- `libegl1-mesa` → `libegl1-mesa-dev`
- `pylint3` → `pylint`

### 1.3 Create Project Workspace
```bash
mkdir -p ~/Code/yopime && cd ~/Code/yopime
```

*Project name "yopime" is a tribute to the indigenous Yopime people from the Guerrero region of Mexico.*

## Step 2: Yocto Project Setup

### 2.1 Clone Poky (Yocto Core)
```bash
cd ~/Code/yopime
git clone git://git.yoctoproject.org/poky && cd poky && git checkout scarthgap && cd ..
```

### 2.2 Clone Required Meta-Layers
```bash
cd ~/Code/yopime
# Meta-openembedded
git clone git://git.openembedded.org/meta-openembedded && cd meta-openembedded && git checkout kirkstone && cd ..

# Meta-rockchip (for Rock Pi S)
git clone https://github.com/JeffyCN/meta-rockchip.git && cd meta-rockchip && git checkout kirkstone && cd ..

# Meta-mender
git clone https://github.com/mendersoftware/meta-mender.git && cd meta-mender && git checkout kirkstone && cd ..
```

## Step 3: Build Configuration

### 3.1 Initialize Build Environment
```bash
source poky/oe-init-build-env build-rockpi-mender
```

### 3.2 Configure bblayers.conf
Edit `conf/bblayers.conf`:
```
BBLAYERS ?= " \
  /var/home/aviler/Code/yopime/poky/meta \
  /var/home/aviler/Code/yopime/poky/meta-poky \
  /var/home/aviler/Code/yopime/poky/meta-yocto-bsp \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-oe \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-python \
  /var/home/aviler/Code/yopime/meta-openembedded/meta-networking \
  /var/home/aviler/Code/yopime/meta-rockchip \
  /var/home/aviler/Code/yopime/meta-mender/meta-mender-core \
  "
```

### 3.3 Configure local.conf
Add to `conf/local.conf`:
```bash
# Machine configuration
MACHINE = "rock-pi-s"

# Mender configuration
INHERIT += "mender-full"
MENDER_ARTIFACT_NAME = "release-1"
MENDER_DEVICE_TYPE = "rock-pi-s"

# Mender storage configuration
MENDER_STORAGE_TOTAL_SIZE_MB = "8192"  # 8GB SD card
MENDER_BOOT_PART_SIZE_MB = "64"
MENDER_DATA_PART_SIZE_MB = "512"

# Network configuration
MENDER_SERVER_URL = "https://hosted.mender.io"
# or your self-hosted server

# Features
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = ""

# Image features
IMAGE_FEATURES += "ssh-server-openssh"
```

## Etapa 4: Build da Imagem

### 4.1 Build Inicial
```bash
bitbake core-image-base
```

**Status:** ⏳ Iniciando configuração...

## Próximos Passos
1. Configurar servidor Mender
2. Gerar certificados/keys
3. Criar primeira imagem OTA
4. Testar deployment

---
*Documento em construção - Atualizado em: [Data/Hora]*
