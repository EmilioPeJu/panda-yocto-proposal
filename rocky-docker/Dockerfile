FROM docker.io/rockylinux:9
RUN yum -y upgrade && yum -y install \
    automake \
    bzip2 \
    chrpath \
    cpio \
    cpp \
    diffstat \
    diffutils \
    dnf \
    epel-release \
    file \
    findutils \
    gawk \
    gcc \
    gcc-c++ \
    git \
    glibc-devel \
    glibc-langpack-en \
    gzip \
    hostname \
    iproute \
    libacl \
    libtool \
    lz4 \
    make \
    man \
    ncurses \
    patch \
    perl \
    perl-bignum \
    perl-Data-Dumper \
    python3-pexpect \
    perl-Text-ParseWords \
    perl-Thread-Queue \
    procps-ng \
    python3 \
    python3-jinja2 \
    python3-pip \
    socat \
    sudo \
    tar \
    tmux \
    unzip \
    wget \
    which \
    xalan-j2 \
    xmlto \
    xz \
    zstd

RUN yum -y install ncurses-compat-libs

# Install rpcgen library
RUN curl -SL https://dl.rockylinux.org/pub/rocky/8/Devel/x86_64/os/Packages/r/rpcgen-1.3.1-4.el8.x86_64.rpm -o /tmp/rpcgen-1.rpm && ls /tmp && \
    yum -y install /tmp/rpcgen-1.rpm && \
    rm /tmp/rpcgen-1.rpm

RUN curl -SL https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/p/perl-File-Copy-2.34-473.module+el8.10.0+1616+0d20cc68.noarch.rpm -o /tmp/perl-FileCopy-2.rpm && \
    yum -y install --skip-broken /tmp/perl-FileCopy-2.rpm && \
    rm /tmp/perl-FileCopy-2.rpm

RUN curl -SL https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/p/perl-File-Compare-1.100.600-473.module+el8.10.0+1616+0d20cc68.noarch.rpm -o /tmp/perl-File-Compare-1.rpm && \
    yum -y install --skip-broken /tmp/perl-File-Compare-1.rpm && \
    rm /tmp/perl-File-Compare-1.rpm

RUN curl -SL https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/p/perl-locale-1.09-473.module+el8.10.0+1616+0d20cc68.noarch.rpm -o /tmp/perl-locale-1.rpm && \
    yum -y install --skip-broken /tmp/perl-locale-1.rpm && \
    rm /tmp/perl-locale-1.rpm

RUN yum -y group install "Development Tools"

RUN ln -s /usr/bin/python3 /usr/bin/python

ENV LANG en_US.utf8

# Make sure git doesn't fail when used to obtain a tag name
RUN git config --global --add safe.directory '*'

RUN echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
CMD ["/bin/bash"]
RUN useradd user -u 1001
