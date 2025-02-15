
# Building and Deploying a Dockerized Python Application with GitLab CI/CD on AWS EC2

![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/pipelinegitlab1.png?raw=true)

## Step 1: Set Up the Environment

**Action:** Set up Ubuntu inside VirtualBox on a Windows laptop, installed Docker and GitLab Runner inside the Ubuntu VM.

**Problem:**
- **Issue:** Docker-in-Docker (DinD) was not running due to Docker socket permissions.

**Solution:** To enable Docker commands within the GitLab CI/CD pipeline, we had to run Docker in privileged mode and mount the Docker socket.

**Commands:**

- Configure GitLab Runner:

    ```bash
    sudo nano /etc/gitlab-runner/config.toml
    ```

- Modify the `config.toml` file to enable privileged mode and Docker socket binding:

    ```toml
    [runners.docker]
      privileged = true
      volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
    ```

- Restart GitLab Runner:

    ```bash
    sudo gitlab-runner restart
    ```

---

## Step 2: Install and Configure GitLab Runner

**Action:** Installed GitLab Runner on Ubuntu, configured it to use Docker as the executor.

**Problem:**
- **Issue:** GitLab Runner required privileged mode to execute Docker commands.

**Solution:** Enabled privileged mode and mounted the Docker socket in the GitLab Runner configuration.

**Commands:**

- Install GitLab Runner:

    ```bash
    sudo apt-get install gitlab-runner
    ```

- Register GitLab Runner:

    ```bash
    sudo gitlab-runner register
    ```

- Specify Docker executor during registration process.

---

## Step 3: Set Up GitLab Repository and Pipeline Configuration

**Action:** Created a GitLab repository and added a `.gitlab-ci.yml` file for CI/CD configuration with stages: test, build, and deploy.

**Problem:**
- **Issue:** Missing Dockerfile or incorrect paths caused pipeline failures.

**Solution:** Used a debugging step to list the contents of the `build/` directory to confirm the presence of the Dockerfile.

**Commands:**

- Check directory structure:

    ```bash
    ls -l build/
    ```

---

## Step 4: Configure GitLab CI/CD Variables

**Action:** Created three GitLab CI/CD variables to securely store credentials:
- `REGISTRY_USER`: Docker Hub username.
- `REGISTRY_PASS`: Docker Hub password.
- `SSH_KEY`: Private SSH key for EC2 instance access.

**Problem:**
- **Issue:** Docker login in the pipeline failed due to missing authentication details.

**Solution:** Configured GitLab CI/CD variables for Docker Hub credentials and the SSH key for EC2 access.

---

## Step 5: Build the Docker Image

**Action:** Created the `build-image` job in the `.gitlab-ci.yml` file to build the Docker image using the Dockerfile located in the `build/` directory.

**Problem:**
- **Issue:** The Dockerfile path was not correctly specified or the file was missing.

**Solution:** Debugged the directory structure and ensured the correct path for the Dockerfile.

**Commands:**

- Build Docker image:

    ```bash
    docker build -t $IMAGE_NAME:$IMAGE_TAG -f build/Dockerfile .
    ```

- Push Docker image to Docker Hub:

    ```bash
    docker push $IMAGE_NAME:$IMAGE_TAG
    ```

---

## Step 6: Deploy the Application on EC2

**Action:** Added a deploy job to deploy the Docker container to an EC2 instance via SSH, using Docker commands on the EC2 instance.

**Problem:**
- **Issue:** Permission issues with Docker commands (e.g., stopping and removing containers) on the EC2 instance.

**Solution:** We used `sudo` to ensure that Docker commands were executed with the correct privileges on the EC2 instance.

**Commands:**

- SSH into EC2 and execute Docker commands:

    ```bash
    ssh -o StrictHostKeyChecking=no -i $SSH_KEY ubuntu@54.210.250.138 "
      sudo docker login -u $REGISTRY_USER -p $REGISTRY_PASS &&
      sudo docker ps -aq | xargs sudo docker stop || true &&
      sudo docker ps -aq | xargs sudo docker rm || true &&
      sudo docker pull $IMAGE_NAME:$IMAGE_TAG &&
      sudo docker run -d -p 5000:5000 $IMAGE_NAME:$IMAGE_TAG"
    ```

