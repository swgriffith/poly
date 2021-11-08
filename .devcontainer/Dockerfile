# Using the azure CLI container so we dont have to install that to start
FROM mcr.microsoft.com/azure-cli

RUN apk update

# Install tools
RUN apk add nano vim tcpdump tmux

RUN apk add zsh \
&& sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
&& install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl \
&& rm kubectl

RUN curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
&& chmod 700 get_helm.sh \
&& ./get_helm.sh \
&& rm get_helm.sh

RUN curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg \
&& mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg \
&& sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-$(lsb_release -cs)-prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list' \
&& apt-get update \
&& apt-get install azure-functions-core-tools-4