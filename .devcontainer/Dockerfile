FROM mcr.microsoft.com/vscode/devcontainers/base:focal

ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    TZ=Europe/Brussels \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US

COPY deploy/*.sh /tmp/deploy/

RUN apt-get update \
   && /bin/bash /tmp/deploy/git.sh \
   && /bin/bash /tmp/deploy/jekyll.sh