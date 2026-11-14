FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    vlc-bin \
    vlc-plugin-base \
    vlc-plugin-access-extra \
    ffmpeg \
    tshark \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m vlc
USER vlc
WORKDIR /home/vlc

CMD ["sleep", "infinity"]
