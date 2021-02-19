# Original: https://github.com/docker-library/ruby/blob/1d3efccb4b5fc99cb7ae15ca9ba3ad7b56b7709b/2.7/buster/slim/Dockerfile
#
# Copyright (C) 2014 Docker, Inc.
# (C) 2021 Masataka SUZUKI.
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
# 1. Redistributions of source code must retain the above copyright
# notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
# notice, this list of conditions and the following disclaimer in the
# documentation and/or other materials provided with the distribution.
#
# THIS SOFTWARE IS PROVIDED BY THE AUTHOR AND CONTRIBUTORS ``AS IS'' AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
# OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
# HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
# OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
# SUCH DAMAGE.

################################################################
# Build Ruby
################################################################
FROM amazonlinux:2 as builder

ENV RUBY_MAJOR=2.7 \
    RUBY_VERSION=2.7.2 \
    RUBY_DOWNLOAD_SHA256=1b95ab193cc8f5b5e59d2686cb3d5dcf1ddf2a86cb6950e0b4bdaae5040ec0d6 \
    RUBY_PREFIX=/opt/ruby

RUN set -eux; \
        yum remove -y glibc-langpack-en; \
        yum update -y; \
        yum install -y \
# runtime dependencies
            bzip2 \
            ca-certificates \
            libffi-devel \
            gmp-devel \
            openssl-devel \
            libyaml-devel \
            procps \
            zlib-devel \
# build tools
            autoconf \
            bison \
            gcc \
            bzip2-devel \
            gdbm-devel \
            glib2-devel \
            ncurses-devel \
            readline-devel \
            libxml2-devel \
            libxslt-devel \
            make \
            ruby \
            wget \
            tar \
            xz \
        ; \
        \
# skip installing gem documentation
        mkdir -p /usr/local/etc; \
        { \
            echo 'install: --no-document'; \
            echo 'update: --no-document'; \
        } >> /usr/local/etc/gemrc; \
        \
# fetch Ruby source code
        wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz"; \
        echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum --check --strict; \
        \
        mkdir -p /usr/src/ruby; \
        tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1; \
        rm ruby.tar.xz; \
        \
        cd /usr/src/ruby; \
        \
# hack in "ENABLE_PATH_CHECK" disabling to suppress:
#   warning: Insecure world writable dir
        { \
            echo '#define ENABLE_PATH_CHECK 0'; \
            echo; \
            cat file.c; \
        } > file.c.new; \
        mv file.c.new file.c; \
        \
# compile and install Ruby
        autoconf; \
        ./configure \
            --build=x86_64-linux-gnu \
            --disable-install-doc \
            --enable-shared \
            --prefix=$RUBY_PREFIX \
        ; \
        make -j "$(nproc)"; \
        make install; \
        \
# collect dependencies
        find $RUBY_PREFIX -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
             | awk '/=>/ { print $(NF-1) }' \
             | sort -u \
             | xargs -r rpm -qf \
             | grep -v '^file ' \
             | sort -u \
             > $RUBY_PREFIX/dependencies.txt \
        ; \
# rough smoke test
        $RUBY_PREFIX/bin/ruby --version; \
        $RUBY_PREFIX/bin/gem --version; \
        $RUBY_PREFIX/bin/bundle --version

################################################################
# Build container image
################################################################
FROM amazonlinux:2

ENV RUBY_PREFIX=/opt/ruby \
    LANG=C.UTF-8

# don't create ".bundle" in all our apps
ENV GEM_HOME=/usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
    BUNDLE_APP_CONFIG=$GEM_HOME \
    PATH=$GEM_HOME/bin:$RUBY_PREFIX/bin:$PATH

# copy ruby from builder image
COPY --from=builder $RUBY_PREFIX $RUBY_PREFIX

RUN set -eux; \
# install Ruby dependencies
        yum install -y $(cat $RUBY_PREFIX/dependencies.txt); \
        yum update -y; \
        \
# skip installing gem documentation
        mkdir -p /usr/local/etc; \
        { \
            echo 'install: --no-document'; \
            echo 'update: --no-document'; \
        } >> /usr/local/etc/gemrc; \
        \
# TODO: to reduce image size
        yum remove -y glibc-langpack-en; \
        find /usr/share/locale -maxdepth 1 -mindepth 1 -type d -not -name default -exec rm -rf {} \; ; \
        rm -rf /usr/share/doc/* ; \
        rm -rf /usr/share/man/* ; \
        \
        rm -rf /var/cache/yum/*; \
        yum autoremove -y; \
        yum clean all; \
        \
# adjust permissions of a few directories for running "gem install" as an arbitrary user
        install -d -m 0777 "$GEM_HOME"; \
        \
# rough smoke test
        ruby --version; \
        gem --version; \
        bundle --version

CMD ["irb"]
