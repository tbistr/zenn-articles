FROM mcr.microsoft.com/vscode/devcontainers/base:debian-11

# install fish terminal
RUN apt-get update && apt-get install -y \
    fish \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y \
    nodejs npm && \
    npm install -g n && \
    n stable && \
    apt-get purge -y nodejs npm && \
    rm -rf /var/lib/apt/lists/*

RUN npm install -g \
    textlint \
    textlint-rule-preset-ja-spacing \
    textlint-rule-preset-ja-technical-writing \
    textlint-rule-spellcheck-tech-word

RUN npm install -g zenn-cli
