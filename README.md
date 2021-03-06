# GNCR portal
This repo contains the static website and sample code/config for deploying apps (e.g. R Shiny, python) to a Kubernetes cluster in IBM cloud, optionally authenticating via AppId.

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Structure](#structure)
- [Setup](#setup)
	- [Prerequisites](#prerequisites)
	- [Connect to the cluster](#connect-to-the-cluster)
	- [Create a local ingress deployment file](#create-a-local-ingress-deployment-file)
- [Initial deployment steps](#initial-deployment-steps)
- [Adding a new app](#adding-a-new-app)
	- [Creating a docker image](#creating-a-docker-image)
	- [Deploying to Kubernetes and configuring Ingress](#deploying-to-kubernetes-and-configuring-ingress)
- [Updating the web app](#updating-the-web-app)
- [Updating a dashboard app](#updating-a-dashboard-app)

<!-- /TOC -->

## Structure
* `app-deployment` - contains deployment files for each app, plus a sample file
* `ingress` - static web app which links to the individual apps, and contains instructions on deploying the on Kubernetes with Ingress, with URL Rewrite and integration with App ID for authentication.
* `original-tutorial` - contains the original tutorial materials from [https://github.com/holken1/deploying-r-on-cloud](https://github.com/holken1/deploying-r-on-cloud), which was the starting point for our deployment, but no longer reflects how things are working. They may contain useful information if the instructions below fail to work.

## Setup

### Prerequisites
 * You have followed the [IBM cloud CLI installation instructions](https://cloud.ibm.com/docs/cli?topic=cli-install-devtools-manually) to install:
   * IBM Cloud CLI
   * Docker (read the [full instructions](https://docs.docker.com/desktop/) to get it working on Windows)
   * Kubernetes command line tools
   * IBM Cloud Container Registry CLI plug-in
   * IBM Cloud Kubernetes Service CLI plug-in
 * You have (or someone in the team has) created a Kubernetes cluster in IBM cloud.
 * You have (or someone in the team has) created an App ID instance in IBM cloud (if using).
 * You have cloned this repository and have a terminal/Powershell/shell open, cd'd to the top-level directory.

### Connect to the cluster

```bash

# Login to ibmcloud and container registry
ibmcloud login
ibmcloud cr login

# check that we are connected
kubectl cluster-info

# (if not connected, run:)
ibmcloud ks cluster ls
ibmcloud ks cluster config --cluster <cluster_name_or_ID>
```

### Create a local ingress deployment file

```bash
# Create ingress configuration file
cp ingress/ingress.yaml.example.noauth ingress/ingress.yaml
# For bash:
ibmcloud ks cluster get --cluster <cluster_name_or_ID> | grep Ingress
# For Powershell:
ibmcloud ks cluster get --cluster <cluster_name_or_ID> | Select-String Ingress
```

Edit `ingress.yaml` to fill in host and secret with the values shown by the previous command ('Ingress Subdomain' and 'Ingress Secret').

## Initial deployment steps

These are only used to set up the cluster initially. They should not be repeated unless creating a new cluster.

If using AppID (we're not currently using it), run:

```bash

# Bind to AppId
ibmcloud ks cluster service bind --cluster <cluster_name_or_ID> --namespace <namespace> --service <App_ID_instance_name>

# Edit ingress.yaml (copied from ingress.yaml.example/appid) to fill in bindSecret with the values shown by the previous command
```

In all cases, run:

```
# Deploy the web app
kubectl apply -f ingress/deploy-web.yaml
# Deploy any other apps (see next section)
kubectl apply -f app-deployment/deploy-<app-name>.yaml

# Deploy Ingress
kubectl apply -f ingress/ingress.yaml

# view the result
kubectl get deployments
kubectl get pods
kubectl get services
```

## Adding a new app

### Creating a docker image

To add an app to the Kubernetes cluster, you will need to get it running in a [docker](https://www.docker.com) image, by creating a Dockerfile. The Dockerfile contains the commands needed to be run in order to get the app running in a container. There is a sample Dockerfile for an R shiny app at [app-deployment/Dockerfile-shiny-example](app-deployment/Dockerfile-shiny-example); for python or other apps there are likely to be example Dockerfiles online.

1. Add a Dockerfile to your app's code repository, modifying it as necessary if you're copying a sample file.
2. Build the image (replacing `<app-image-name>` with the name of your app):
    ```bash
    docker build . -t <app-image-name>
    ```
3. Run the image (replacing `<port>` with the port your app runs on, e.g. `3838` for Shiny):
    ```bash
    docker run -p <port>:<port> <app-image-name>
    ```
4. Point a browser at `http://localhost:<port>` and check your app is working as expected.
5. Create a [dockerhub](https://hub.docker.com) account (if you don't already have one) and follow their [quickstart guide](https://docs.docker.com/docker-hub/) to log in with the command line client, but create a public rather than private repository (see below if you want to keep it private).
6. Push your image to docker:
    ```bash
    docker tag <app-image-name> <your_username>/<app-image-name>
    docker push <your_username>/<app-image-name>
    ```
7. Commit your Dockerfile to your repository.

You may wish to [enable automated builds](https://docs.docker.com/docker-hub/builds/) on Dockerhub so that when you push to GitHub, a new docker image will automatically be built.

**NB: If you don't want your docker image to be publicly available, you can create a single private repository with a free dockerhub account. However, you will need to follow steps 1 and 2 of [these additional instructions](https://developer.ibm.com/tutorials/accessing-dockerhub-repos-in-iks/#deploy-the-application-from-your-private-dockerhub) to deploy from the private repository.**

### Deploying to Kubernetes and configuring Ingress

1. Copy the file [app-deployment/deploy-example.yaml](app-deployment/deploy-example.yaml), renaming it as appropriate.
2. Replace all instances of `<app-name>` with your app name, and `<dockerhub-image-name>` with the full path of your image in dockerhub, e.g. `alisonrclarke/covid19seir:v0.1.0`. It is a good idea to use either versioned tags or SHA digests (e.g. `alisonrclarke/dashboard-test@sha256:bd0590f1b17106217e131bbf8d7b58e1c12a5f5bd88d3d6470c29e8e81dde3ff`) to identify the build (rather than always using the `latest` tag), so we can make the Kubernetes cluster aware that an image has changed.
3. Apply your new configuration file:
    ```bash
    kubectl apply -f app-deployment/deploy-<app-name>.yaml
    ```
4. Edit [ingress/ingress.yaml.example.noauth](ingress/ingress.yaml.example.noauth) as follows:

    1. Update the annotations under `metadata/annotations`:
        1. Append `;serviceName=<app-name>-service rewrite=/` to the `rewrite-path` annotation, ensuring there are no spaces around the semicolon.

	```yml
    metadata:
      name: gncr-ingress-noauth
      annotations:
        ingress.bluemix.net/rewrite-path: "... rewrite=/;serviceName=<app-name>-service rewrite=/"
        ...    
    ```

    2. Add a new `path` section under `spec/rules/host/http/paths`, replacing `<app-name>` with your app name and `<port>` with the port number (as in the deployment file):
    	```yml
        spec:
          ...
          rules:
            - host: <host>
              http:
                paths:
                - path: /<app-name>/
                  backend:
                    serviceName: <app-name>-service
                    servicePort: <port>
                ...
        ```
5. Copy these `ingress.yaml.example.noauth` file changes to your local copy of `ingress.yaml`.
6. Apply the updated configuration file:
    ```bash
    kubectl apply -f ingress/ingress.yaml
    ```
7. Visit the service frontend and sign in, then manually alter the URL to add `/<app-name>/` to the end, to check your new app works.
8. Commit the changes to the yaml files.
9. Edit [ingress/web/index.html](ingress/web/index.html) to add a new link under the "Links to tools" section, replacing `app-name` and `Detailed app title`:
    ```html
    <p>Links to tools:</p>
    <ul>
        ...
        <li><p><a href="/app-name">Detailed app title</a></p></li>
    <ul>
    ```
10. Check the html page looks OK in a browser (the new link won't work, however), then commit your changes and push them to GitHub.
11. See [Updating the web app](#updating-the-web-app) to deploy the changes.


## Updating the web app
1. Make your changes in the ingress/web folder. Commit them and push to GitHub.
2. Go to [https://hub.docker.com/repository/docker/alisonrclarke/dashboard-test/builds](https://hub.docker.com/repository/docker/alisonrclarke/dashboard-test/builds) - a new build should have started from your latest commit. Wait until it has finished, then go to the **Tags** tab to view the build. copy the latest tag. Click the link to the SHA digest, and copy the full digest.
3. Open [ingress/deploy-web.yaml](ingress/deploy-web.yaml) and paste in the SHA digest under `spec/template/spec/containers/image`:
    ```yml
    spec:
      ...
      template:
        ...
        spec:
          containers:
          - name: dashboard
            image: alisonrclarke/dashboard-test@sha256:bd0590f1b17106217e131bbf8d7b58e1c12a5f5bd88d3d6470c29e8e81dde3ff
    ```
4. Deploy the new static web app:
    ```bash
    kubectl apply -f ingress/deploy-web.yaml
    ```
5. Refresh the main portal page and check your link appears and works.

## Updating a dashboard app
1. Make your changes to the app and build a new docker image. (If your repository is connected to dockerhub the image will be built automatically when you put a commit to GitHub or create a new tag.)
2. Edit the yaml file in `app-deployment` that relates to your app, e.g. `deploy-complexit.yaml`, to change the version number or SHA hash under the `image` section, e.g. `ducovid19tools/covid19seir:v0.1.0` (version number, for tagged image) or `ducovid19tools/covid19seir@sha256:bd0590f1b17106217e131bbf8d7b58e1c12a5f5bd88d3d6470c29e8e81dde3ff` (SHA hash, for untagged image) e.g.:
    ```yml
    spec:
      ...
      template:
        ...
        spec:
          containers:
          - name: complexit
            image: ducovid19tools/covid19seir:<new-version-number>
    ```
3. Apply the updated configuration file:
    ```bash
    kubectl apply -f app-deployment/deploy-<app-name>.yaml
    ```
4. Visit the service frontend to check your updates are in place and working.
5. Commit your change to the yaml file and push to GitHub.
