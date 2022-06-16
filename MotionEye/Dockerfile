FROM nvidia/cuda:11.7.0-base-ubuntu22.04
LABEL maintainer="Micah Focht <micah@focht.xyz>"

ENV MOTION_VERSION=4.4.0
ENV MOTIONEYE_VERSION=0.42.1
ENV FFMPEG_VERSION=5.0.1
ARG BUILD_DATE
ARG VCS_REF


# By default, run as root.
ARG RUN_UID=0
ARG RUN_GID=0

COPY . /tmp/motioneye

# Compile and install fresh ffmpeg from sources:
# See: https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu
RUN apt-get update -qq && apt-get -y install \
      autoconf \
      automake \
      build-essential \
      cmake \
      git-core \
      libass-dev \
      libfreetype6-dev \
      libsdl2-dev \
      libtool \
      libva-dev \
      libvdpau-dev \
      libvorbis-dev \
      libxcb1-dev \
      libxcb-shm0-dev \
      libxcb-xfixes0-dev \
      pkg-config \
      texinfo \
      wget \
      zlib1g-dev \
      nasm \
      yasm \
      libx265-dev \
      libnuma-dev \
      libvpx-dev \
      libmp3lame-dev \
      libopus-dev \
      libx264-dev \
      libfdk-aac-dev
      
RUN mkdir -p ~/ffmpeg_sources ~/bin && cd ~/ffmpeg_sources && \
    wget -O ffmpeg-${FFMPEG_VERSION}.tar.bz2 https://ffmpeg.org/releases/ffmpeg-${FFMPEG_VERSION}.tar.bz2 && \
    tar xjvf ffmpeg-${FFMPEG_VERSION}.tar.bz2 && \
    cd ffmpeg-${FFMPEG_VERSION} && \
    PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
      --prefix="$HOME/ffmpeg_build" \
      --pkg-config-flags="--static" \
      --extra-cflags="-I$HOME/ffmpeg_build/include" \
      --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
      --extra-libs="-lpthread -lm" \
      --bindir="$HOME/bin" \
      --enable-libfdk-aac \
      --enable-gpl \
      --enable-libass \
      --enable-libfreetype \
      --enable-libmp3lame \
      --enable-libopus \
      --enable-libvorbis \
      --enable-libvpx \
      --enable-libx264 \
      --enable-libx265 \
      --enable-cuda-nvcc \
      --enable-libnpp \
      --extra-cflags=-I/usr/local/cuda/include \
      --extra-ldflags=-L/usr/local/cuda/lib64 \
      --disable-static \
      --enable-shared \
      --enable-nonfree && \
    PATH="$HOME/bin:$PATH" make -j8 && \
    make install -j8 && \
    hash -r
RUN mv ~/bin/ffmpeg /usr/local/bin && mv ~/bin/ffprobe /usr/local/bin && mv ~/bin/ffplay /usr/local/bin

RUN apt-get update && \
    DEBIAN_FRONTEND="noninteractive" apt-get -t jammy --yes --option Dpkg::Options::="--force-confnew" --no-install-recommends install \
      build-essential \
      curl \
      libmicrohttpd12 \
      libpq5 \
      lsb-release \
      mosquitto-clients \
      python2 \
      python2.7-dev \
      python-pip \
      python2-pip-whl \
      v4l-utils \
      libmariadb3 \
      libssl-dev\
      libcurl4-gnutls-dev \
      librtmp-dev \
      fdisk \ 
      default-libmysqlclient-dev && \
    curl -Lo /motion.deb https://github.com/Motion-Project/motion/releases/download/release-${MOTION_VERSION}/jammy_motion_${MOTION_VERSION}-1_amd64.deb \
    && dpkg -i /motion.deb \
    && rm -f /motion.deb  
    
RUN python2 -m pip install wheel pycurl

RUN python2 -m pip install jinja2 pillow tornado setuptools pytz\
    # Change uid/gid of user/group motion to match our desired IDs.  This will
    # make it easier to use execute motion as our desired user later.
    && sed -i -e "s/^\(motion:[^:]*\):[0-9]*:[0-9]*:\(.*\)/\1:${RUN_UID}:${RUN_GID}:\2/" /etc/passwd && \
    sed -i -e "s/^\(motion:[^:]*\):[0-9]*:\(.*\)/\1:${RUN_GID}:\2/" /etc/group && \
    python2 -m pip install motioneye==${MOTIONEYE_VERSION} && \
    # Cleanup
    rm -rf /tmp/motioneye && \
    apt-get purge --yes python-setuptools && \
    apt-get autoremove --yes && \
    apt-get --yes clean && rm -rf /var/lib/apt/lists/* && rm -f /var/cache/apt/*.bin

ADD extra/motioneye.conf.sample /usr/share/motioneye/extra/

# R/W needed for motioneye to update configurations
VOLUME /etc/motioneye

# Video & images
VOLUME /var/lib/motioneye

CMD test -e /etc/motioneye/motioneye.conf || \
    cp /usr/share/motioneye/extra/motioneye.conf.sample /etc/motioneye/motioneye.conf ; \
    # We need to chown at startup time since volumes are mounted as root. This is fugly.
    chown motion:motion /var/run /var/log /etc/motioneye /var/lib/motioneye /usr/share/motioneye/extra ; \
    su -g motion motion -s /bin/bash -c "/usr/local/bin/meyectl startserver -c /etc/motioneye/motioneye.conf"

EXPOSE 8765