# Notes for SLES 15

**Experimental.** NVIDIA does not publish DOCA packages for SLES built for the
BlueField DPU (`arm64-dpu`). This BFB combines:

- The SUSE kernel (`kernel-default`) and OS packages from the SUSE repositories.
  The base image defaults to the latest service pack (`bci-base:15.7`), override with
  `SLES_BASE_IMAGE`, e.g. `SLES_BASE_IMAGE=registry.suse.com/bci/bci-base:15.6` for SP6.
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
The `repos/sles.repo` file is a placeholder for the SUSE repositories (aarch64):
Basesystem, Server Applications and Development Tools modules (Pool and Updates)
of the service pack matching `SLES_BASE_IMAGE`.
They are available from SCC, RMT or SUSE Manager for a registered system.

Each `baseurl` entry in the `repos/sles.repo` file should be replaced with the
appropriate SUSE repository URL. Same goes for the `gpgkey` entries.

**The BFB image CANNOT be built without the SUSE repositories.**

Please make sure to edit the `repos/sles.repo` file accordingly before executing
the `./bfb-build sles 15` command.
