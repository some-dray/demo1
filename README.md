# Chainguard Demo

This demo showcases the benefits of using Chainguard images over traditional, general-purpose container images. We will compare a standard `python:latest` image from Docker Hub with a Chainguard Python image.

## Prerequisites

Before you begin, ensure you have the following tools installed:

*   [Docker](https://docs.docker.com/get-docker/)
*   [grype](https://github.com/anchore/grype)
*   [cosign](https://docs.sigstore.dev/cosign/installation/)

## Vulnerability Scanning Comparison

Let's start by scanning the `python:latest` image for vulnerabilities using `grype`.

### Docker Hub `python:latest`

```bash
grype python:latest
```

The scan of `python:latest` reveals a large number of vulnerabilities:

```
NAME                          INSTALLED                     FIXED-IN            TYPE    VULNERABILITY     SEVERITY   
apt                           2.6.1                                             deb     CVE-2011-3374     Negligible  
binutils                      2.40-2                                            deb     CVE-2017-13716    Negligible  
binutils                      2.40-2                                            deb     CVE-2018-20673    Negligible  
binutils                      2.40-2                                            deb     CVE-2018-20712    Negligible  
binutils                      2.40-2                                            deb     CVE-2018-9996     Negligible  
binutils                      2.40-2                                            deb     CVE-2021-32256    Negligible  
binutils                      2.40-2                                            deb     CVE-2023-1972     Negligible  
... (many more vulnerabilities)
```

### Chainguard `cgr.dev/chainguard/python:latest`

Now, let's scan the Chainguard Python image.

```bash
grype cgr.dev/chainguard/python:latest
```

The scan of the Chainguard image shows a significant reduction in vulnerabilities:

```
No vulnerabilities found
```

This stark contrast highlights the security benefits of using Chainguard images, which are built from the ground up with a minimal set of components to reduce the attack surface.


## Verifying Image Signatures with `cosign`

Chainguard images are signed with `cosign`, allowing you to verify their authenticity. To verify the signature of the Chainguard Python image, you need to provide the correct certificate identity and OIDC issuer.

```bash
cosign verify \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity=https://github.com/chainguard-images/images/.github/workflows/release.yaml@refs/heads/main \
  cgr.dev/chainguard/python | jq
```

Successful verification will produce output similar to the following:

```
Verification for cgr.dev/chainguard/python:latest --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The code-signing certificate was verified using trusted certificate authority certificates
```

This confirms that the image was signed by Chainguard's official build process and has not been tampered with.

## Software Bill of Materials (SBOM)

Chainguard images include a Software Bill of Materials (SBOM), which provides a complete inventory of the software components in the image. You can download the SBOM using `cosign`.

```bash
cosign verify-attestation \
  --type https://spdx.dev/Document \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity=https://github.com/chainguard-images/images/.github/workflows/release.yaml@refs/heads/main \
  cgr.dev/chainguard/python
```

This command will download a JSON file containing the SBOM. You can then inspect this file to see all the packages and their versions included in the image. This is invaluable for security audits and compliance checks.


## Application Demo

Here is a simple Flask application that we will containerize.

### `app/app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, from a container!'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```

### Dockerfiles

Here are the two Dockerfiles we will use to containerize the application.

#### `Dockerfile.python`

```dockerfile
FROM python:latest

WORKDIR /app

COPY app/ /app

RUN pip install Flask

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0"]
```

#### `Dockerfile.chainguard`

```dockerfile
# Build stage
FROM cgr.dev/chainguard/python:latest-dev AS builder

WORKDIR /app

COPY app/ /app

RUN python -m venv venv
ENV PATH="/app/venv/bin":$PATH

RUN pip install Flask

# Final stage
FROM cgr.dev/chainguard/python:latest

WORKDIR /app

COPY --from=builder /app/venv /app/venv
ENV PATH="/app/venv/bin:$PATH"

COPY app/ /app

EXPOSE 5000

ENTRYPOINT ["python", "app.py"]
```

## Building and Running the Containers

### Standard Python Image

```bash
docker build -t python-app -f Dockerfile.python .
docker run -d -p 5000:5000 python-app
docker stop python-app
docker run -it -p 5000:5000 python-app bash
```

### Chainguard Image

```bash
docker build -t chainguard-app -f Dockerfile.chainguard .
docker run -d -p 5002:5000 chainguard-app
docker stop chainguard-app
```
