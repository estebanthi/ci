# docker-build-push

This GitHub Actions workflow builds and pushes Docker images to a container registry.
It serves as a base workflow and is usable this way, but it may be customized depending on the exact use case.

## Use cases

### Build and push Docker images for CI/CD

This workflow can be used in CI/CD pipelines to automate the process of building and pushing Docker images whenever code is pushed to the repository or a pull request is created.

I use it with [watchtower](https://github.com/containrrr/watchtower) to automatically update running containers with the latest images.

### Build an upstream

You may want to build an upstream image from another repository and push it to your own container registry.
You can do this this by modifying the checkout step to pull from the external repository and pass the correct build context to the Docker build step.

```yaml
      - name: Checkout external repository to ./external-src
        uses: actions/checkout@v5
        with:
          repository: owner/repo-name
          ref: main
          server-url: ${{ github.server_url }}
          path: external-src
          fetch-depth: 0  # Fetch all history for all branches and tags

      # ...
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./external-src
      
      # ...

```

### When SSH access is needed during build

If your Docker build process requires SSH access (for example, to clone private repositories), you can enable SSH agent, and configure the Docker build step to use it.
You will also need to change the Dockerfile to use the SSH mount.

```yaml
      - name: Start ssh-agent
        uses: https://github.com/webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.CI_SSH_PRIVATE_KEY }}

      # ...
      
      - name: Build & push
        uses: docker/build-push-action@v5
        with:
          ssh: default
          build-args: |
            GITEA_HOSTKEY=${{ secrets.SSH_GITEA_HOSTKEY }}  # Pass host key as build-arg
```

And modify your Dockerfile like this:

```Dockerfile
# Install dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        openssh-client \
        ca-certificates \
        libnss3 \
        nss-plugin-pem \
        libbrotli1 && \
    rm -rf /var/lib/apt/lists/*

# Add Gitea host key to known_hosts
ARG GITEA_HOSTKEY
RUN set -eux; \
    mkdir -p /etc/ssh; \
    printf '%s\n' "$GITEA_HOSTKEY" > /etc/ssh/ssh_known_hosts; \
    chmod 644 /etc/ssh/ssh_known_hosts; \
    ssh-keygen -l -E sha256 -f /etc/ssh/ssh_known_hosts

# Clone private repository using SSH during build
RUN --mount=type=ssh git clone git@your-gitea-server:your-repo.git /path/to/destination

# You can do whatever you need with SSH by using the --mount=type=ssh flag
# RUN --mount=type=ssh \
#     GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=yes -o UserKnownHostsFile=/etc/ssh/ssh_known_hosts' \
#     pip install --no-cache-dir -r requirements.txt
```