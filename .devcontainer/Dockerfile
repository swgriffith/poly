# Using the azure CLI container so we dont have to install that to start
FROM ghcr.io/cse-labs/k3d

RUN wget -q https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb
RUN dpkg -i packages-microsoft-prod.deb

RUN apt update
RUN apt install -y azure-functions-core-tools-4
