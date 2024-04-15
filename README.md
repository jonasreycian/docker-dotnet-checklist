# docker-dotnet-sample

A simple .NET web application example for [Docker's .NET Language Guide](https://docs.docker.com/language/dotnet/).


## Run tests when developing locally
The sample application already has an xUnit test inside the tests directory. When developing locally, you can use Compose to run your tests.

Run the following command in the docker-dotnet-sample directory to run the tests inside a container.
```
docker compose run --build --rm server dotnet test /source/tests
````

## Run tests when building
To run your tests when building, you need to update your Dockerfile. You can create a new test stage that runs the tests, or run the tests in the existing build stage. For this guide, update the Dockerfile to run the tests in the build stage.

The following is the updated Dockerfile.

```Dockerfile
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS build
ARG TARGETARCH
COPY . /source
WORKDIR /source/src
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet publish -a ${TARGETARCH/amd64/x64} --use-current-runtime --self-contained false -o /app
RUN dotnet test /source/tests # Add this
```

Run the following command to build an image using the build stage as the target and view the test results. Include --progress=plain to view the build output, --no-cache to ensure the tests always run, and --target build to target the build stage.

```bash
docker build -t dotnet-docker-image-test --progress=plain --no-cache --target build .
```
You should see output containing the following.

```
11 [build 5/5] RUN dotnet test /source/tests
11 1.564   Determining projects to restore...
11 3.421   Restored /source/src/myWebApp.csproj (in 1.02 sec).
11 19.42   Restored /source/tests/tests.csproj (in 17.05 sec).
11 27.91   myWebApp -> /source/src/bin/Debug/net6.0/myWebApp.dll
11 28.47   tests -> /source/tests/bin/Debug/net6.0/tests.dll
11 28.49 Test run for /source/tests/bin/Debug/net6.0/tests.dll (.NETCoreApp,Version=v6.0)
11 28.67 Microsoft (R) Test Execution Command Line Tool Version 17.3.3 (x64)
11 28.67 Copyright (c) Microsoft Corporation.  All rights reserved.
11 28.68
11 28.97 Starting test execution, please wait...
11 29.03 A total of 1 test files matched the specified pattern.
11 32.07
11 32.08 Passed!  - Failed:     0, Passed:     1, Skipped:     0, Total:     1, Duration: < 1 ms - /source/tests/bin/Debug/net6.0/tests.dll (net6.0)
11 DONE 32.2s
```

## Configure CI/CD for your .NET application

### Prerequisites

Complete all the previous sections of this guide, starting with Containerize a .NET application. You must have a GitHub account and a Docker account to complete this section.

### Overview

In this section, you'll learn how to set up and use GitHub Actions to build and test your Docker image as well as push it to Docker Hub. You will complete the following steps:

1. Create a new repository on GitHub.
2. Define the GitHub Actions workflow.
3. Run the workflow.

### Step one: Create the repository

1. Create a GitHub repository, configure the Docker Hub secrets, and push your source code.
2. Create a new repository on GitHub.
3. Open the repository Settings, and go to Secrets and variables > Actions.
4. Create a new secret named DOCKER_USERNAME and your Docker ID as value.
5. Create a new Personal Access Token (PAT) for Docker Hub. You can name this token tutorial-docker.
5. Add the PAT as a second secret in your GitHub repository, with the name DOCKERHUB_TOKEN.
6. In your local repository on your machine, run the following command to change the origin to the repository you just created. Make sure you change your-username to your GitHub username and your-repository to the name of the repository you created.
    ```bash
    git remote set-url origin https://github.com/your-username/your-repository.git
    ```
7. In your local repository on your machine, run the following command to rename the branch to main.
    ```bash
    git branch -M main
    ```
8. Run the following commands to stage, commit, and then push your local repository to GitHub.
    ```
    git add -A
    git commit -m "my first commit"
    git push -u origin main
    ```

### Step two: Set up the workflow
Set up your GitHub Actions workflow for building, testing, and pushing the image to Docker Hub.
1. Go to your repository on GitHub and then select the Actions tab.
2. Select **set up a workflow yourself**. 

    This takes you to a page for creating a new GitHub actions workflow file in your repository, under ``.github/workflows/main.yml`` by default.
3. In the editor window, copy and paste the following YAML configuration.
```yml
name: ci

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v4
      -
        name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      -
        name: Build and test
        uses: docker/build-push-action@v5
        with:
          context: .
          target: build
          load: true
      -
        name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          target: final
          tags: ${{ secrets.DOCKER_USERNAME }}/${{ github.event.repository.name }}:latest
```

### Step three: Run the workflow
Save the workflow file and run the job.

1. Select **Commit changes...** and push the changes to the ``main`` branch.
    
    After pushing the commit, the workflow starts automatically.
2. Go to the Actions tab. It displays the workflow.

    Selecting the workflow shows you the breakdown of all the steps.
3. When the workflow is complete, go to your repositories on Docker Hub.

    If you see the new repository in that list, it means the GitHub Actions successfully pushed the image to Docker Hub.