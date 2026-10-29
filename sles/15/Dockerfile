ARG SLES_BASE_IMAGE=registry.suse.com/bci/bci-base:15.7
FROM --platform=linux/arm64 ${SLES_BASE_IMAGE}

ENV RUN_FW_UPDATER="no"

ARG BASE_URL
ARG BSP_VERSION
ARG SLES_TAG=sles-15

ADD bootimages bootimages/
ADD bf-rpms bf-rpms/
ADD 10-mlx-console-messages.conf /etc/sysctl.d/
ADD install.env /root/workspace/install.env/
ADD install.sh create_bfb /root/workspace/
ADD repos/ /etc/zypp/repos.d/

RUN rpm -ihv --force bootimages/mlxbf-bootimages-*.aarch64.rpm || true

# Base OS
RUN zypper --non-interactive --gpg-auto-import-keys refresh && \
    zypper --non-interactive install --no-recommends --force-resolution \
        bash-completion \
        binutils \
        chrony \
        cpio \
        curl \
        dosfstools \
        dracut \
        e2fsprogs \
        efibootmgr \
        file \
        findutils \
        gawk \
        gcc \
        glibc-locale \
        grub2 \
        grub2-arm64-efi \
        gzip \
        hostname \
        iproute2 \
        iputils \
        jq \
        kernel-default \
        kernel-default-devel \
        kexec-tools \
        kmod \
        less \
        lsof \
        lvm2 \
        make \
        mokutil \
        mstflint \
        mtools \
        nvme-cli \
        openssh \
        openssl \
        parted \
        pciutils \
        perl \
        python3 \
        rsync \
        shim \
        sudo \
        systemd \
        systemd-sysvinit \
        tar \
        tcpdump \
        timezone \
        udev \
        usbutils \
        util-linux \
        util-linux-systemd \
        vim \
        wget \
        which \
        wicked \
        xfsprogs \
        xorriso \
        xz \
        zip \
    && \
    for pkg in \
        acpid \
        audit \
        cryptsetup \
        dhcp-client \
        i2c-tools \
        ipmitool \
        kdump \
        kernel-default-extra \
        lm_sensors \
        ltrace \
        mmc-utils \
        net-tools \
        net-tools-deprecated \
        numactl \
        rasdaemon \
        sg3_utils \
        sshpass \
        sysstat \
        unzip \
        watchdog \
        xxd \
    ; do \
        zypper --non-interactive install --no-recommends $pkg || echo "WARNING: $pkg is not available"; \
    done

# DOCA from the SLES host (arm64-sbsa) repository. Kernel modules are built
# with DKMS against the SUSE kernel. The DKMS scriptlets build for `uname -r`,
# which is the build host's kernel inside docker, so point it to the image's
# kernel while the packages are installed.
RUN kver=$(/bin/ls -1 /lib/modules/ | head -1) && \
    mv /usr/bin/uname /usr/bin/uname.real && \
    printf '#!/bin/sh\nif [ "$1" = "-r" ]; then echo %s; else exec /usr/bin/uname.real "$@"; fi\n' "$kver" > /usr/bin/uname && \
    chmod 755 /usr/bin/uname && \
    zypper --non-interactive --gpg-auto-import-keys install --no-recommends --force-resolution \
        mlnx-ofa_kernel \
        mlnx-ofa_kernel-dkms \
        kernel-mft-dkms \
        mlnx-tools \
        ofed-scripts \
        doca-ofed-runtime-userspace \
        doca-runtime \
        mft \
        mft-oem \
        mft-autocomplete \
    ; rc=$? ; \
    mv -f /usr/bin/uname.real /usr/bin/uname && \
    # 107: an rpm scriptlet failed, verified by the dkms check below
    { [ $rc -eq 0 ] || [ $rc -eq 107 ]; } && \
    dkms status -k $kver && \
    dkms status -k $kver | grep -q "^mlnx-ofa_kernel/.*: installed" && \
    dkms status -k $kver | grep -q "^kernel-mft/.*: installed" && \
    zypper --non-interactive clean --all

# BlueField platform packages borrowed from the RHEL DPU repository (they are
# not part of the SLES host repository). Installed without dependency checks
# as they are el9 builds; only distro-neutral packages are used.
RUN rpm -ihv --nodeps --force bf-rpms/*.rpm && \
    rm -rf bf-rpms && \
    sed -i -E "s/_rhel-[0-9.]+_/_${SLES_TAG}_/;s/(_unsigned|_prod|_dev)/_@IMAGE_TYPE@@CUSTOM_VERSION@/;" /etc/mlnx-release

# bf-release looks for these under /usr/share/doc/mlnx-ofa_kernel*, SUSE uses /usr/share/doc/packages
RUN rule82=$(/bin/ls -1 /usr/share/doc/packages/mlnx-ofa_kernel*/82-net-setup-link.rules 2> /dev/null | head -1) && \
    if [ -n "$rule82" ]; then install -D -m 0644 $rule82 /lib/udev/rules.d/82-net-setup-link.rules; fi && \
    vf_net=$(/bin/ls -1 /usr/share/doc/packages/mlnx-ofa_kernel*/vf-net-link-name.sh 2> /dev/null | head -1) && \
    if [ -n "$vf_net" ]; then \
        install -D -m 0755 $vf_net /etc/infiniband/vf-net-link-name.sh && \
        sed -i -e 's@^PORT_NAME=$1@PORT_NAME=`echo ${1} | sed -e "s/c[[:digit:]]\\+//"`@' /etc/infiniband/vf-net-link-name.sh; \
    fi

# SUSE does not load kernel modules flagged as unsupported by default
RUN echo "allow_unsupported_modules 1" > /etc/modprobe.d/10-unsupported-modules.conf

# Network: SLES 15 uses wicked, bf-release ships RHEL ifcfg files only
ADD network/ /etc/sysconfig/network/

RUN rm -f /etc/machine-id && \
    truncate -s0 /etc/machine-id && \
    (update-pciids || true) && \
    echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PermitRootLogin yes" >> /etc/ssh/sshd_config

# Manage services
RUN systemctl enable sshd || true
RUN systemctl enable wicked || true
RUN systemctl enable chronyd || true
RUN systemctl enable openibd || true
RUN systemctl enable mst || true
RUN systemctl enable acpid || true
RUN systemctl enable bfvcheck.service || true
RUN systemctl enable bfup.service || true
RUN systemctl enable bfrtc-poll.timer || true
# Host side tools, not used on the DPU
RUN systemctl disable rshim || true
RUN systemctl disable opensmd || true
