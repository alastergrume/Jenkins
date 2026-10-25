FROM jenkins/ssh-agent:jdk21

USER root

# Install docker-cli and docker-compose plugin
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl gnupg git openssh-client ansible sshpass \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && chmod a+r /etc/apt/keyrings/docker.asc \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
    > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends docker-ce-cli docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*

# Copy custom certificates
COPY custom-certs/ /tmp/custom-certs/
RUN for cert in /tmp/custom-certs/*.crt /tmp/custom-certs/*.pem; do \
      [ -f "$cert" ] || continue; \
      cp "$cert" "/usr/local/share/ca-certificates/custom-$(basename "${cert%.*}").crt"; \
    done \
 && update-ca-certificates \
 && rm -rf /tmp/custom-certs

# Acces to docker-engine via socket on host machine
ARG DOCKER_GID=999
RUN group="$(getent group "${DOCKER_GID}" | cut -d: -f1)" \
 && if [ -z "$group" ]; then group=docker-host; groupadd -g "${DOCKER_GID}" "$group"; fi \
 && usermod -aG "$group" jenkins

# Add ssh key for the remote agent node in main jenkins
COPY jenkins-agent/jenkins_agent_key.pub /home/jenkins/.ssh/authorized_keys
RUN chown -R jenkins:jenkins /home/jenkins/.ssh \
 && chmod 700 /home/jenkins/.ssh \
 && chmod 600 /home/jenkins/.ssh/authorized_keys
