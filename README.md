# DigiCert® Software Trust Container Sign Action

Sign container images using DigiCert® Software Trust with CoSign via GitHub Actions. 

## Features

The DigiCert® Software Manager Container Sign Action offers the following features:

- Pulling the DigiCert® container signer tool
- Extracting the PKCS#11 module path and key URIs automatically
- Efficient client certificate handling with Docker volumes
- Automatic registry authentication for private registries
- Signing container images using CoSign with recursive signing (optional)
- Multiple signature verification methods (optional)
- Software Trust connectivity healthchecks 
- Comprehensive error handling and logging

## Prerequisites

Before you begin, ensure the following:

- Docker is installed and available in the runner environment.
- The runner can pull Docker images from your container registry.
- The `digicert-container-signer` Docker image contains the required `COSIGN_PKCS11_MODULE_PATH` environment variable.
- The DigiCert® client certificate is base64 encoded.

Additionally: 
- The `COSIGN_PKCS11_MODULE_PATH` is automatically extracted from the `digicertinc/digicert-container-signer:latest` Docker image, tagged as `digicert-container-signer`.
- The client certificate (`SM_CLIENT_CERT_FILE_B64`) is created once in a Docker volume for reuse across all signing operations. 
- You can sign single-architecture and multi-architecture container images.

### Security considerations

- Store sensitive information like keypair aliases as GitHub secrets
- Restrict access to your container registry
- Rotate signing keys regularly
- Use only the official DigiCert® container signer tool

## Workflow

Review the following high-level workflow for this action: 

1. Pulls the DigiCert® container signer tool.
2. Detects the PKCS#11 module path automatically.
3. Creates the client certificate in a Docker volume.
4. Sets up a reusable Docker command with environment variables.
5. Verifies versions of CoSign and SMCTL.
6. Checks connectivity to Software Trust.
7. Extracts the key URI for the specified keypair alias.
8. Logs in to the registry, when credentials are provided.
9. Signs the container image, optionally using recursive signing.
10. Verifies the signature, if verification is enabled.

## Configuration and parameters

### Inputs

| Input | Description | Required? | Default state |
|-------|-------------|----------|---------|
| `input` | Container image to sign | Yes | - |
| `keypair-alias` | Keypair alias to use for signing | Yes | - |
| `verify` | To verify container image signature | No | `false` |
| `registry-url` | Container registry URL | Yes | - |
| `recursive` | To sign multi-architecture images recursively | No | `false` |


### Environment variables

This action automatically extracts the required `COSIGN_PKCS11_MODULE_PATH` from the DigiCert® container signer tool. 

#### Required environment variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SM_API_KEY` | Software Trust API key | Yes |
| `SM_HOST` | Software Trust host | Yes |
| `SM_CLIENT_CERT_PASSWORD` | Software Trust client certificate password | Yes |
| `SM_CLIENT_CERT_FILE_B64` | Software Trust client certificate (base64 encoded) | Yes |

#### Registry authentication variables (optional)

| Variable | Description | Required? |
|----------|-------------|-----------|
| `REGISTRY_USERNAME` | Container registry username for authentication | No. <br>Only required for private registries that need authentication. |
| `REGISTRY_PASSWORD` | Container registry password for authentication | No. <br>Only required for private registries that need authentication. |


### Secrets

| Secret | Description | Required? |
|--------|-------------|-----------|
| `DIGICERT_KEYPAIR_ALIAS` | Software Trust keypair alias | Yes, if stored as a secret. |
| `SM_API_KEY` | Software Trust API Key | Yes |
| `SM_CLIENT_CERT_PASSWORD` | Software Trust client certificate password | Yes |
| `SM_CLIENT_CERT_FILE_B64` | Software Trust client certificate (base64 encoded) | Yes |
| `REGISTRY_USERNAME` | Container registry username | No. <br> Only required for private registries. |
| `REGISTRY_PASSWORD` | Container registry password | No. <br> Only required for private registries. |
| `VERBOSE` | Enable verbose output for cosign commands | No. <br> Only required to run command in verbose mode. |

## Usage examples

### Example of basic image signing

```yaml
name: Sign Container Image
on:
  push:
    branches: [ main ]

jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign Container Image
        uses: digicert/digicert-stm-container-sign-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-signing-key'
          registry-url: 'myregistry.io'
        env:
          SM_API_KEY: ${{ secrets.SM_API_KEY }}
          SM_HOST: ${{ vars.SM_HOST }}
          SM_CLIENT_CERT_PASSWORD: ${{ secrets.SM_CLIENT_CERT_PASSWORD }}
          SM_CLIENT_CERT_FILE_B64: ${{ secrets.SM_CLIENT_CERT_FILE_B64 }}
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

### Example of multi-architecture image signing

```yaml
name: Sign Multi-Architecture Container Image
on:
  push:
    branches: [ main ]

jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign Multi-Architecture Container Image
        uses: digicert/digicert-stm-container-sign-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-signing-key'
          registry-url: 'myregistry.io'
          recursive: 'true'  # Enable recursive signing for multi-arch images
          verbose: 'true' # Enable verbose mode for cosign command
        env:
          SM_API_KEY: ${{ secrets.SM_API_KEY }}
          SM_HOST: ${{ vars.SM_HOST }}
          SM_CLIENT_CERT_PASSWORD: ${{ secrets.SM_CLIENT_CERT_PASSWORD }}
          SM_CLIENT_CERT_FILE_B64: ${{ secrets.SM_CLIENT_CERT_FILE_B64 }}
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

### Example of image signing with verification

```yaml
name: Sign and Verify Container Image
on:
  push:
    branches: [ main ]

jobs:
  sign-and-verify:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign and Verify Container Image
        uses: digicert/digicert-stm-container-sign-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-signing-key'
          verify: 'true'
          registry-url: 'myregistry.io'
        env:
          SM_API_KEY: ${{ secrets.SM_API_KEY }}
          SM_HOST: ${{ vars.SM_HOST }}
          SM_CLIENT_CERT_PASSWORD: ${{ secrets.SM_CLIENT_CERT_PASSWORD }}
          SM_CLIENT_CERT_FILE_B64: ${{ secrets.SM_CLIENT_CERT_FILE_B64 }}
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

### Example of a complete workflow 

```yaml
name: Build, Sign, and Deploy Container

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: Sign container image
        uses: digicert/digicert-stm-container-sign-action@v1
        with:
          input: ${{ steps.meta.outputs.tags }}
          keypair-alias: ${{ secrets.DIGICERT_KEYPAIR_ALIAS }}
          registry-url: ${{ env.REGISTRY }}
          verify: 'true'
          recursive: 'true'  # Enable for multi-architecture images
          verbose: 'true' # Enable verbose mode for cosign command
        env:
          SM_API_KEY: ${{ secrets.SM_API_KEY }}
          SM_HOST: ${{ vars.SM_HOST }}
          SM_CLIENT_CERT_PASSWORD: ${{ secrets.SM_CLIENT_CERT_PASSWORD }}
          SM_CLIENT_CERT_FILE_B64: ${{ secrets.SM_CLIENT_CERT_FILE_B64 }}
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

## Troubleshooting

### Error handling for common issues

This action handles the following common issues:

- Missing DigiCert® container signer tool
- Missing or invalid `COSIGN_PKCS11_MODULE_PATH` 
- Missing or invalid `SM_CLIENT_CERT_FILE_B64` 
- Certificate conversion failures
- Missing key URI for the provided alias 
- Signing or verification failures 

### Self-troubleshooting tips for common issues

1. **Key URI not found**: Ensure the `keypair-alias` matches exactly the alias in your Software Trust.
2. **Docker image pull failure**: Ensure the runner has access to pull the `digicertinc/digicert-container-signer:latest` image, tagged as `digicert-container-signer`.
3. **Missing PKCS#11 module path**: Verify the container image contains the `COSIGN_PKCS11_MODULE_PATH` environment variable.
4. **Certificate creation failure**: Ensure `SM_CLIENT_CERT_FILE_B64` contains a valid, base64-encoded .p12 certificate without extra whitespace.
5. **Permission denied**: Ensure the runner has permission to pull and sign images.
6. **Registry authentication failure**: For private registries, ensure `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` are set correctly.
7. **Unauthorized signing error**: Ensure the registry URL matches the image registry. (This action automatically logs the registry if credentials are provided.)
8. **Multi-architecture signing issues**: Use `recursive: 'true'` when signing multi-architecture images.
9. **Healthcheck failures**: Verify your network connectivity to Software Trust. (This action will continue despite healthcheck failures.)
10. **Cosign command error**: set true to `verbose` in the input to run cosign command in verbose mode.

### Enable debug mode

To enable debug logging, add the following to your workflow:

```yaml
env:
  ACTIONS_STEP_DEBUG: true
```

## Contributions

To submit contributions, create a pull request.

## License information

This project is licensed under the MIT License. To learn more, see the [LICENSE file](LICENSE).

## Support

For support questions, open an issue in this repository.

## Feedback and issues
[Contact DigiCert](https://www.digicert.com/contact-us)


## Learn more
To learn more about centralizing and automating your code signing workflows with Software Trust Manager, reach out to [Sales/Enquiry](mailto:sales@digicert.com) or visit: https://www.digicert.com/software-trust-manager.
