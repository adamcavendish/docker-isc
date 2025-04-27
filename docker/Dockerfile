FROM ubuntu:noble AS base

# =========================
# Build Stage
# =========================
FROM base AS builder
ARG KEA_VERSION

# Build dependencies.
RUN apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y \
        apt-transport-https \
        gnupg2 \
        build-essential \
        curl \
        libboost-system-dev \
        libkrb5-dev \
        liblog4cplus-dev \
        libssl-dev \
        postgresql-server-dev-all

# Download and verify the source code.
WORKDIR /data
RUN curl -LOR "https://ftp.isc.org/isc/kea/${KEA_VERSION}/kea-${KEA_VERSION}.tar.gz{,.asc}" && \
    install -m 0700 -o root -g root -d /root/.gnupg && \
    curl -L "https://www.isc.org/docs/isc-keyblock.asc" | gpg2 --import && \
    gpg2 --no-options --verbose --keyid-format 0xlong \
        --keyserver-options auto-key-retrieve=true \
        --verify "kea-${KEA_VERSION}.tar.gz.asc" \
        "kea-${KEA_VERSION}.tar.gz" && \
    tar xzpf "kea-${KEA_VERSION}.tar.gz"
    
# Build with PostgreSQL flag enabled.
WORKDIR /data/kea-${KEA_VERSION}
RUN ./configure --with-openssl --with-pgsql --with-gssapi --enable-static=no
RUN make -j$(nproc)
RUN make install

# Clean up unused files.
RUN cd /usr/local/lib/ && \
    rm -v *.la && \
    rm -v kea/hooks/*.la

# Strip debug symbols to reduce file size of binaries
RUN find /usr/local/sbin/ /usr/local/lib/ -type f -exec strip --strip-unneeded {} \;

# Separate the hooks directory to reduce image size.
RUN mv "/usr/local/lib/kea/hooks" / && mkdir "/usr/local/lib/kea/hooks"


# =========================
# Common Runtime Layer
# =========================
FROM base AS common
ARG KEA_USER=kea

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
        adduser \
        libboost-system1.83.0 \
        libkrb5-3 \
        liblog4cplus-2.0.5t64 \
        libpq5 \
        libssl3t64 && \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/* && \
    addgroup --system --gid 101 ${KEA_USER} && \
    adduser --system --disabled-login --ingroup ${KEA_USER} --no-create-home --gecos "Kea user" --shell /bin/false --uid 101 ${KEA_USER} && \
    install -m 0775 -o ${KEA_USER} -g ${KEA_USER} -d \
        /opt/kea \
        /usr/local/var/log/kea \
        /usr/local/var/lib/kea \
        /usr/local/var/run/kea \
    install -m 0775 -o ${KEA_USER} -g ${KEA_USER} -d \
        /kea \
        /kea/config \
        /kea/leases \
        /kea/logs \
        /kea/sockets && \
    mkdir /entrypoint.d

# From the build stage we copy the library files from "lib", and then all the
# C++ header files in the "include" folder. There are a couple of folders used
# during runtime in "var" as well, but we created them above with the Kea user
# as the owner instead.
COPY --from=builder /usr/local/lib /usr/local/lib
COPY --from=builder /usr/local/include/kea /usr/local/include/kea

# =========================
# Runtime with Hooks
# =========================
FROM common AS common-hooks
COPY --from=builder /hooks/*.so /usr/local/lib/kea/hooks/
RUN echo "/usr/local/lib/kea/hooks" > /etc/ld.so.conf.d/kea.conf
RUN ldconfig

# =========================
# DHCP4
# =========================
FROM common-hooks AS kea-dhcp4
COPY --from=builder /usr/local/sbin/kea-dhcp4 /usr/local/sbin/kea-lfc /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/kea-dhcp4"]

# =========================
# DHCP6
# =========================
FROM common-hooks AS kea-dhcp6
COPY --from=builder /usr/local/sbin/kea-dhcp6 /usr/local/sbin/kea-lfc /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/kea-dhcp6"]

# =========================
# Kea Control Agent
# =========================
FROM common AS kea-ctrl-agent
COPY --from=builder /usr/local/sbin/kea-ctrl-agent /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/kea-ctrl-agent"]

# =========================
# Kea DHCP DDNS
# =========================
FROM common AS kea-dhcp-ddns
COPY --from=builder /usr/local/sbin/kea-dhcp-ddns /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/kea-dhcp-ddns"]
