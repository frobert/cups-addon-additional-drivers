FROM ghcr.io/home-assistant/base:latest

# Add env
ENV LANG=C.UTF-8
# Set CUPS' persistent configuration directory to the Supervisor-managed
# app config folder. Cache, logs, state and the spool stay in the container.
ENV CUPS_SERVERROOT=/config

# Setup base
# Note: foomatic-db and foomatic-db-ppds were removed from Alpine 3.23 and are
# no longer available in stable repositories. hplip is available again
# (Alpine 3.24) and is installed below together with its PPD set.
# Alpine splits this: `gutenprint` is libraries/tools only; `gutenprint-cups`
# ships rastertogutenprint and the CUPS PPDs (Canon, HP, Brother, etc.).
# `hplip` + `hplip-ppd` add HP's own driver: the hpcups filter, the `hp`
# backend and ~3000 PPDs for HP LaserJet, OfficeJet, DeskJet and Envy models
# (installed under /usr/share/ppd/HP, which cups-driverd scans).
RUN \
    echo "http://dl-cdn.alpinelinux.org/alpine/v$(cat /etc/alpine-release | cut -d'.' -f1,2)/main" > /etc/apk/repositories && \
    echo "http://dl-cdn.alpinelinux.org/alpine/v$(cat /etc/alpine-release | cut -d'.' -f1,2)/community" >> /etc/apk/repositories && \
    apk update && \
    apk add --no-cache \
        cups \
        cups-filters \
        ghostscript \
        libjpeg-turbo \
        net-snmp \
        epson-inkjet-printer-escpr \
        gutenprint \
        gutenprint-cups \
        hplip \
        hplip-ppd \
        avahi \
        dbus \
        gcompat \
        libstdc++ \
        dpkg \
        jq

# Build the open-source rastertokpsl filter for Kyocera GDI (KPSL) printers
# (FS-1040/1041, FS-1060DN/1061DN, FS-1020/1025/1120/1125MFP). Their PPDs
# reference /usr/lib/cups/filter/rastertokpsl; the official filter is
# proprietary and not available in the Alpine repositories, so compile the
# vendored Apache-2.0 re-implementation (sv99/rastertokpsl-re).
COPY rastertokpsl /tmp/rastertokpsl
RUN \
    apk add --no-cache --virtual .rastertokpsl-build \
        gcc \
        musl-dev \
        cups-dev && \
    gcc -O2 -o /usr/lib/cups/filter/rastertokpsl \
        /tmp/rastertokpsl/main.c \
        /tmp/rastertokpsl/rastertokpsl.c \
        /tmp/rastertokpsl/halfton.c \
        /tmp/rastertokpsl/libjbig/jbig.c \
        /tmp/rastertokpsl/libjbig/jbig_ar.c \
        /tmp/rastertokpsl/unicode/ConvertUTF.c \
        -I/tmp/rastertokpsl \
        -lcups -lm && \
    rm -rf /tmp/rastertokpsl && \
    apk del .rastertokpsl-build

# Build and install the Dymo CUPS drivers (LabelWriter/LabelManager). The
# Dymo PPDs (e.g. lw4xl.ppd for the LabelWriter 4XL) reference
# /usr/lib/cups/filter/raster2dymolw; the filter is proprietary and not
# available in the Alpine repositories, so compile the vendored GPLv2
# dymo-cups-drivers source and install it (filters raster2dymolw and
# raster2dymolm plus the official Dymo PPDs).
COPY dymo-cups-drivers /tmp/dymo-cups-drivers
RUN \
    apk add --no-cache --virtual .dymo-build \
        g++ \
        make \
        cups-dev && \
    cd /tmp/dymo-cups-drivers && \
    find . -exec touch -t 202001010000 {} + && \
    ./configure && \
    make && \
    make install && \
    cd / && \
    rm -rf /tmp/dymo-cups-drivers && \
    apk del .dymo-build

