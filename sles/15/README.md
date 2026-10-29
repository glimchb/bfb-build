# Notes for SLES 15

**Experimental.** NVIDIA does not publish DOCA packages for SLES built for the
BlueField DPU (`arm64-dpu`). This BFB combines:

- The SUSE kernel (`kernel-default`) and OS packages from the SUSE repositories.
  The base image is `bci-base:15.6` (SP6), override with `SLES_BASE_IMAGE`, e.g.
  `SLES_BASE_IMAGE=registry.suse.com/bci/bci-base:15.7`, together with matching repositories.
- DOCA/OFED from the SLES host repository (`DOCA_REPO_URL`, default
  `https://linux.mellanox.com/public/repo/doca/<DOCA_VERSION>/sles15/arm64-sbsa`):
  `mlnx-ofa_kernel-dkms` and `kernel-mft-dkms` are built with DKMS against the SUSE
  kernel during the docker build, plus `doca-runtime`, `doca-ofed-runtime-userspace`
  and MFT.
- The BlueField platform packages, which are not part of the host repository, from
  the RHEL DOCA DPU repository (`DOCA_DPU_DISTRO`, default `rhel9.6`): mlxbf-bootimages,
  mlxbf-bfscripts (mlx-mkbfb, bfcfg, ...), mlxbf-bootctl, bf-release,
  mlnx-fw-updater-signed (BlueField NIC firmware), BMC/CEC firmware.
  They are el9 builds installed with `rpm --nodeps`.

### Limitations

- DOCA packages are the host builds. DPU specific packages (e.g. mlnx-snap,
  doca-devel for the DPU) are not available.
- The BlueField platform drivers come from the SUSE kernel (`mlxbf_tmfifo`, `mlxbf-gige`,
  `mlxbf-bootctl`, `sdhci-of-dwcmshc`, ...). Drivers missing from the SUSE kernel are skipped.
- DKMS modules are not signed. With UEFI secure boot enabled on the DPU they will not
  load unless their signing key is enrolled (MOK).
- Building the DKMS modules under qemu (x86_64 build server) takes a long time.
- Management interfaces (`oob_net0` DHCP, `tmfifo_net0` 192.168.100.2/30) are
  configured with wicked under `/etc/sysconfig/network/`.
- Root file system is XFS (not the SUSE default Btrfs).
- Default root password is `sles`.

## Repositories

#### sles.repo
By default `repos/sles.repo` uses the openSUSE Leap 15.6 (aarch64) repositories, which
need no registration, so `./bfb-build sles 15` works out of the box. Leap only provides
the OS packages; the result is not a supported SLES image.

For a SLES build, replace the entries in `repos/sles.repo` with the SLES 15 SP6 Basesystem, Server Applications and Development Tools
module repositories (Pool and Updates) for aarch64
from SCC, RMT or SUSE Manager.

#### Corporate proxy
When HTTPS traffic goes through a TLS intercepting proxy, pass its CA certificates with
`EXTRA_CA_CERTS=<file or directory>`. They are trusted only while building the image and
are not included in the BFB.
