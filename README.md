# Container signing with DigiCert® Software Trust Manager

Sign container images using DigiCert® Software Trust with CoSign or Notation and GitHub Actions. 

This GitHub Actions enables secure, automated container signing within your CI/CD workflows using Software Trust. It supports two signature formats:

- **CoSign (default):** Uses PKCS#11 to reference your private key through the smpkcs11 library. Produces Sigstore-compatible signatures.
- **Notation (COSE/JWS):** Uses the DigiCert STM Notation plugin to sign OCI artifacts with CNCF Notary Project signatures. Supports both JWS and COSE envelope formats.

## Features

Review the following key features of this action: 

### CoSign mode (default)
- Pulls the DigiCert® container signer tool
- Extracts the PKCS#11 module path and key URIs automatically
- Signs container images using CoSign with recursive signing (optional)
- Multiple signature verification methods (optional)

### Notation mode
- Pulls the DigiCert® container signer tool with pre-installed STM plugin
- Signs using the Notation Plugin Framework with `notation-digicert-stm`
- Supports JWS and COSE envelope formats
- Optional RFC 3161 timestamping via `--timestamp-url`
- Multi-arch image signing (iterates child manifests when `recursive: true`)
- Trust policy-based verification with automatic CA certificate extraction

### Common
- Efficient client certificate handling with Docker volumes
- Automatic registry authentication for private registries
- Software Trust connectivity healthchecks 
- Comprehensive error handling and logging

## Before you begin

Before you begin, review the following prerequisites: 

- Docker is installed and available in the runner environment
- The runner can pull Docker images from your container registry
- The `digicert-container-signer` Docker image contains the required `COSIGN_PKCS11_MODULE_PATH` environment variable
- The DigiCert® client certificate is base64 encoded

Additionally, consider the following statements: 
- The `COSIGN_PKCS11_MODULE_PATH` is automatically extracted from the `digicertinc/digicert-container-signer:latest` Docker image, tagged as `digicert-container-signer`.
- The client certificate (`SM_CLIENT_CERT_FILE_B64`) is created once in a Docker volume for reuse across all signing operations. 
- You can sign single-architecture and multi-architecture container images.

### Security best practices

Review the following security-related best practices:

- Store sensitive information like keypair aliases as GitHub secrets.
- Restrict access to your container registry.
- Rotate signing keys regularly.
- Use only the official DigiCert® container signer tool.

## Workflow

Review the following high-level workflow where this action: 

1. Pulls the DigiCert® container signer tool.
2. Detects the PKCS#11 module path automatically.
3. Creates the client certificate in a Docker volume.
4. Sets up a reusable Docker command with environment variables.
5. Verifies CoSign and SMCTL versions.
6. Checks connectivity to Software Trust.
7. Extracts the key URI for the specified keypair alias.
8. Logs in to the registry, when credentials are provided.
9. Signs the container image, optionally using recursive signing.
10. Verifies the signature, if verification is enabled.


## Configuration and parameters

### Inputs

| Input | Description | Required? | Default state |
|-------|-------------|----------|---------|
| `input` | The container image to sign | Yes | - |
| `keypair-alias` | The keypair alias to use for signing | Yes | - |
| `signature-format` | Signature format: `cosign` or `notation` | No | `cosign` |
| `envelope-type` | Notation envelope type: `jws` or `cose` (notation only) | No | `jws` |
| `timestamp-url` | RFC 3161 TSA URL for timestamping (notation only) | No | - |
| `timestamp-root-cert` | Path to TSA root certificate PEM (notation only) | No | - |
| `verify` | To verify the container image signature | No | `false` |
| `registry-url` | The container registry URL | Yes | - |
| `recursive` | To sign multi-architecture images recursively | No | `false` |
| `verbose` | To enable verbose output for signing commands | No | `false` |


### Environment variables

This action automatically extracts the required `COSIGN_PKCS11_MODULE_PATH` from the DigiCert® container signer tool. 

#### Required environment variables

| Variable | Description | Required? |
|----------|-------------|----------|
| `SM_API_KEY` | The Software Trust API key | Yes |
| `SM_HOST` | The Software Trust host. <br><br>To view acceptable values, review **SM_Host options** below.  | Yes |
| `SM_CLIENT_CERT_PASSWORD` | The Software Trust client certificate password | Yes |
| `SM_CLIENT_CERT_FILE_B64` | The Software Trust client certificate (base64 encoded) | Yes |

##### SM_Host options
Review the following values that you can use in the `SM_HOST` variable:

| Country                          | Host type   | SM_HOST value                                         |
|----------------------------------|-------------|-------------------------------------------------------|
| United States of America (USA)   | Demo        | https://clientauth.demo.one.digicert.com              |
| United States of America (USA)   | Production  | https://clientauth.one.digicert.com                   |
| Switzerland (CH)                 | Demo        | https://clientauth.demo.one.ch.digicert.com           |
| Switzerland (CH)                 | Production  | https://clientauth.one.ch.digicert.com                |
| Japan (JP)                       | Demo        | https://clientauth.demo.one.digicert.co.jp            |
| Japan (JP)                       | Production  | https://clientauth.one.digicert.co.jp                 |
| Netherlands (NL)                 | Demo        | https://clientauth.demo.one.nl.digicert.com           |
| Netherlands (NL)                 | Production  | https://clientauth.one.nl.digicert.com                |


