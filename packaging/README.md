# FlagCX Packaging

This directory contains packaging configurations for various Linux distributions.

## Directory Structure

```
packaging/
├── debian/              # Debian/Ubuntu packaging
│   ├── control         # Package metadata
│   ├── rules           # Build rules
│   ├── changelog       # Version history
│   ├── copyright       # License information
│   └── build-helpers/  # Build scripts and Dockerfiles
│       ├── build-flagcx.sh          # Unified build script
│       ├── Dockerfile.nvidia-deb    # NVIDIA build configuration
│       └── Dockerfile.metax-deb     # MetaX build configuration
└── rpm/                # Future: RPM packaging for RHEL/Fedora/etc.
```

## Why `packaging/` Instead of Top-Level `/debian`?

Following [Debian UpstreamGuide](https://wiki.debian.org/UpstreamGuide) recommendations:

> Upstream projects should NOT include a top-level `/debian` directory.
> Use `contrib/debian/` or `packaging/debian/` instead.

**Benefits:**
- Avoids conflicts with distribution maintainers' packaging
- Clearly indicates upstream-maintained packaging
- Allows multi-format support (Debian + RPM + others)
- Industry standard (see [Miniflux](https://github.com/miniflux/v2/tree/main/packaging), etc.)

## Building Debian Packages

Use the unified build script to build packages for any backend:

### NVIDIA Backend

```bash
./packaging/debian/build-helpers/build-flagcx.sh nvidia
```

**Output:** `debian-packages/nvidia/*.deb`

### MetaX Backend

```bash
./packaging/debian/build-helpers/build-flagcx.sh metax
```

**Output:** `debian-packages/metax/*.deb`

### Advanced Usage

Specify a custom base image version:

```bash
./packaging/debian/build-helpers/build-flagcx.sh nvidia latest
./packaging/debian/build-helpers/build-flagcx.sh metax v1.2.3
```

The build script uses upstream base images:
- NVIDIA: `harbor.baai.ac.cn/flagbase/flagbase-nvidia`
- MetaX: `harbor.baai.ac.cn/flagbase/flagbase-metax`

## Installation

```bash
# NVIDIA
sudo dpkg -i debian-packages/nvidia/*.deb

# MetaX
sudo dpkg -i debian-packages/metax/*.deb
```

## CI/CD

Automated builds are triggered by:
- Push to `main` branch (when packaging files change)
- Pull requests to `main`
- Manual workflow dispatch

See `.github/workflows/build-deb.yml` for details.

### Publishing to Nexus APT Repository

Packages are uploaded to Nexus when:
- A version tag is pushed: `git tag v1.0.0 && git push origin v1.0.0`
- Manual workflow dispatch via GitHub Actions

After upload, users can install packages from the APT repository:

```bash
# Add the FlagOS APT repository
echo "deb https://resource.flagos.net/repository/flagos-apt-hosted/ flagos-apt-hosted main" | \
  sudo tee /etc/apt/sources.list.d/flagcx.list

# Update and install
sudo apt-get update
sudo apt-get install flagcx-nvidia  # or flagcx-metax
```

## Architecture

The build process uses multi-stage Docker builds:

1. **Builder stage**: Based on upstream `flagbase-nvidia` or `flagbase-metax` images
   - Contains all necessary build dependencies (CUDA/NCCL or MACA SDK)
   - Installs Debian packaging tools (`debhelper`, `dpkg-dev`, etc.)
   - Runs `dpkg-buildpackage` to create `.deb` packages

2. **Output stage**: Minimal Alpine image
   - Only contains the built `.deb` files
   - Used to extract packages to the host

This approach ensures:
- ✓ Reproducible builds using official base images
- ✓ No custom Docker images to maintain
- ✓ Clean separation of build environment and outputs

## Future Plans

- [ ] Add RPM packaging in `packaging/rpm/`
- [ ] Add Arch Linux packaging
- [x] Add APT repository hosting (Nexus)
