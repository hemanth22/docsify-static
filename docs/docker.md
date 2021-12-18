## create a dockerfile

1. Create a dockerfile in ./docs/Dockerfile folders

```
FROM alpine:latest

MAINTAINER Hemanth BITRA<hemanthbitra@live.com>
LABEL Quickly serve static files using Python http.server module.

WORKDIR /usr/app

ADD docs/. /usr/app

RUN echo "Hello World" && \
  apk add python3 && \
  rm -rf /tmp/* && \
  rm -rf /var/cache/* 
EXPOSE 8000
ENTRYPOINT ["python3", "-m", "http.server"]
```

2. commit above dockerfile in docs and add github-action to create and deploy the docker image.


__vi .github/workflows/docker-image.yml__
```yaml
name: Docker Image CI

on:
  push:
    branches: [ container ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2 # Checking out the repo
    - name: Build and Publish latest Docker image
      uses: VaultVulp/gp-docker-action@1.1.7
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }} # Provide GITHUB_TOKEN to login into the GitHub Packages
        image-name: image_name # Provide only Docker image name, tag will be automatically set to latest
        dockerfile: ./docs/Dockerfile
    
  snyk:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Build a Docker image
      run: docker build -t image_name:v1 -f ./docs/Dockerfile .
    - name: Build a Docker image
      run: docker ps -a; docker images
    - name: Run Snyk to check Docker image for vulnerabilities
      # Snyk can be used to break the build when it detects vulnerabilities.
      # In this case we want to upload the issues to GitHub Code Scanning
      continue-on-error: true
      uses: snyk/actions/docker@master
      env:
        # In order to use the Snyk Action you will need to have a Snyk API token.
        # More details in https://github.com/snyk/actions#getting-your-snyk-token
        # or you can signup for free at https://snyk.io/login
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        image: image_name:v1
        args: --file=./docs/Dockerfile
    - name: Upload result to GitHub Code Scanning
      uses: github/codeql-action/upload-sarif@v1
      with:
        sarif_file: snyk.sarif

  docker:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      name: Check out code

    - uses: mr-smithers-excellent/docker-build-push@v5
      name: Build & push Docker image
      with:
        image: docker_namespace/image_name
        tags: v1, latest
        registry: docker.io
        dockerfile: ./docs/Dockerfile
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
```
