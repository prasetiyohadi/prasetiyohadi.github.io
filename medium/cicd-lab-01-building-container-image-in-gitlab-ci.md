# CI/CD Lab: Building Container Image in Gitlab CI

## CI/CD Lab Introduction

I asked LLM to present me the best practice for CI/CD pipeline and it give me this diagram.

![CI/CD best practice according to LLM](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*BsgnPQRpyCJ5u8WaJeLPlw.png)

Based on this diagram, I queried the LLM to give me the implementation for this CI/CD pipeline for Gitlab CI. Why Gitlab CI? Gitlab CI is the main feature of Gitlab platform and it is the main workflow of anyone using Gitlab to store their code, and I have most experience working with Gitlab CI in my entire career. The LLM gives me some configuration files and `.gitlab-ci.yml`, but it does not actually work out of the box. Before fixing the pipeline, let’s talk with initial work that I’ve been done in this CI/CD lab before my discussion with the LLM.

## Building Container Image in Gitlab CI Pipeline

The diagram shows that the first phase of the pipeline is a set of preflight tests. But, before I started this lab, I actually already playing with Gitlab CI, especially building container image using Gitlab CI pipelines. Based on Gitlab Runners that available to use we can use several methods to build container image. In this demo, I only explore building container image using **Docker-in-Docker** and **rootless mode using BuildKit**. The following table shows comparison between Docker-in-Docker and rootless mode using BuildKit.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*NZYhtJ5gn_A-7ZIYRLglBA.png)

I created a simple Go app and created the following `Dockerfile` to define the container image.

```dockerfile
# --- Stage 1: Build ---
FROM golang:1.25-alpine AS builder
# Set working directory
WORKDIR /app
# Install build dependencies and security patches
RUN apk add --no-cache git ca-certificates && update-ca-certificates
# Leverage Docker cache for dependencies
# Rule: cache dependencies to speed up builds and reduce network calls, improving efficiency and reliability.
COPY go.mod ./
# COPY go.sum ./  # Uncomment if you have dependencies
RUN go mod download
# Copy source code
COPY . .
# Build the binary
# -ldflags="-s -w": Strips debug info to reduce binary size
# CGO_ENABLED=0: Static binary for maximum portability
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .
# --- Stage 2: Final Runtime ---
# Using "static" distroless image: no shell, no package manager, highly secure.
# Rule: a minimal base image with only the necessary runtime dependencies to run the application, reducing attack surface and improving security.
FROM gcr.io/distroless/static-debian12:latest-amd64
WORKDIR /
# Copy only the compiled binary from the builder
COPY --from=builder /app/server /server
# Use a non-root user (Standard in Distroless)
# Rule: least privileged user with UID 65532 (nobody) to run the application
USER 65532:65532
# Expose the application port
EXPOSE 8080
# Run the binary
ENTRYPOINT ["/server"]
```

The `Dockerfile` uses several best practices to make sure the container image is small and secure.

- Multi-stage build pattern
- Language-specific compilation flag to ensure the application binary is small and maximize portability
- Use minimal and secure base image with only necessary runtime dependencies
- Use a non-root user in the base image

We can further improve the source code by adding a `.dockerignore` file to make sure only the necessary source code is copied to the builder image.

