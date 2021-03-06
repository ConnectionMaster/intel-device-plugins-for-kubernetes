# CLEAR_LINUX_BASE and CLEAR_LINUX_VERSION can be used to make the build
# reproducible by choosing an image by its hash and installing an OS version
# with --version=:
# CLEAR_LINUX_BASE=clearlinux@sha256:b8e5d3b2576eb6d868f8d52e401f678c873264d349e469637f98ee2adf7b33d4
# CLEAR_LINUX_VERSION="--version=29970"
#
# This is used on release branches before tagging a stable version.
# The main branch defaults to using the latest Clear Linux.
ARG CLEAR_LINUX_BASE=clearlinux:latest

FROM ${CLEAR_LINUX_BASE} as builder

ARG CLEAR_LINUX_VERSION=

RUN swupd update --no-boot-update ${CLEAR_LINUX_VERSION} && \
    swupd bundle-add wget c-basic devpkg-json-c devpkg-util-linux devpkg-hwloc devpkg-tbb git
# Fetch dependencies and source code
ARG OPAE_RELEASE=1.5.0-2

# workaround for a swupd failure discussed in https://github.com/clearlinux/distribution/issues/831
RUN ldconfig
RUN mkdir -p /usr/src/opae && \
    cd /usr/src/opae && \
    wget -q https://github.com/OPAE/opae-sdk/archive/${OPAE_RELEASE}.tar.gz -O- | tar -zx

# Build OPAE
RUN cd /usr/src/opae/opae-sdk-${OPAE_RELEASE} && \
    mkdir build && \
    cd build && \
    cmake -DCMAKE_BUILD_TYPE=Release -DOPAE_BUILD_LIBOPAE_PY=OFF .. && \
    make xfpga nlb0 nlb3

# Install clean os-core and libstdcpp bundle in target directory
RUN mkdir /install_root \
    && swupd os-install  \
    ${CLEAR_LINUX_VERSION} \
    --path /install_root --statedir /swupd-state \
    --bundles=libstdcpp \
    --no-boot-update \
    && rm -rf /install_root/var/lib/swupd/*

# Minimal result image
FROM scratch as final
ENV PATH="/usr/local/bin/:${PATH}"
COPY --from=builder /install_root /

# OPAE
# nlb0 and nlb3 examples
COPY --from=builder /usr/src/opae/opae-sdk-*/build/bin/nlb* /usr/local/bin/
# libxfpga.so, libopae-c*.so*
COPY --from=builder /usr/src/opae/opae-sdk-*/build/lib/lib*.so* /usr/local/lib64/

RUN echo /usr/local/lib64/ >> /etc/ld.so.conf && ldconfig

COPY test_fpga.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/test_fpga.sh"]
