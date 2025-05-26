FROM myoung34/github-runner:latest

LABEL org.opencontainers.image.source https://github.com/HepworthKubernetes/runners

# Pin version explicitly for reliability
ARG KUBECTL_VERSION=v1.29.3

RUN curl -sSL -o kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl
