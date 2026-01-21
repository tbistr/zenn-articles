FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:24-bullseye

# install fish terminal
RUN apt-get update && apt-get install -y \
    fish \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g \
    textlint \
    textlint-rule-preset-ja-spacing \
    textlint-rule-preset-ja-technical-writing \
    textlint-rule-spellcheck-tech-word

RUN npm install -g zenn-cli
