FROM ubuntu:latest

RUN apt-get update && apt-get -y install build-essential socat pcscd pcsc-tools \
    opensc openssh-server openssh-client software-properties-common

RUN apt-add-repository ppa:yubico/stable

RUN apt-get update && apt-get -y install yubikey-manager

COPY configure_hsm_with_ssh_agent /usr/local/bin/configure_hsm_with_ssh_agent

RUN chmod +x /usr/local/bin/configure_hsm_with_ssh_agent



