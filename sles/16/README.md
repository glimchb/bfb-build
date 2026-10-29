# Notes for SLES 16

**Experimental.** NVIDIA does not publish DOCA packages for SLES built for the
BlueField DPU (`arm64-dpu`). This BFB combines:

- The SUSE kernel (`kernel-default`) and OS packages from the SUSE repositories
  (base image `bci-base:16.0`, override with `SLES_BASE_IMAGE`).
- DOCA/OFED from the SLES host repository (`DOCA_REPO_URL`, default
  `https://linux.mellanox.com/public/repo/doca/<DOCA_VERSION>/sles16/arm64-sbsa`):
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
  configured with NetworkManager keyfiles under `/etc/NetworkManager/system-connections/`.
- Root file system is XFS (not the SUSE default Btrfs).
- Default root password is `sles`.

## Repositories

#### sles.repo
By default `repos/sles.repo` uses the openSUSE Leap 16.0 (aarch64) repositories, which
need no registration, so `./bfb-build sles 16` works out of the box. Leap only provides
the OS packages; the result is not a supported SLES image.

For a SLES build, replace the entries in `repos/sles.repo` with the SLES 16.0 product repositories (Pool and Updates) for aarch64
from SCC, RMT or SUSE Manager.

#### Corporate proxy
When HTTPS traffic goes through a TLS intercepting proxy, pass its CA certificates with
`EXTRA_CA_CERTS=<file or directory>`. They are trusted only while building the image and
are not included in the BFB.
