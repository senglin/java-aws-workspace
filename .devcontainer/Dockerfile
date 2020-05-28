#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

FROM golang:1 AS fargatedist

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Configure apt, install packages and tools
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \
    #
    # Verify git, process tools, lsb-release (common in install instructions for CLIs) installed
    && apt-get -y install git openssh-client zip vim less iproute2 procps lsb-release upx file make \
    #
    # SSL dependencies
    && apt-get -y install apt-transport-https ca-certificates gnupg2 \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/gotools

WORKDIR /fargate

RUN git clone -b combined-fixes https://github.com/senglin/fargatecli.git .
RUN go mod download
RUN GOOS=linux GOARCH=amd64 go build -o fargate main.go

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog

#-------------------------------------------------------------------------------------------------------------

FROM openjdk:11-jdk

# This Dockerfile adds a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG NONROOT_USER=$USERNAME

# Options for common package install script
ARG INSTALL_ZSH="true"
ARG UPGRADE_PACKAGES="true"
ARG COMMON_SCRIPT_SOURCE="https://raw.githubusercontent.com/microsoft/vscode-dev-containers/master/script-library/common-debian.sh"
ARG COMMON_SCRIPT_SHA="dev-mode"

# Configure apt
RUN apt-get update \
    && export DEBIAN_FRONTEND=noninteractive \
    #
    # Verify git, common tools / libs installed, add/modify non-root user, optionally install zsh
    && apt-get -y install --no-install-recommends curl vim ca-certificates 2>&1 \
    && curl -sSL  ${COMMON_SCRIPT_SOURCE} -o /tmp/common-setup.sh \
    && ([ "${COMMON_SCRIPT_SHA}" = "dev-mode" ] || (echo "${COMMON_SCRIPT_SHA} */tmp/common-setup.sh" | sha256sum -c -)) \
    && /bin/bash /tmp/common-setup.sh "${INSTALL_ZSH}" "${USERNAME}" "${USER_UID}" "${USER_GID}" "${UPGRADE_PACKAGES}" \
    && rm /tmp/common-setup.sh
ENV VISUAL=vim
ENV EDITOR=vim

#-------------------Uncomment the following steps to install Maven CLI Tools----------------------------------
ARG MAVEN_VERSION=3.6.3
ARG MAVEN_SHA=c35a1803a6e70a126e80b2b3ae33eed961f83ed74d18fcd16909b2d44d7dada3203f1ffe726c17ef8dcca2dcaa9fca676987befeadc9b9f759967a8cb77181c0
RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
    && export DEBIAN_FRONTEND=noninteractive \
    && curl -fsSL -o /tmp/apache-maven.tar.gz https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
    && echo "${MAVEN_SHA} /tmp/apache-maven.tar.gz" | sha512sum -c - \
    && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
    && rm -f /tmp/apache-maven.tar.gz \
    && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn
COPY maven-settings.xml /usr/share/maven/ref/
ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG /root/.m2
#-------------------------------------------------------------------------------------------------------------

#-------------------Uncomment the following steps to install Gradle CLI Tools---------------------------------
# ENV GRADLE_HOME /opt/gradle
# ENV GRADLE_VERSION 5.4.1
# ARG GRADLE_DOWNLOAD_SHA256=7bdbad1e4f54f13c8a78abc00c26d44dd8709d4aedb704d913fb1bb78ac025dc
# RUN curl -sSL --output gradle.zip "https://services.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip" \
#     && export DEBIAN_FRONTEND=noninteractive \
#     && echo "${GRADLE_DOWNLOAD_SHA256} *gradle.zip" | sha256sum --check - \
#     && unzip gradle.zip \
#     && rm gradle.zip \
#     && mv "gradle-${GRADLE_VERSION}" "${GRADLE_HOME}/" \
#     && ln -s "${GRADLE_HOME}/bin/gradle" /usr/bin/gradle
#-------------------------------------------------------------------------------------------------------------

# AWS Tools
RUN \
    # Install AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && sudo ./aws/install \
    && rm -rf aws awscliv2.zip \
    #
    # Install AWS SAM CLI
    # Install Homebrew before installing aws-sam-cli
    && apt-get -y install ruby build-essential groff \
    && echo | sudo -u ${USERNAME} sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)" \
    && echo 'eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv)' >> /home/${USERNAME}/.bashrc \
    && eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv) \
    && brew tap aws/tap \
    && brew install aws-sam-cli \
    #
    # Install AWS ECS CLI
    && sudo curl -o /usr/local/bin/ecs-cli https://s3.cn-north-1.amazonaws.com.cn/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && sudo chmod +x /usr/local/bin/ecs-cli
    #
    # Install FARGATE CLI
COPY --from=fargatedist /fargate/fargate /usr/local/bin/
    # && curl -L -o fargate-0.3.2-linux-amd64.zip https://github.com/awslabs/fargatecli/releases/download/0.3.2/fargate-0.3.2-linux-amd64.zip \
    # && sudo unzip fargate-0.3.2-linux-amd64.zip -d /usr/local/bin/ \
    # && rm -rf fargate-0.3.2-linux-amd64.zip

# Docker installation
RUN \
    # Install Docker CE CLI (and engine)
    apt-get install -y apt-transport-https ca-certificates curl gnupg2 lsb-release \
    && curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | apt-key add - 2>/dev/null \
    && echo "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y docker-ce-cli containerd.io docker-ce socat \
    #
    # Install Docker Compose
    && LATEST_COMPOSE_VERSION=$(curl -sSL "https://api.github.com/repos/docker/compose/releases/latest" | grep -o -P '(?<="tag_name": ").+(?=")') \
    && curl -sSL "https://github.com/docker/compose/releases/download/${LATEST_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose


# 3-WAYS TO ENABLE DOCKER IN THE CONTAINER
# https://github.com/Microsoft/vscode-dev-containers/tree/master/containers/docker-in-docker
RUN echo '#!/bin/sh\n\
    if [ "$VSC_DIND" = "root" ] \n\
    then \n\
        # Create docker-root-entrypoint.sh to link host docker socket to container docker socket. This is only for root access.
        # In order to use this script, please ensure that remoteUser is set to root (or removed) in devcontainer.json file.
        sudo ln -s /var/run/docker-host.sock /var/run/docker.sock \n\
        echo Symbolic created at /var/run/docker.sock \n\
    elif [ "$VSC_DIND" = "nonroot" ] \n\
    then \n\
        # Create docker-nonroot-entrypoint.sh to spin up socat, to enable non-root access
        sudo rm -rf /var/run/docker.sock\n\
        ((sudo socat UNIX-LISTEN:/var/run/docker.sock,fork,mode=660,user='${NONROOT_USER}' UNIX-CONNECT:/var/run/docker-host.sock) 2>&1 >> /tmp/vscr-dind-socat.log) & > /dev/null\n\
        echo socat: UNIX-LISTEN:/var/run/docker.sock UNIX-CONNECT:/var/run/docker-host.sock \n\
    else \n\
        # Create dockerd-entrypoint.sh to start dockerd in the container, please make sure that non-root users are added to docker group
        # This requires --privileged to be set on devcontainer.json
        ((sudo dockerd --host="unix:///var/run/docker.sock") 2>&1 >> /tmp/vscr-dockerd.log) & >/dev/null\n\
        sudo usermod -aG docker '${NONROOT_USER}' \n\
        echo Dockerd started \n\
    fi \n\
    "$@"' >> /usr/local/share/entrypoint.sh \
    && chmod +x /usr/local/share/entrypoint.sh


# Clean up
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Allow for a consistant java home location for settings - image is changing over time
RUN if [ ! -d "/docker-java-home" ]; then ln -s "${JAVA_HOME}" /docker-java-home; fi


# Setting the ENTRYPOINT to [docker-root-entrypoint.sh | docker-nonroot-entrypoint.sh | dockerd-entrypoint.sh]
# will configure docker installation type in container
# when "overrideCommand": false is set in devcontainer.json.
# The script will also execute CMD if you need to alter startup behaviors.
ENTRYPOINT [ "/usr/local/share/entrypoint.sh" ]
CMD [ "sleep", "infinity" ]