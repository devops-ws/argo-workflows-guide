FROM ghcr.io/linuxsuren/hd:v0.0.70

RUN hd i kubernetes-sigs/kubectl
RUN hd get https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml

CMD ["kubectl", "apply", "-f", "install.yaml"]
