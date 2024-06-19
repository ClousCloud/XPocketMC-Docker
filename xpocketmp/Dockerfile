# Build stage for compilation. Build tools like g++ will not be copied into the final stage to reduce image size.
FROM ubuntu:18.04 AS builder
ARG PMMP_TAG
ARG COMPILE_SH_ARGS="-f -g"
ARG PMMP_REPO=xpocketmc/XPocketMP

RUN test ! -z ${PMMP_TAG} || (echo "Missing build-arg PMMP_TAG" && false)

RUN apt-get update && apt-get install --no-install-recommends -y \
    curl \
    ca-certificates \
    build-essential \
    m4 \
    gzip \
    bzip2 \
    bison \
    git \
    cmake \
    autoconf \
    automake \
    pkg-config \
    libtool \
    libtool-bin \
    re2c

WORKDIR /build
RUN git clone https://github.com/${PMMP_REPO}.git -b ${PMMP_TAG} --recursive . || \
    (echo "Branch ${PMMP_TAG} not found. Cloning default branch instead." && git clone https://github.com/${PMMP_REPO}.git --recursive .)
WORKDIR /build/build/php
RUN ./compile.sh -t linux64 -j ${THREADS:-$(nproc)} ${COMPILE_SH_ARGS} -P $(echo ${PMMP_TAG} | cut -d "." -f1)
RUN ln -s /build/build/php/bin/php7/bin/php /usr/bin/php

WORKDIR /build
RUN curl -L https://getcomposer.org/installer | php
RUN mv composer.phar /usr/bin/composer

RUN composer install --classmap-authoritative --no-dev --prefer-source
RUN if test -d build/preprocessor; then php build/preprocessor/PreProcessor.php --path=src --multisize; else echo "Skipping preprocessor step (preprocessor not found)"; fi
RUN php $(test -d vendor/pocketmine/bedrock-data && echo -n vendor/pocketmine/bedrock-data || echo -n src/pocketmine/resources/vanilla)/.minify_json.php

RUN php -dphar.readonly=0 build/server-phar.php --git $(git rev-parse HEAD)
# Just to make sure DevTools didn't false-positive-exit
RUN test -f /build/XPocketMP.phar

# Final stage
FROM ubuntu:18.04
LABEL maintainer="PMMP Team <team@pmmp.io>"

RUN apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates \
    wget

RUN groupadd -g 1000 xpocketmp
RUN useradd -r -d /xpocketmp -p "" -u 1000 -m -g xpocketmp xpocketmp

WORKDIR /xpocketmp
COPY --from=builder /build/build/php/bin/php7 /usr/php
RUN grep -q '^extension_dir' /usr/php/bin/php.ini && \
    sed -i.bak "s{^extension_dir=.*{extension_dir=\"$(find /usr/php -name '*debug-zts*')\"{" /usr/php/bin/php.ini || echo "extension_dir=\"$(find /usr/php -name '*debug-zts*')\"" >> /usr/php/bin/php.ini
RUN ln -s /usr/php/bin/php /usr/bin/php
COPY --from=builder /build/XPocketMP.phar XPocketMP.phar
ADD start.sh /usr/bin/start-xpocketmp

RUN mkdir /plugins /data
RUN chown 1000:1000 /plugins /data . -R
RUN chmod o+x /usr/bin/php

USER xpocketmp

ENV TERM=xterm

EXPOSE 19132/tcp
EXPOSE 19132/udp

VOLUME ["/data", "/plugins"]

CMD ["start-xpocketmp"]