# Build and install brlaser, the open-source CUPS driver for Brother
# host-based (GDI) laser printers such as the HL-2130, HL-2140, HL-1110,
# DCP-7030 and many more. These printers speak neither PCL nor PostScript,
# so Gutenprint cannot drive them, and Brother's own Linux driver is an
# i386/glibc binary that does not run on Alpine. The vendored source is
# Owl-Maintain/brlaser v6.2.8 (GPL-2.0). Install paths are pinned explicitly
# so the driver lands in the image regardless of the CUPS_* environment.
COPY brlaser /tmp/brlaser
RUN \
    apk add --no-cache --virtual .brlaser-build \
        cmake \
        g++ \
        make \
        cups-dev && \
    cmake -S /tmp/brlaser -B /tmp/brlaser/build \
        -DCMAKE_BUILD_TYPE=Release \
        -DCUPS_DATA_DIR=/usr/share/cups \
        -DCUPS_SERVER_BIN=/usr/lib/cups && \
    cmake --build /tmp/brlaser/build && \
    cmake --install /tmp/brlaser/build && \
    rm -rf /tmp/brlaser && \
    apk del .brlaser-build

# Canon BJNP network backend (port 8611): build from Debian's published
# cups-bjnp 2.0.3 source. The checksum protects against unexpected changes.
RUN \
    apk add --no-cache --virtual .bjnp-build \
        build-base \
        cups-dev \
        curl \
        ca-certificates && \
    mkdir -p /tmp/cups-bjnp && \
    curl --fail --location --retry 3 \
      -o /tmp/cups-bjnp.tar.gz \
      https://deb.debian.org/debian/pool/main/c/cups-bjnp/cups-bjnp_2.0.3.orig.tar.gz && \
    echo "c914b2fd9d8e26ce22f2df6244d9defeec31ee9a5360f723eecb3b00f2168500  /tmp/cups-bjnp.tar.gz" | sha256sum -c - && \
    tar -xzf /tmp/cups-bjnp.tar.gz -C /tmp/cups-bjnp --strip-components=1 && \
    cd /tmp/cups-bjnp && \
    ./configure --prefix=/usr \
      --with-cupsbackenddir=/usr/lib/cups/backend \
      --disable-Werror && \
    make && \
    install -m 755 bjnp /usr/lib/cups/backend/bjnp && \
    test -x /usr/lib/cups/backend/bjnp && \
    cd / && \
    rm -rf /tmp/cups-bjnp /tmp/cups-bjnp.tar.gz && \
    apk del .bjnp-build

# SpliX: CUPS driver for Samsung, Xerox, Dell, Lexmark and Toshiba host-based
# lasers that speak SPL/QPDL instead of PCL or PostScript (e.g. Samsung
# ML-1510, ML-2010, ML-2240, ML-2160, SCX-4200, CLP-300, CLX-3160, Xerox
# Phaser 3117/6110). Alpine's own `splix` package is built with
# DISABLE_JBIG=1, which breaks the color CLP/CLX models and the QPDL v3
# models (ML-2160/2165, ML-3310) with "J-BIG compression algorithm has been
# disabled". Build OpenPrinting SpliX 2.0.1 instead, linked statically against
# libjbig85 from JBIG-KIT 2.1 (not in Alpine stable). Both archives are pinned
# by SHA-256. The ML-2160/2165 and ML-3310 PPDs ship in the tarball root
# instead of ppd/, so move them into place before installing.
RUN \
    apk add --no-cache --virtual .splix-build \
        build-base \
        cups-dev \
        curl \
        ca-certificates \
        xz && \
    mkdir -p /tmp/jbigkit /tmp/splix && \
    curl --fail --location --retry 3 \
      -o /tmp/jbigkit.tar.gz \
      https://deb.debian.org/debian/pool/main/j/jbigkit/jbigkit_2.1.orig.tar.gz && \
    echo "de7106b6bfaf495d6865c7dd7ac6ca1381bd12e0d81405ea81e7f2167263d932  /tmp/jbigkit.tar.gz" | sha256sum -c - && \
    curl --fail --location --retry 3 \
      -o /tmp/splix.tar.xz \
      https://github.com/OpenPrinting/splix/releases/download/2.0.1/splix-2.0.1.tar.xz && \
    echo "533946d57897bf62a2cf8f74e488258e11fa0c55028fad43ada24c5686f38a06  /tmp/splix.tar.xz" | sha256sum -c - && \
    tar -xzf /tmp/jbigkit.tar.gz -C /tmp/jbigkit --strip-components=1 && \
    tar -xJf /tmp/splix.tar.xz -C /tmp/splix --strip-components=1 && \
    make -C /tmp/jbigkit/libjbig libjbig85.a CFLAGS="-O2 -fPIC" && \
    cd /tmp/splix && \
    mv ml2160*.ppd ml2165*.ppd ml3310*.ppd ppd/ && \
    CPATH=/tmp/jbigkit/libjbig LIBRARY_PATH=/tmp/jbigkit/libjbig \
      make DISABLE_JBIG=0 && \
    make install DISABLE_JBIG=0 \
      CUPSFILTER=/usr/lib/cups/filter \
      CUPSPPD=/usr/share/cups/model && \
    test -x /usr/lib/cups/filter/rastertoqpdl && \
    test -f /usr/share/cups/model/samsung/ml2240.ppd && \
    test -f /usr/share/cups/model/samsung/clp300.ppd && \
    cd / && \
    rm -rf /tmp/jbigkit /tmp/splix /tmp/jbigkit.tar.gz /tmp/splix.tar.xz && \
    apk del .splix-build

