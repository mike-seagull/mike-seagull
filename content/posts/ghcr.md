---
categories:
    - tutorial
keywords:
    - github actions
    - github container registry
    - containers
    - ghcr
    - zsh
    - oh-my-zsh
    - gitpod
summary: Tutorial for build containers using Github Actions and deploying them GitHub Container Registry.
title: "Build and deploy containers to ghcr.io"
date: 2022-01-17T04:42:40Z
---
I've been using [Gitpod.io](https://gitpod.io) as my main coding IDE. The browser-based IDE has a [feature for using a custom docker image](https://www.gitpod.io/docs/config-docker) as a base for your development space.

I base my containers with zsh and oh-my-zsh preinstalled
How to:
1. Create a Dockerfile in a repo
```dockerfile
FROM gitpod/workspace-full:latest
RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v1.1.2/zsh-in-docker.sh)" -- \
    -t fishy \
    -p ubuntu \
    -p git \
    -p npm -p nvm \
    -p pyenv -p pip -p python \
    -p docker -p docker-compose \ 
    -p golang \
    -p ansible \
    -p terraform \
    -p https://github.com/zsh-users/zsh-autosuggestions \
    -p https://github.com/zsh-users/zsh-completions \
    -a 'export ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#757575"'
RUN echo "if [ -t 1 ]; then" >> ~/.bashrc
RUN echo "exec zsh" >> ~/.bashrc
RUN echo "fi" >> ~/.bashrc
```
There's a cool [project out there for installing zsh and oh-my-zsh in docker](https://github.com/deluan/zsh-in-docker). I added all of the oh-my-zsh plugins I might use with the `-p` option.

2. Create a GitHub Action workflow
```yaml
name: base docker image

on:
  push:
    branches: ["master"]

jobs:
  gitpod-base:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Log in to the Container registry
        uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
        with:
          images: ghcr.io/${{ github.repository_owner }}/gitpod-images

      - name: Build and push Docker image
        uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
        with:
          context: .
          file: base.Dockerfile
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/gitpod-images:base
          labels: ${{ steps.meta.outputs.labels }}
```
This workflow will build the container on a push of the main branch. The container will be stored as `ghcr.io/${GITHUB_USERNAME}/gitpod-images:base`.

3. Use the image in Gitpod
Using the image is pretty straight-forward. Gitpod allows you to create a [config file (.gitpod.yml)](https://www.gitpod.io/docs/config-gitpod-file) in your repo.
```yaml
github:
  prebuilds:
    master: true
    branches: true
image:
  file: ghcr.io/mike-seagull/gitpod-images:base
```

All of the code is available in [my Github repo](https://github.com/mike-seagull/mike-seagull.github.io).
