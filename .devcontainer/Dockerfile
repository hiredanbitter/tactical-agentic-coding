FROM node:20

ARG TZ
ENV TZ="$TZ"

ARG CLAUDE_CODE_VERSION=latest
ARG JAVA_VERSION=21

# Install key management tools before adding signed repositories
RUN apt-get update && apt-get install -y --no-install-recommends gnupg2 dirmngr ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# Add Eclipse Temurin apt repository for latest open-license Java LTS
RUN curl -fsSL https://packages.adoptium.net/artifactory/api/gpg/key/public \
  | gpg --dearmor \
  | tee /usr/share/keyrings/adoptium.gpg > /dev/null \
  && echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb bookworm main" \
  | tee /etc/apt/sources.list.d/adoptium.list > /dev/null

# Add GitHub CLI apt repository
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
  && chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | tee /etc/apt/sources.list.d/github-cli.list > /dev/null

# Install basic development tools and iptables/ipset
RUN apt-get update && apt-get install -y --no-install-recommends \
  less \
  git \
  procps \
  sudo \
  fzf \
  zsh \
  man-db \
  unzip \
  gnupg2 \
  gh \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  jq \
  nano \
  vim \
  redis-server \
  postgresql \
  postgresql-contrib \
  maven \
  temurin-${JAVA_VERSION}-jdk \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Ensure default node user has access to /usr/local/share
RUN mkdir -p /usr/local/share/npm-global && \
  chown -R node:node /usr/local/share

ARG USERNAME=node

# Persist bash history.
RUN SNIPPET="export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  && mkdir /commandhistory \
  && touch /commandhistory/.bash_history \
  && chown -R $USERNAME /commandhistory

# Set `DEVCONTAINER` environment variable to help with orientation
ENV DEVCONTAINER=true

# Create workspace and config directories and set permissions
RUN mkdir -p /workspace /home/node/.claude && \
  chown -R node:node /workspace /home/node/.claude

WORKDIR /workspace

ARG GIT_DELTA_VERSION=0.18.2
RUN ARCH=$(dpkg --print-architecture) && \
  wget "https://github.com/dandavison/delta/releases/download/${GIT_DELTA_VERSION}/git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  sudo dpkg -i "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  rm "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb"

# Set up non-root user
USER node

# Install global packages
ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global
ENV PATH=$PATH:/usr/local/share/npm-global/bin

# Set the default shell to zsh rather than sh
ENV SHELL=/bin/zsh

# Set the default editor and visual
ENV EDITOR=nano
ENV VISUAL=nano

# Default powerline10k theme
ARG ZSH_IN_DOCKER_VERSION=1.2.0
RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v${ZSH_IN_DOCKER_VERSION}/zsh-in-docker.sh)" -- \
  -p git \
  -p fzf \
  -a "source /usr/share/doc/fzf/examples/key-bindings.zsh" \
  -a "source /usr/share/doc/fzf/examples/completion.zsh" \
  -a "export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  -x

# RUN echo 'alias c="claude config set apiKey $ANTHROPIC_API_KEY"' >> /home/node/.zshrc

# Install Claude
RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} pnpm@latest-10

# Set Java environment variables
ENV JAVA_HOME=/usr/lib/jvm/temurin-${JAVA_VERSION}-jdk-amd64
ENV PATH=$JAVA_HOME/bin:$PATH

# Copy and set up firewall and post-start scripts
COPY init-firewall.sh /usr/local/bin/
COPY post-start.sh /usr/local/bin/

# Set up Claude Code API key helper
RUN mkdir -p /home/node/.claude && \
  echo '{"apiKeyHelper": "/home/node/.claude/anthropic_key.sh"}' > /home/node/.claude/settings.json && \
  echo '#!/bin/sh\necho $ANTHROPIC_API_KEY' > /home/node/.claude/anthropic_key.sh && \
  chmod +x /home/node/.claude/anthropic_key.sh && \
  chown -R node:node /home/node/.claude

USER root
RUN chmod +x /usr/local/bin/init-firewall.sh /usr/local/bin/post-start.sh && \
  echo "node ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh, /usr/bin/apt-get, /usr/bin/systemctl, /usr/sbin/service, /usr/bin/psql, /usr/bin/createdb" > /etc/sudoers.d/node-sudoers && \
  chmod 0440 /etc/sudoers.d/node-sudoers
USER node