# foo2zjs: open-source drivers for ZjStream, XQX, QPDL, LAVAFLOW, HIPERC,
# SLX, HBPL and DDST host-based printers — HP LaserJet 1018/1020/1022,
# P1005-P1008, P1102, P1505, P2014/P2035, M1005/M1120/M1132/M1212, Color
# LaserJet 1600/2600n/CP1215/CP1025, Konica Minolta magicolor, Oki C3xx/C5xx,
# Samsung CLP/CLX color, Xerox Phaser 3010/6110/6115/6121, Dell 1355,
# Ricoh SP 112 and more (98 PPDs). Built from Debian's pinned 20200505 source.
# The PPDs drive foomatic-rip (from cups-filters) with the foo2*-wrapper
# scripts, which render through Ghostscript.
RUN \
    apk add --no-cache --virtual .foo2zjs-build \
        build-base \
        curl \
        ca-certificates \
        xz && \
    mkdir -p /tmp/foo2zjs && \
    curl --fail --location --retry 3 \
      -o /tmp/foo2zjs.tar.xz \
      https://deb.debian.org/debian/pool/main/f/foo2zjs/foo2zjs_20200505dfsg0.orig.tar.xz && \
    echo "87465e6a31f8501b90508d50e97901d2cce8b5b3de5418cc156e66d5c500421b  /tmp/foo2zjs.tar.xz" | sha256sum -c - && \
    tar -xJf /tmp/foo2zjs.tar.xz -C /tmp/foo2zjs --strip-components=1 && \
    cd /tmp/foo2zjs && \
    make foo2zjs foo2hp foo2xqx foo2lava foo2qpdl foo2oak foo2slx \
        foo2hiperc foo2hbpl2 foo2ddst \
        foo2zjs-wrapper foo2oak-wrapper foo2hp2600-wrapper foo2xqx-wrapper \
        foo2lava-wrapper foo2qpdl-wrapper foo2slx-wrapper foo2hiperc-wrapper \
        foo2hbpl2-wrapper foo2ddst-wrapper foo2zjs-pstops && \
    install -m 755 \
        foo2zjs foo2hp foo2xqx foo2lava foo2qpdl foo2oak foo2slx \
        foo2hiperc foo2hbpl2 foo2ddst \
        foo2zjs-wrapper foo2oak-wrapper foo2hp2600-wrapper foo2xqx-wrapper \
        foo2lava-wrapper foo2qpdl-wrapper foo2slx-wrapper foo2hiperc-wrapper \
        foo2hbpl2-wrapper foo2ddst-wrapper foo2zjs-pstops \
        /usr/bin/ && \
    install -d /usr/share/foo2zjs /usr/share/cups/model/foo2zjs && \
    install -m 644 gamma.ps gamma-lookup.ps /usr/share/foo2zjs/ && \
    for ppd in PPD/*.ppd; do \
        ./modify-ppd < "$ppd" | gzip -9 > "/usr/share/cups/model/foo2zjs/${ppd#PPD/}.gz"; \
    done && \
    test -s /usr/share/cups/model/foo2zjs/HP-LaserJet_1020.ppd.gz && \
    cd / && \
    rm -rf /tmp/foo2zjs /tmp/foo2zjs.tar.xz && \
    apk del .foo2zjs-build

# Copy data
COPY rootfs /

# Expose CUPS web interface port
EXPOSE 631

HEALTHCHECK \
    CMD lpstat -r || exit 1
