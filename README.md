# GNCR portal
This repo contains the static website and sample code/config for deploying apps (e.g. R Shiny, python) to a Kubernetes cluster in IBM cloud, authenticating via AppId.

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Structure](#structure)
- [Initial deployment steps](#initial-deployment-steps)
	- [Prerequisites](#prerequisites)
	- [Steps](#steps)
- [Adding a new app](#adding-a-new-app)
	- [Creating a docker image](#creating-a-docker-image)
	- [Deploying to Kubernetes and configuring Ingress](#deploying-to-kubernetes-and-configuring-ingress)
- [Updating the web app](#updating-the-web-app)

<!-- /TOC -->

## Structure
* `app-deployment` - contains deployment files for each app, plus a sample file
* `ingress` - static web app which links to the individual apps, and contains instructions on deploying the on Kubernetes with Ingress, with URL Rewrite and integration with App ID for authentication.
* `original-tutorial` - contains the original tutorial materials from [https://github.com/holken1/deploying-r-on-cloud](https://github.com/holken1/deploying-r-on-cloud), which was the starting point for our deployment, but no longer reflects how things are working. They may contain useful information if the instructions below fail to work.

## Initial deployment steps

### Prerequisites
 * You have installed the Kubernetes command link tools, and the IBM cloud CLI with the Kubernetes plugin.
 * You have created a Kubernetes cluster in IBM cloud.
 * You have created an App ID instance in IBM cloud.

### Steps

```bash

ibmcloud cr login

# check that we are connected
kubectl cluster-info

# Bind to appid
ibmcloud ks cluster-service-bind --cluster <cluster_name_or_ID> --namespace <namespace> --service <App_ID_instance_name>

# Create ingress configuration file
cp ingress/ingress.yaml.example ingress/ingress.yaml
# Edit ingress.yaml to fill in bindSecret, host and secret

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
4. Edit [ingress/ingress.yaml.example](ingress/ingress.yaml.example) as follows:

    1. Update the annotations under `metadata/annotations`:
        1. Append `;serviceName=<app-name>-service rewrite=/` to the `rewrite-path` annotation, ensuring there are no spaces around the semicolon.
		2. Modify the `appid-auth` annotation to append `,<app-name>-service` to the `serviceName=` parameter.

	```yml
    metadata:
      name: gncr-ingress-appid
      annotations:
        ingress.bluemix.net/rewrite-path: "... rewrite=/;serviceName=<app-name>-service rewrite=/"
        ingress.bluemix.net/appid-auth: "bindSecret=... serviceName=...,<app-name>-service,web-service idToken=false"
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
5. Copy these `ingress.yaml.example` file changes to `ingress.yaml` so that others can redeploy without losing your app.
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
