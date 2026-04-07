# Supply Chain Lab

## Intro

This lab will show you how to:

1. Build and publish a container image with attestation
2. Inspect attestation locally
3. Extract SBOM from manifest

## Prerequisites

It that you are already familiar with GitHub Actions (GHA).

If you need to brush up on GHA knowledge, try:

- [Introduction to GitHub Actions](https://learn.microsoft.com/en-us/training/modules/introduction-to-github-actions/)
- [Leverage GitHub Actions to publish to GitHub Packages](https://learn.microsoft.com/en-us/training/modules/github-actions-packages/)

For Docker, take a look at [Docker - Get Started](https://docs.docker.com/get-started/)

You also need to have [Docker
Desktop](https://www.docker.com/products/docker-desktop/) installed.

## Build and publish a container image with attestation

### Workflow

Create a new Actions workflow with the following:

```yaml
name: "Build, Attest and Push container image"

# This workflow is triggered manually
on: workflow_dispatch

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write
  artifact-metadata: write

env:
  CONTAINER_IMAGE_NAME: ${{ github.repository }}

jobs:
  container-image:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Log in to the Github Package Container registry
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata
        id: docker-meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ env.CONTAINER_IMAGE_NAME }}
          tags: type=sha,format=long

      - name: Build and Push container images
        uses: docker/build-push-action@v7
        id: build-and-push
        with:
          platforms: linux/amd64,linux/arm64
          sbom: true
          provenance: mode=max
          push: true
          tags: ${{ steps.docker-meta.outputs.tags }}
          labels: ${{ steps.docker-meta.outputs.labels }}

      - name: Generate container image attestation
        uses: actions/attest@v4
        with:
          subject-name: ghcr.io/${{ env.CONTAINER_IMAGE_NAME }}
          subject-digest: ${{ steps.build-and-push.outputs.digest }}
```

### Understand the workflow

Get an overview of what each of the actions does.

- <https://github.com/docker/setup-buildx-action>
- <https://github.com/docker/login-action>
- <https://github.com/docker/metadata-action>
- <https://github.com/docker/build-push-action>
- <https://github.com/actions/attest>

You can find additional information about Docker GitHub Actions
[here](https://docs.docker.com/build/ci/github-actions/).

### Run the workflow

Now that you understand the workflow, let's run it.
Go to the "Actions" tab and trigger the workflow.
Wait for the workflow to complete.

You can inspect attestation metadata directly from GitHub.

![GH Actions attestation](./docs/gh-attestation.png)

## Inspect attestation locally

### Find image URL

Go to your repository page.
Click on the package shown on the right.

![GH repository packages](./docs/gh-packages.png)

Alternatively, you can go to your GitHub page and click on the Packages tab
(`https://github.com/<user>?tab=packages`).

You can run the command shown on your computer to pull the image.
It should show `docker pull <image-url>`, where the image URL is structured like
`ghcr.io/<user>/<repo>:sha-<hash>`.

## Inspect and verify attestation

To inspect provenance attestation, run:

```sh
docker buildx imagetools inspect <image-url> --format "{{ json (index .Provenance \"linux/amd64\").SLSA }}"
```

You can replace `linux/amd64` for `linux/arm64` to extract for a different platform.

*Note: without multi-platform build it would be `--format "{{ json .Provenance.SLSA }}"`*

To inspect SBOM attestation, run:

```sh
docker buildx imagetools inspect <image-url> --format "{{ json (index .SBOM \"linux/amd64\").SPDX }}"
```

Extract the SBOM:

```sh
docker scout sbom --format spdx <image-url>
```

You can verify the attestation with [GH CLI](https://cli.github.com/).

```sh
gh attestation verify --owner <user> oci://ghcr.io/<user>/<repo>@sha256:<digest-hash>
```

## Scan image

You can scan the image for vulnerabilities with [Trivy](https://trivy.dev/).
I prefer to run Trivy through Docker, given the [recent
breach](https://www.bleepingcomputer.com/news/security/trivy-vulnerability-scanner-breach-pushed-infostealer-via-github-actions/).

```sh
docker run dhi.io/trivy:0-debian13-dev image <image-url>
```

### Scan for secrets

Trivy can also detect secrets in images.
To try it, you first need to add something that looks like a secret.
Open `Dockerfile` and add:

```
ENV GITHUB_TOKEN=ghp_12DAQ1B6mS2BnXW2xAGjzzzzzzzzzzzzzzzz
```

Push a commit.
Re-run the workflow and try to scan the new image.

**Important:** There is no guarantee that Trivy will find secrets, since it
just looks for known patterns.

### Scan for vulnerabilities

To test vulnerability scanning, you can try to add a vulnerable package to `.csproj` file.
Try to add:

```xml
    <PackageReference Include="Magick.NET-Q16-AnyCPU" Version="14.0.0" />
```

Push a commit.
Re-run the workflow and try to scan the new image.

## Dependency-Track

[Dependency-Track](https://dependencytrack.org/) is a OWASP project that can
analyze SBOMs across all of your projects, and alert you when vulnerabilities
have been discovered.

Let's try it out!

Run the commands in [Quickstart (Docker
Compose)](https://docs.dependencytrack.org/getting-started/deploy-docker/#quickstart-docker-compose)
guide.

On first login, use:

```
Username: admin
Password: admin
```

Create a new project.

![Create project dialog](./docs/dt-create-project.png)

You can upload a SBOM manually from the Components tab of the project.

![Upload SBOM](./docs/dt-upload-sbom.png)

> [!TIP]
> If you get validation errors when uploading an SBOM, try to disable
> validation under Administration -> BOM Formats -> BOM Validation.

Dependency-Track unfortunately only support CycloneDX format, not the SPDX in
which the SBOM attestation was created.

You can extract the SPDX from image manifest, then use [CycloneDX CLI](https://github.com/CycloneDX/cyclonedx-cli) to convert it.

```sh
docker buildx imagetools inspect <image-url> --format "{{ json (index .SBOM \"linux/amd64\").SPDX }}" > sbom.spdx.json
docker run -it --volume .:/root cyclonedx/cyclonedx-cli convert --input-file /root/sbom.spdx.json --output-file /root/sbom.cyclonedx.json
```

1. Upload the SBOM to Dependency-Track.
2. [Create a GitHub classic personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic)
3. Go to Administration -> Vulnerability Sources -> GitHub Advisories, add your PAT.
4. Go to "Audit vulnerabilities" tab and click "Reanalyse".

Dependency-Track has schedulers for various tasks.
Some take 24 hours to trigger.
You can change the schedulers by going to Administration -> Task scheduler.

Restarting the `dependency-track-apiserver-1` container also seems to trigger the scheduled tasks.

> [!IMPORTANT]
> In real world, one would upload SBOM for new releases from CI/CD.
> There is a both a [REST API](https://docs.dependencytrack.org/usage/cicd/)
> and [GitHub
> Action](https://github.com/marketplace/actions/upload-bom-to-dependency-track)
> for uploading SBOMs.

## GitHub Security and quality

GitHub has some security tools built-in.
You can find them in "Security and quality" tab on your repository page.

Read the [Quickstart for securing your
repository](https://docs.github.com/en/code-security/getting-started/quickstart-for-securing-your-repository).
