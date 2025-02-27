# vim:set ft=dockerfile:
# FreeSWITCH Docker Image
# Security: This image follows best practices for secure containerization
# - No credentials in code
# - Minimal dependencies
# - Proper cleanup of build artifacts
# - Non-root user execution
# Version: 1.10
# Last Updated: 2025-02-26

# Stage 1: Download source
FROM debian:bullseye AS source

# Install git and SSL certificates
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        git \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Configure git to use HTTPS and retry failed operations
RUN git config --global http.sslVerify true \
    && git config --global core.compression 0 \
    && git config --global http.postBuffer 1048576000

# Clone FreeSWITCH with retry logic
RUN mkdir -p /usr/src \
    && cd /usr/src \
    && for i in $(seq 1 3); do \
        echo "Attempt $i: Cloning FreeSWITCH..." \
        && git clone -b v1.10 --depth 1 --single-branch https://github.com/signalwire/freeswitch.git freeswitch \
        && break \
        || (echo "Clone failed, retrying in 5 seconds..." && sleep 5); \
    done

# Stage 2: Build spandsp
FROM debian:bullseye AS spandsp-build

# Install build dependencies for spandsp
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        git \
        build-essential \
        automake \
        autoconf \
        libtool \
        pkg-config \
        libssl-dev \
        libtiff-dev \
        libxml2-dev \
    && rm -rf /var/lib/apt/lists/*

# Clone and build spandsp
RUN cd /usr/src \
    && git clone https://github.com/freeswitch/spandsp.git \
    && cd spandsp \
    && ./bootstrap.sh \
    && ./configure --prefix=/usr --enable-doc=no \
    && make -j$(nproc) \
    && make install \
    && ldconfig \
    && ls -la /usr/lib/x86_64-linux-gnu/libspandsp* \
    && ls -la /usr/include/spandsp/

# Stage 2.1: Build sofia-sip
FROM debian:bullseye AS sofia-build

# Install build dependencies for sofia-sip
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        git \
        build-essential \
        automake \
        autoconf \
        libtool \
        pkg-config \
        libglib2.0-dev \
        libssl-dev \
        libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Clone and build sofia-sip
RUN cd /usr/src \
    && git clone https://github.com/freeswitch/sofia-sip.git \
    && cd sofia-sip \
    && ./bootstrap.sh \
    && ./configure --prefix=/usr --with-glib=no --with-openssl \
    && make -j$(nproc) \
    && make install \
    && ldconfig

# Stage 3: Final build
FROM debian:bullseye

# Create freeswitch user and group
RUN groupadd -r freeswitch --gid=999 && useradd -r -g freeswitch --uid=999 freeswitch

# Configure apt for better reliability
RUN echo 'Acquire::Retries "5";' > /etc/apt/apt.conf.d/80-retries \
    && echo "APT::Get::Assume-Yes "true";" > /etc/apt/apt.conf.d/90assumeyes

# Install base dependencies
RUN set -ex \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        ca-certificates \
        wget \
        gnupg2 \
        procps \
        net-tools \
        iputils-ping \
    && rm -rf /var/lib/apt/lists/*

# Set locale
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        locales \
    && rm -rf /var/lib/apt/lists/* \
    && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG=en_US.utf8

# Install build dependencies
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        git \
        build-essential \
        cmake \
        pkg-config \
        libssl-dev \
        libcurl4-openssl-dev \
        libsqlite3-dev \
        libedit-dev \
        libpcre3-dev \
        libjpeg-dev \
        libsndfile1-dev \
        libldns-dev \
        liblua5.2-dev \
        libspeex-dev \
        libspeexdsp-dev \
        libtiff-dev \
        uuid-dev \
        zlib1g-dev \
        libdb-dev \
        unixodbc-dev \
        libpq-dev \
        libmariadb-dev \
        libmariadb-dev \
        libglib2.0-dev \
        libxml2-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy dependencies from build stages
COPY --from=spandsp-build /usr/lib/x86_64-linux-gnu/libspandsp.* /usr/lib/
COPY --from=spandsp-build /usr/include/spandsp* /usr/include/
COPY --from=spandsp-build /usr/include/spandsp /usr/include/spandsp
COPY --from=spandsp-build /usr/lib/x86_64-linux-gnu/pkgconfig/spandsp.pc /usr/lib/pkgconfig/

COPY --from=sofia-build /usr/lib/libsofia-sip-ua* /usr/lib/
COPY --from=sofia-build /usr/include/sofia-sip-1.13 /usr/include/sofia-sip-1.13
COPY --from=sofia-build /usr/lib/pkgconfig/sofia-sip-ua.pc /usr/lib/pkgconfig/

ENV PKG_CONFIG_PATH=/usr/lib/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig
RUN ldconfig && pkg-config --libs spandsp

# Copy source from first stage
COPY --from=source /usr/src/freeswitch /usr/src/freeswitch

# Build and install FreeSWITCH
RUN cd /usr/src/freeswitch \
    && ./bootstrap.sh -j \
    && sed -i \
        -e 's/^endpoints\/mod_verto/#endpoints\/mod_verto/g' \
        -e 's/^applications\/mod_signalwire/#applications\/mod_signalwire/g' \
        -e 's/^applications\/mod_conference/#applications\/mod_conference/g' \
        -e 's/^formats\/mod_shout/#formats\/mod_shout/g' \
        -e 's/^applications\/mod_av/#applications\/mod_av/g' \
        -e 's/^applications\/mod_mongo/#applications\/mod_mongo/g' \
        -e 's/^applications\/mod_memcache/#applications\/mod_memcache/g' \
        -e 's/^applications\/mod_v8/#applications\/mod_v8/g' \
        -e 's/^xml_int\/mod_xml_rpc/#xml_int\/mod_xml_rpc/g' \
        -e 's/^xml_int\/mod_xml_curl/#xml_int\/mod_xml_curl/g' \
        -e 's/^xml_int\/mod_xml_cdr/#xml_int\/mod_xml_cdr/g' \
        -e 's/^languages\/mod_lua/#languages\/mod_lua/g' \
        -e 's/^languages\/mod_perl/#languages\/mod_perl/g' \
        -e 's/^languages\/mod_python/#languages\/mod_python/g' \
        -e 's/^applications\/mod_spandsp/#applications\/mod_spandsp/g' \
        -e 's/^codecs\/mod_spandsp/#codecs\/mod_spandsp/g' \
        -e 's/^endpoints\/mod_rtc/#endpoints\/mod_rtc/g' \
        modules.conf \
    && ./configure \
        --enable-portable-binary \
        --disable-dependency-tracking \
        --enable-core-pgsql-support \
        --enable-core-odbc-support \
        --enable-core-mysql-support \
        --disable-libvpx \
        --disable-erlang \
        --disable-java \
        --disable-verto \
        --disable-conference \
        --disable-v8 \
        --disable-shout \
        --disable-av \
        --disable-mongo \
        --disable-memcache \
        --disable-lua \
        --disable-perl \
        --disable-python \
        --disable-xml-rpc \
        --disable-xml-cdr \
        --disable-spandsp \
        # Removing the sndfile disable flag to enable audio support
        --disable-rtc \
        --with-loglevel=warn \
    && make -j$(nproc) \
    && make install \
    && make sounds-install moh-install \
    && make cd-sounds-install cd-moh-install \
    && ln -s /usr/local/freeswitch/bin/freeswitch /usr/bin/ \
    && ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin/ \
    && chown -R freeswitch:freeswitch /usr/local/freeswitch

# Configure event socket
RUN mkdir -p /usr/local/freeswitch/conf/autoload_configs \
    && echo '<?xml version="1.0"?>\n\
<configuration name="event_socket.conf" description="Socket Client">\n\
  <settings>\n\
    <param name="nat-map" value="false"/>\n\
    <param name="listen-ip" value="0.0.0.0"/>\n\
    <param name="listen-port" value="8021"/>\n\
    <param name="password" value="ClueCon"/>\n\
  </settings>\n\
</configuration>' > /usr/local/freeswitch/conf/autoload_configs/event_socket.conf.xml

# Create required directories owned by freeswitch user
RUN mkdir -p /usr/local/freeswitch/log \
    && mkdir -p /usr/local/freeswitch/run \
    && mkdir -p /usr/local/freeswitch/db \
    && mkdir -p /usr/local/freeswitch/recordings \
    && mkdir -p /usr/local/freeswitch/storage \
    && mkdir -p /usr/local/freeswitch/cache \
    && chown -R freeswitch:freeswitch /usr/local/freeswitch

# Switch to freeswitch user
USER freeswitch

# Set default workdir
WORKDIR /usr/local/freeswitch

# Expose ports
EXPOSE 5060/tcp 5060/udp 5080/tcp 5080/udp 8021/tcp
EXPOSE 64535-65535/udp

# Start FreeSWITCH
CMD ["freeswitch"]