You can check the code for this article in this repository [https://gitlab.com/pras-labs/ci-demo/-/tree/build-ci?ref_type=heads](https://gitlab.com/pras-labs/ci-demo/-/tree/build-ci?ref_type=heads).

### Building image with Docker-in-Docker

Using Docker-in-Docker is the easiest way to build container image in Gitlab CI pipeline. It requires **Docker executor** or **Kubernetes executor** runner with **privileged mode** which the hosted Gitlab Runners usually provide. Just provide the `Dockerfile` and create the Gitlab CI job in the `.gitlab-ci.yml` file.

```yaml
build:
  image: docker:29.2.1-cli
  stage: build
  services:
    - docker:29.2.1-dind
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
```

The Gitlab CI has [predefined variables](https://docs.gitlab.com/ci/variables/predefined_variables/)

- `CI_REGISTRY` : the container registry address `registry.gitlab.com`
- `CI_REGISTRY_USER` : the registry user used by Gitlab CI pipeline
- `CI_REGISTRY_PASSWORD` : the registry password used by Gitlab CI pipeline
- `CI_REGISTRY_IMAGE` : equal to `$CI_REGISTRY/<project namespace>/<project name>` that you can use directly in the Gitlab CI pipeline to define address for your container image
- `CI_COMMIT_REF_SLUG` : the branch or tag name for which project is built by the pipeline normalized for usage in URL

When the pipeline running in `main` branch, it will build and publish image with address _registry.gitlab.com/<project namespace>/<project name>:main_

![Gitlab CI job for Docker-in-Docker method](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HAxlnu5VXi1KhU6FBz6-Wg.png)

### Building image in rootless mode with BuildKit

If you need increased security for **Gitlab Runner**, for example not allowing the Gitlab Runner to run in **privileged mode**, then Docker-in-Docker method maybe not available to build your container image. [Gitlab CI documentation](https://docs.gitlab.com/ci/docker/using_docker_build/#docker-alternatives) provides two alternatives to build container image in non-privileged mode with BuildKit and Buildah. I choose the BuildKit method since I already use `docker buildx` to build container image in my local environment and want to have consistent workflow. Use the following Gitlab CI job in the `.gitlab-ci.yml` file to create build job using rootless BuildKit.

```yaml
build:
  image:
    name: moby/buildkit:rootless
    entrypoint: [""]
  stage: build
  variables:
    # Using flag for rootless, secure builds
    BUILDKITD_FLAGS: --oci-worker-no-process-sandbox
    # Using cache
    CACHE_IMAGE: $CI_REGISTRY_IMAGE:cache
    # $IMAGE_TAG is a custom variable that combines the registry address with $CI_COMMIT_REF_SLUG, the image tag.
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  before_script:
    - mkdir -p ~/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > ~/.docker/config.json
  script:
    # This script uses cache and able to build multi-platform images: linux/amd64; linux/arm64, using --opt platform= flag
    - |
      buildctl-daemonless.sh build \
        --frontend dockerfile.v0 \
        --local context=. \
        --local dockerfile=. \
        --export-cache type=registry,ref=$CACHE_IMAGE \
        --import-cache type=registry,ref=$CACHE_IMAGE \
        --opt platform=linux/amd64 \
        --output type=image,name=$IMAGE_TAG,push=true
```

When using the rootless BuildKit method, we need to specify `BUILDKITD_FLAGS` to configure permission, create `~/.docker/config.json` to authenticate to the Gitlab Container Registry, and use provided script `buildctl-daemonless.sh` to build the container image. The script has complete configuration flags to configure caching, multi-platform support, and output type of the container image.

![Gitlab CI job for rootless BuildKit method](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*lFCQKdg0GBV8wD57Mj3R7Q.png)

Compared to the build job using Docker-in-Docker method in previous section, we can see that the build time is faster when using the rootless BuildKit method (52 seconds) compared to the Docker-in-Docker method (1 minute 15 seconds).

## Gitlab Container Registry

Gitlab provides each repository or “project” its own container registry to store container image built using Gitlab CI. The container registry is available at this address `https://gitlab.com/<project namespace>/<project name>/container_registry`.

![Gitlab Container Registry](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*sIkEo6DMr8h7gR90laSsfg.png)

The Container Registry also has several production ready features that should be enable to prevent artifacts related vulnerabilities. Such as **container repository path protection**, **container image tags protection**, and **automatic clean up** to manage the storage usage of the stored container images.

![Gitlab Container Registry Settings](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*W_BUZKi3myzvVlXoTaOVAA.png)

The Gitlab CI pipelines automatically has access to create and publish image to the Container Registry. We should use the container repository path protection feature to limit actor that can publish image so only user with specific role such as Maintainer can trigger Gitlab CI pipeline to create and publish container image.

If you want to access container image stored in Gitlab Container Registry, you need to create personal access token or group token (not available in free account!) with fine-grained resource permission to read (or delete) the container image.

![Creating Fine-Grained Token](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*oMCLci6vbXIGhTyUPUQmhw.png)

After generating the access token, you can authenticate to the registry and pulling the image using the docker command.

```bash
echo $GITLAB_TOKEN | docker login -u $GITLAB_USER --password-stdin registry.gitlab.com
docker pull registry.gitlab.com/<project namespace>/<project name>:<image tag>
```

## Conclusion

In this story, I explored two methods of building container image using Gitlab CI pipeline. Building a good container image start with a good `Dockerfile` that follow the best practices to create a small and secure container image. Then, we configure the build job in the CI/CD pipeline to follow the best practice according to our environment requirements. The general rule is to always make sure to configure build cache to speed up the build process. Docker-in-Docker is the simplest method to build container image in Gitlab CI pipeline, but it requires privileged runner which could be non-starter for security-conscious organization. The better alternative is to use rootless mode with BuildKit for building container image. It runs in non-privileged mode and supports caching and multi-platform out-of-the-box.

## What’s Next

In the next post, we will start exploring the best practice CI/CD pipeline created by the LLM and its implementation in Gitlab CI. We could also deep dive to the building container images in Gitlab CI using self-hosted runner since there is quota limit when using hosted Gitlab Runner for free user. Stay tuned for next stories with Pras’ Labs: The Hands-On Engineer! 👋🏼
