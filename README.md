# apt-deb-collect

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Shell: Bash](https://img.shields.io/badge/Shell-Bash-121011?logo=gnu-bash)](#)
[![OS: Ubuntu%20%7C%20Debian](https://img.shields.io/badge/OS-Ubuntu%20%7C%20Debian-E95420?logo=ubuntu)](#)

`apt-deb-collect` is a small utility for building an offline `.deb` bundle from Ubuntu/Debian package metadata.

## Quick start

Resolve packages only:

```bash
./apt-deb-collect --dry-run nvidia-container-toolkit
```

Download a recursive offline bundle:

```bash
./apt-deb-collect --out /tmp/deb-bundle nvidia-container-toolkit
```

Download for a different target architecture:

```bash
./apt-deb-collect --arch arm64 --out /tmp/deb-bundle nvidia-container-toolkit
```

## Why this script exists

The original need was air-gapped installation work. Downloading a single `.deb` is not enough because package installation often fails later on transitive dependencies.

This script was written to solve three specific problems:

- Follow dependencies recursively instead of stopping at one depth.
- Resolve virtual packages such as `<debconf-2.0>` to a concrete provider.
- Allow an explicit target architecture such as `amd64` or `arm64`.

## How it works

The script does a breadth-first traversal over package dependencies.

1. Start from one or more root packages.
2. Read direct `Depends` and `PreDepends` using `apt-cache depends`.
3. For normal packages, enqueue them directly.
4. For virtual packages, choose one provider and enqueue that provider.
5. Keep a visited set to avoid loops and duplicate work.
6. Write the resolved package list to `pkgs.txt`.
7. Download every resolved package with `apt download`.
8. If `dpkg-scanpackages` exists, generate `Packages.gz` for local repository use.

Provider selection for virtual packages uses this order:

- An installed provider on the current host
- The only available provider, if there is exactly one
- The first concrete provider visible in apt metadata

## Output structure

For each requested root package, the script creates one directory:

```text
<out>/<root-package>/
  pkgs.txt
  providers.txt
  failed.txt        # only if some downloads fail
  packages/
    *.deb
    Packages.gz
```

## Usage

Show help:

```bash
./apt-deb-collect --help
```

Dry-run only:

```bash
./apt-deb-collect --dry-run nvidia-container-toolkit
```

Download for the native architecture:

```bash
./apt-deb-collect nvidia-container-toolkit
```

Download for a target architecture:

```bash
./apt-deb-collect --arch amd64 nvidia-container-toolkit
./apt-deb-collect --arch arm64 nvidia-container-toolkit
```

Write output under a specific directory:

```bash
./apt-deb-collect --out /tmp/deb-bundle nvidia-container-toolkit
```

Collect multiple roots at once:

```bash
./apt-deb-collect --out /tmp/deb-bundle \
  nvidia-container-toolkit \
  containerd
```

## Target architecture notes

Yes, target architecture can be passed as an argument:

```bash
./apt-deb-collect --arch <arch> <package>
```

Examples:

- `amd64`
- `arm64`

If the target architecture is not enabled in `dpkg`, the script fails early and prints the required command:

```bash
sudo dpkg --add-architecture <arch>
sudo apt-get update
```

Without that, apt metadata for the foreign architecture may not exist locally, and downloads can fail or resolve incorrectly.

## Limitations

- The script follows `Depends` and `PreDepends`. It does not collect `Recommends` or `Suggests`.
- Virtual package provider choice is heuristic. For packages with many valid providers, you may want to inspect `providers.txt`.
- Cross-architecture collection depends on the host having that architecture enabled in apt/dpkg metadata.
- Some packages may still fail if their repository is not enabled on the collecting host.