---

## Step 7: Add SSH Key for EC2 Access

**Action:** Stored the private SSH key as a GitLab variable (`SSH_KEY`) for secure access to the EC2 instance.

**Problem:**
- **Issue:** The SSH key wasn’t properly referenced or had incorrect file permissions, causing connection issues.

**Solution:** Set the SSH key file to the correct permissions and ensured it was properly referenced in the `.gitlab-ci.yml` file.

**Commands:**

- Set SSH key file permissions:

    ```bash
    chmod 400 $SSH_KEY
    ```

---

## Step 8: Configure EC2 Security Group

**Action:** Modified the EC2 security group to allow inbound traffic on port 5000 (for the application) and port 22 (for SSH access).

**Problem:**
- **Issue:** The EC2 instance was not accessible externally due to restrictive security group settings.

**Solution:** We added inbound rules to the EC2 security group for ports 5000 and 22 to enable access for the application and SSH.

---

## Step 9: Debugging and Fixes

**Action:** Debugged the pipeline and resolved issues related to Docker login warnings, EC2 permissions, and container management on the EC2 instance.

**Problem:**
- **Issue 1:** Docker login warning about insecure password handling.

**Solution:** Ignored the warning, but recommended configuring a secure credential helper to store Docker Hub credentials.

- **Issue 2:** Permission issues when stopping and removing containers.

**Solution:** Used `sudo` with Docker commands in the EC2 deployment script to ensure the commands were executed with root privileges.

- **Issue 3:** The `docker ps` command failed without any containers to stop.

**Solution:** Added `|| true` after `xargs sudo docker stop` and `xargs sudo docker rm` to handle cases where no containers were running.

---

## Final Pipeline YAML File

```yaml
variables:
  IMAGE_NAME: mzshabban/python-demo
  IMAGE_TAG: python-app-1.0

stages:
  - test
  - build
  - deploy

test-job:
  stage: test
  image: python:3.9-slim-buster
  before_script:
    - apt-get update && apt-get install -y make
  script:
    - make test  # Running the tests

build-image:
  stage: build
  image: docker:20.10.16
  before_script:
    - docker login -u "$REGISTRY_USER" -p "$REGISTRY_PASS"
  script:
    - ls -l build/  # Debugging, check if Dockerfile is correct
    - docker build -t $IMAGE_NAME:$IMAGE_TAG -f build/Dockerfile .  # Build Docker image
    - docker push $IMAGE_NAME:$IMAGE_TAG  # Push image to Docker Hub
  services: []

deploy:
  stage: deploy
  image: alpine  # Use a lightweight image for SSH commands
  before_script:
    - apk add --no-cache openssh-client  # Install SSH client
    - chmod 400 $SSH_KEY  # Ensure SSH key permissions are correct
  script:
    - ssh -o StrictHostKeyChecking=no -i $SSH_KEY ubuntu@54.210.250.138 "
        sudo docker login -u $REGISTRY_USER -p $REGISTRY_PASS &&
        sudo docker ps -aq | xargs sudo docker stop || true &&
        sudo docker ps -aq | xargs sudo docker rm || true &&
        sudo docker pull $IMAGE_NAME:$IMAGE_TAG &&
        sudo docker run -d -p 5000:5000 $IMAGE_NAME:$IMAGE_TAG"
  only:
    - main
```

---
![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/pythondemoapp1.png?raw=true)
## Conclusion

We successfully set up a GitLab CI/CD pipeline for deploying a Dockerized Python application to AWS EC2. We faced several issues along the way, such as Docker permissions, SSH key configuration, Docker login warnings, and security group misconfigurations. By addressing each issue with the solutions above, we managed to deploy the application successfully and automate the process using GitLab CI/CD.