To learn more about host environments, review the [Host environment](https://docs.digicert.com/en/software-trust-manager/get-started/requirements.html#host-environment-367442) section in the DigiCert documentation site.

#### Registry authentication variables (optional)

| Variable | Description | Required? |
|----------|-------------|-----------|
| `REGISTRY_USERNAME` | The container registry username for authentication | No. <br>Optional if a `docker login` step was already performed in your workflow. |
| `REGISTRY_PASSWORD` | The container registry password for authentication | No. <br>Optional if a `docker login` step was already performed in your workflow. |


### Secrets

| Secret | Description | Required? |
|--------|-------------|-----------|
| `DIGICERT_KEYPAIR_ALIAS` | The Software Trust keypair alias | Yes, if stored as a secret. |
| `SM_API_KEY` | The Software Trust API key | Yes |
| `SM_CLIENT_CERT_PASSWORD` | The Software Trust client certificate password | Yes |
| `SM_CLIENT_CERT_FILE_B64` | The Software Trust client certificate (base64 encoded) | Yes |
| `REGISTRY_USERNAME` | The container registry username | No. <br> Optional if a `docker login` step was already performed in your workflow. |
| `REGISTRY_PASSWORD` | The container registry password | No. <br> Optional if a `docker login` step was already performed in your workflow. |
| `VERBOSE` | To enable verbose output for CoSign commands | No. <br> Only required to run command in verbose mode. |

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
        uses: digicert/digicert-container-signing-action@v1
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
        uses: digicert/digicert-container-signing-action@v1
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
        uses: digicert/digicert-container-signing-action@v1
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

### Example of notation signing with JWS envelope

```yaml
name: Sign Container Image with Notation (JWS)
on:
  push:
    branches: [ main ]

jobs:
  sign-notation:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign Container Image with Notation
        uses: digicert/digicert-container-signing-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-notation-key'
          signature-format: 'notation'
          envelope-type: 'jws'
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

### Example of notation signing with COSE envelope and timestamping

```yaml
name: Sign Container Image with Notation (COSE + TSA)
on:
  push:
    branches: [ main ]

jobs:
  sign-notation-cose:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign Container Image with Notation COSE
        uses: digicert/digicert-container-signing-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-rsa3072-key'
          signature-format: 'notation'
          envelope-type: 'cose'
          timestamp-url: 'http://timestamp.digicert.com'
          verify: 'true'
          registry-url: 'myregistry.io'
          verbose: 'true'
        env:
          SM_API_KEY: ${{ secrets.SM_API_KEY }}
          SM_HOST: ${{ vars.SM_HOST }}
          SM_CLIENT_CERT_PASSWORD: ${{ secrets.SM_CLIENT_CERT_PASSWORD }}
          SM_CLIENT_CERT_FILE_B64: ${{ secrets.SM_CLIENT_CERT_FILE_B64 }}
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

### Example of notation multi-arch signing

```yaml
name: Sign Multi-Arch Image with Notation
on:
  push:
    branches: [ main ]

jobs:
  sign-multiarch:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Sign Multi-Arch Image
        uses: digicert/digicert-container-signing-action@v1
        with:
          input: 'myregistry/myimage:latest'
          keypair-alias: 'my-notation-key'
          signature-format: 'notation'
          envelope-type: 'cose'
          recursive: 'true'
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
        uses: digicert/digicert-container-signing-action@v1
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

### Built-in troubleshooting

This action troubleshoots the following common issues:

- Missing DigiCert® container signer tool
- Missing or invalid `COSIGN_PKCS11_MODULE_PATH` 
- Missing or invalid `SM_CLIENT_CERT_FILE_B64` 
- Certificate conversion failures
- Missing key URI for alias 
- Signing or verification failures 

### Self-troubleshooting for common issues

| Issue | Troubleshooting step | 
|-------|-------------|
| Key URI not found | Ensure the `keypair-alias` matches the alias in Software Trust. | 
| Docker image pull failure | Ensure the runner has access to pull the `digicertinc/digicert-container-signer:latest` image, tagged as `digicert-container-signer`. | 
| Missing PKCS#11 module path | Verify the container image contains the `COSIGN_PKCS11_MODULE_PATH` environment variable. | 
| Certificate creation failure | Ensure `SM_CLIENT_CERT_FILE_B64` contains a valid, base64-encoded .p12 certificate without extra whitespace. | 
| Permission denied | Ensure the runner has permission to pull and sign images. | 
| Registry authentication failure | For private registries, ensure `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` are set correctly. | 
| Unauthorized signing error | Ensure the registry URL matches the image registry. (This action automatically logs the registry if credentials are provided.) | 
| Multi-architecture signing issue | Use `recursive: 'true'` when signing multi-architecture images. | 
| Healthcheck failures | Verify your network connectivity to Software Trust. (This action will continue despite healthcheck failures.) | 
| CoSign command error | Set `verbose` to `true` in the input to run CoSign command in verbose mode. | 
| Notation plugin not found | Ensure the `digicert-container-signer` image is accessible and contains the pre-installed plugin. Run `docker run --rm digicert-container-signer notation plugin list` to verify. |
| Notation trust policy rejection | Check that `registryScopes` matches the image registry. Run with `verbose: 'true'` to see the exact trust policy error. |
| Notation CA cert extraction fails | The keypair must have an associated certificate chain in STM. Verify with `smctl kp cert-chain <alias>`. |
| COSE/JWS mismatch | Ensure `envelope-type` matches what admission controllers expect. Ratify supports both, but the trust policy must match. |



### Enable debug mode

To enable debug logging, add the following to your workflow:

```yaml
env:
  ACTIONS_STEP_DEBUG: true
```

## Submit feedback and questions

* To submit contributions, create a pull request.
* To submit support questions, open an issue in this repository.
* To submit general feedback, [contact DigiCert](https://www.digicert.com/contact-us).


## Additional information

For more information about centralizing and automating your container signing workflows with Software Trust, contact [Sales/Enquiry](mailto:sales@digicert.com) or visit our [documentation site](https://docs.digicert.com/).

## License information

This project is licensed under the MIT License. To learn more, see the [LICENSE file](LICENSE).
