# ArgoCD-Project
Building and deploying a .NET application using GitHub Actions as the CI/CD pipeline and ArgoCD for continuous deployment on a Minikube cluster(k8s cluster). Let's break down the steps to achieve this:

## Setting Up GitHub Repository:

- Create a GitHub repository for .NET application and push the application files to this repository.
- Create another GitHub repository for storing configuration and manifests (e.g., YAML files for Kubernetes) and push the configuration files to this repository. In my case, I am using "config-branch" for storing configuration and mainfestes - to keep this project context at one repo. (But it is recommended to seperate repos for application files and configuration files)

## GitHub Actions for CI:

In the .NET application repository, set up GitHub Actions for CI. Create a .github/workflows/ci.yml file.
- Create GitHub secrets for Docker Hub credentials and GitHub credentials - to reference in the workflow file.
  ![Screenshot from 2023-10-05 14-56-43](https://github.com/KarthikSaladi047/argocd-project/assets/105864615/d4f65e79-64ff-4130-afcc-d244ba8f54ce)
- Configure the CI workflow to build, test, and create a Docker image from our application.
- Push the Docker image to Docker Hub.
- Configure the Image tag in the Configuration repo "helm/webapp/values.yaml" file.
  
Here's is the workflow YAML file:
```
name: ci

env:
  CONFIG_REPO_NAME: argocd-project
  
on:
  push:
    branches:
      - 'master'

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building and testing the app ..."
  docker-build-push:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          file: WebApplication1/Dockerfile
          push: true
          tags: karthiksaladi047/argocd-project:${{ github.sha }}
  promote-to-dev-environment:
    runs-on: ubuntu-latest
    needs: docker-build-push
    steps:
      - run: |
          echo "promoting into dev environment!"
          git config --global user.email ci-bot@argocd.com && git config --global user.name ci-bot
          echo "cloning config repo $CONFIG_REPO_NAME"
          git clone -b config-branch https://oauth2:${{ secrets.GH_PAT }}@github.com/${{ github.repository_owner }}/$CONFIG_REPO_NAME.git
          cd $CONFIG_REPO_NAME
          echo "checkout config branch"
          git checkout config-branch
          echo "updating image tag in values file"
          sed -i "s,tag:.*,tag:\ ${{ github.sha }}," helm/webapp/values.yaml
          git add . && git commit -m "update image tag"
          git push

```
## Docker Hub account

- Create a Docker Hub accout.
- Create a public repository called argocd-project within Docker Hub.
- Add the Username and Auth token of Docker Hub as Git Hub secretes.
![Screenshot from 2023-10-05 15-01-19](https://github.com/KarthikSaladi047/argocd-project/assets/105864615/cd4ee747-53ee-4ace-815a-bc235bacbd9f)

## ArgoCD Configuration Repository:

- Create a separate GitHub repository for storing ArgoCD configuration and manifests - as a best practice.
- We are deploying a helm chart into destination Kubernetes Cluster and image tag withing the values.yaml file gets updated by GitHub Actions CI pipeline everytime a commit is made to master branch.
 ![Screenshot from 2023-10-05 15-03-11](https://github.com/KarthikSaladi047/argocd-project/assets/105864615/5d66347e-29dc-4924-b8ce-b309508d6567)

## ArgoCD on Minikube:

- Install Minikube on your GitHub Codespace if not already installed. Please go through this link for installing minikube: https://minikube.sigs.k8s.io/docs/start/
- Install and configure ArgoCD on your Minikube cluster.
  ```
  kubectl create ns argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl get all -n argocd
  ```
- Inorder to open we UI of Argocd, we can port-forward to access at localhost:8080
  ```
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
- The password for admin account is available as a kubernetes secret ( base64 encoded - need to decode it to use)
  ```
  kubectl get secret -n argocd argocd-initial-admin-secret -o yaml
  ```
## ArgoCD Application Configuration:

- Create an ArgoCD Application resource in our ArgoCD configuration repository to define how application should be deployed.
- Specify the GitHub repository and branch that contains the Kubernetes manifests/helm chart for application.
- Set up sync policies to automatically synchronize changes when the configuration repository is updated.

Here's ArgoCD Application resource YAML:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: webapp
  namespace: argocd
spec:
  destination:
    namespace: webapp
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: helm/webapp
    repoURL: "https://github.com/karthiksaladi047/argocd-project.git"
    targetRevision: config-branch
  syncPolicy:
    automated: 
      prune: true
    syncOptions:
      - CreateNamespace=true
```
Deploy the ArgoCd Application resource.
```
kubectl apply -n argocd -f arg-application.yaml
```
## Verifing CICD Pipeline and Argocd Sync
- Make changes to the application files and commit the changes.
- We can see that the GitHub Action CI Pipeline got triggerd and started running the taskes mentioned in the Workflow ci.yml file.
- We can see a new docker image is pushed to Docker Hub.
- Now we can see that the Image tag in Configuration repo has be modified by ci pipeline.
- ArgoCD sync the changes made to Configuration repo and we can see pod got reployed with new image tag.
