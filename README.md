
# Deploying a containerized web application

This tutorial shows you how to package a web application in a Docker container image, and run that container image on a Google Kubernetes Engine (GKE) cluster as a load-balanced set of replicas that can scale to the needs of your users.

## Objectives

To package and deploy your application on GKE, you must:

1. Package your app into a Docker image.
2. Upload the image to a registry.
3. Run the container locally on your machine (optional).
4. Create a container cluster.
5. Deploy your app to the cluster.
6. Expose your app to the internet.
7. Scale up your deployment.
8. Deploy a new version of your app.

## Before you begin

Take the following steps to enable the Kubernetes Engine API:

1. Visit the [Kubernetes Engine page](https://console.cloud.google.com/projectselector/kubernetes) in the Google Cloud Console.
2. Create or select a project.
3. Wait for the API and related services to be enabled. This can take several minutes.
4. Make sure that billing is enabled for your Google Cloud project. [Learn how to confirm billing is enabled for your project](/billing/docs/how-to/modify-project).

### Option A: Use Cloud Shell

You can follow this tutorial using [Cloud Shell](/shell), which comes preinstalled with the `gcloud`, `docker`, and `kubectl` command-line tools used in this tutorial. If you use Cloud Shell, you don't need to install these command-line tools on your workstation.

To use Cloud Shell:

1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Click the `Activate Shell Button` at the top of the Console window.

    A Cloud Shell session opens inside a new frame at the bottom of the console and displays a command-line prompt.

    ```bash
    Cloud Shell session
    ```

### Option B: Use command-line tools locally

If you prefer to follow this tutorial on your workstation, you need to install the following tools:

1. [Install the Google Cloud SDK](/sdk/docs/quickstarts), which includes the `gcloud` command-line tool.
2. Using the `gcloud` command line tool, install the [Kubernetes](https://kubernetes.io) command-line tool. `kubectl` is used to communicate with Kubernetes, which is the cluster orchestration system of GKE clusters:
    ```console
    gcloud components install kubectl
    ```

3. Install [Docker Community Edition (CE)](https://docs.docker.com/engine/installation/) on your workstation. You will use this to build a container image for the application.

4. Install the [Git source control](https://git-scm.com/downloads) tool to fetch the sample application from GitHub.

## Step 1: Build the container image

GKE accepts Docker images as the application deployment format. To build a Docker image, you need to have an application and a Dockerfile.

For this tutorial, you will deploy a [sample web application](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/hello-app) called `hello-app`, a web server written in [Go](https://golang.org/) that responds to all requests with the message “Hello, World!” on port 80.

The application is packaged as a Docker image, using the [Dockerfile](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/hello-app/Dockerfile) that contains instructions on how the image is built. You will use this Dockerfile to package your application.

1. Download the `hello-app` source code by running the following commands:

    ```console
    git clone https://github.com/GoogleCloudPlatform/kubernetes-engine-samples
    cd kubernetes-engine-samples/hello-app
    ```

2. Set the `PROJECT_ID` environment variable to your [Google Cloud project ID](/resource-manager/docs/creating-managing-projects#identifying_projects) (<var>project-id</var>). The `PROJECT_ID` variable will be used to associate the container image with your project's [Container Registry](/container-registry).

    ```console
    export PROJECT_ID=<var>project-id</var>
    ```

3. Build the container image of this application and tag it for uploading:

    ```console
    docker build -t gcr.io/${PROJECT_ID}/hello-app:v1 .
    ```

    This command instructs Docker to build the image using the `Dockerfile` in the current directory and tag it with a name, such as `gcr.io/my-project/hello-app:v1`. The `gcr.io` prefix refers to [Container Registry](/container-registry), where the image will be hosted. Running this command does not upload the image yet.

4. Run the `docker images` command to verify that the build was successful:
    ```console
    docker images
    ```

    Output:
    ```console
    REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
    gcr.io/my-project/hello-app    v1                  25cfadb1bf28        10 seconds ago      54 MB
    ```

## Step 2: Upload the container image

You need to upload the container image to a registry so that GKE can download and run it.

1. Configure the Docker command-line tool to authenticate to [Container Registry](/container-registry) (you need to run this only once):

    ```consolegcloud auth configure-docker
    ```

2. You can now use the Docker command-line tool to upload the image to your Container Registry:

    ```console
    docker push gcr.io/${PROJECT_ID}/hello-app:v1
    ```

## Step 3: Run your container locally (optional)

1. Test your container image using your local Docker engine:

    ```console
    docker run --rm -p 8080:8080 gcr.io/${PROJECT_ID}/hello-app:v1
    ```

2. If you're using Cloud Shell, click the **Web Preview** button ![Web Preview Button](/shell/docs/images/web_preview.svg) and then select the `8080` port number. GKE opens the preview URL on its proxy service in a new browser window.

3. Otherwise, open a new terminal window (or a Cloud Shell tab) and run to verify if the container works and responds to requests with "Hello, World!":

    ```console
    curl http://localhost:8080
    ```

    Once you've seen a successful response, you can shut down the container by pressing **Ctrl+C** in the tab where the `docker run` command is running.

## Step 4: Create a container cluster

Now that the container image is stored in a registry, you need to create a [cluster](/kubernetes-engine/docs/concepts/cluster-architecture) to run the container image. A cluster consists of a pool of [Compute Engine VM instances](/compute) running [Kubernetes](https://kubernetes.io), the open source cluster orchestration system that powers GKE.

Once you have created a GKE cluster, you use Kubernetes to deploy applications to the cluster and manage the applications' lifecycle.

1. Set your [project ID](/resource-manager/docs/creating-managing-projects#identifying_projects) and [Compute Engine zone](/compute/docs/zones#available) options for the `gcloud` tool:

    ```console
    gcloud config set project $PROJECT_ID
    gcloud config set compute/zone <var>compute-zone</var>
    ```

2. Create a two-node cluster named `hello-cluster`:
    ```console
    gcloud container clusters create hello-cluster --num-nodes=2
    ```

    It may take several minutes for the cluster to be created.

3. After the command completes, run the following command to see the cluster's two worker VM instances:

    ```console
    gcloud compute instances list
    ```

    Output:
    ```console
    NAME                                          ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
    gke-hello-cluster-default-pool-07a63240-822n  us-central1-b  n1-standard-1               10.128.0.7   35.192.16.148   RUNNING
    gke-hello-cluster-default-pool-07a63240-kbtq  us-central1-b  n1-standard-1               10.128.0.4   35.193.136.140  RUNNING
    ```

<aside class="note">**Note:** If you are using an existing Google Kubernetes Engine cluster or if you have created a cluster through Google Cloud Console, you need to run the following command to retrieve cluster credentials and configure `kubectl` command-line tool with them:

```gcloud container clusters get-credentials hello-cluster```

If you have already created a cluster with the `gcloud container clusters create` command listed above, this step is not necessary.

## Step 5: Deploy your application

To deploy and manage applications on a GKE cluster, you must communicate with the Kubernetes cluster management system. You typically do this by using the `kubectl` command-line tool.

Kubernetes represents applications as [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/), which are units that represent a container (or group of tightly-coupled containers). The Pod is the smallest deployable unit in Kubernetes. In this tutorial, each Pod contains only your `hello-app` container.

The `kubectl create deployment` command below causes Kubernetes to create a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) named `hello-web` on your cluster. The Deployment manages multiple copies of your application, called replicas, and schedules them to run on the individual nodes in your cluster. In this case, the Deployment will be running only one Pod of your application.

1. Run the following command to deploy your application:

    ```kubectl create deployment hello-web --image=gcr.io/${PROJECT_ID}/hello-app:v1```

2. To see the Pod created by the Deployment, run the following command:

    ```kubectl get pods```

    Output:
    ```NAME                         READY     STATUS    RESTARTS   AGE
    hello-web-4017757401-px7tx   1/1       Running   0          3s
    ```

## Step 6: Expose your application to the internet

By default, the containers you run on GKE are not accessible from the internet because they do not have external IP addresses. You must explicitly expose your application to traffic from the internet.

Run the following command to expose your application to traffic from the internet:

<devsite-code>

```kubectl expose deployment hello-web --type=LoadBalancer --port 80 --target-port 8080
```

</devsite-code>

This command creates a [Service](https://kubernetes.io/docs/user-guide/services/) resource, which provides networking and IP support to your application's Pods. GKE creates an external IP and a Load Balancer ([subject to billing](/compute/pricing#lb)) for your application.

The `--port` flag specifies the port number configured on the Load Balancer, and the `--target-port` flag specifies the port number that the `hello-app` container is listening on.

<aside class="note">**Note:** <span>GKE assigns the external IP address to the **Service** resource—not the Deployment. If you want to find out the external IP that GKE provisioned for your application, you can inspect the Service with the `kubectl get service` command:<devsite-code>

```kubectl get service
```

Output:

```NAME         CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
hello-web    10.3.251.122    203.0.113.0     80:30877/TCP     3d
```

Once you've determined the external IP address for your application, make note of the EXTERNAL-IP address. Point your browser to this URL (such as `http://203.0.113.0`) to check if your application is accessible.

## Step 7: Scale up your application

You add more replicas to your application's Deployment resource by using the `kubectl scale` command.

1. Add two additional replicas to your Deployment (for a total of three):

    ```kubectl scale deployment hello-web --replicas=3
    ```

2. View the new replicas running on your cluster:

    ```kubectl get deployment hello-web
    ```

    Output:

    ```NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    hello-web   3         3         3            2           1m```

3. View the Pods for your deployment:

    ```kubectl get pods
    ```

    Output:

    ```NAME                         READY     STATUS    RESTARTS   AGE
    hello-web-4017757401-ntgdb   1/1       Running   0          9s
    hello-web-4017757401-pc4j9   1/1       Running   0          9s
    hello-web-4017757401-px7tx   1/1       Running   0          1m```

Now, you have multiple instances of your application running independently of each other and you can use the `kubectl scale` command to adjust capacity of your application.

The load balancer you provisioned in the previous step will start routing traffic to these new replicas automatically.

## Step 8: Deploy a new version of your app

GKE's rolling update mechanism ensures that your application remains up and available even as the system replaces instances of your old container image with your new one across all the running replicas.

1. You can create an image for the v2 version of your application by building the same source code and tagging it as v2 (or you can change the `"Hello, World!"` string to `"Hello, GKE!"` before building the image):

    ```docker build -t gcr.io/${PROJECT_ID}/hello-app:v2 .```

2. Push the image to the Container Registry:

    ```docker push gcr.io/${PROJECT_ID}/hello-app:v2```

3. Apply a rolling update to the existing deployment with an image update:

    ```kubectl set image deployment/hello-web hello-app=gcr.io/${PROJECT_ID}/hello-app:v2```

4. Visit your application again at `http://<var>external-ip</var>`, and observe the changes you made take effect.

## Cleaning up

To avoid incurring charges to your Google Cloud Platform account for the resources used in this tutorial:

After completing this tutorial, perform these steps to remove the following resources and prevent unwanted charges incurring on your account:

1. **Delete the Service:** This deallocates the Cloud Load Balancer created for your Service:

    ```kubectl delete service hello-web```

2. **Delete the cluster:** This deletes the resources that make up the cluster, such as the compute instances, disks and network resources:

    ```gcloud container clusters delete hello-cluster```

## What's next

* Read the [Load Balancers](/kubernetes-engine/docs/tutorials/http-balancer) tutorial, which demonstrates advanced load balancing configurations for web applications.

* Learn how to store persistent data in your application through the [MySQL and WordPress](/kubernetes-engine/docs/tutorials/persistent-disk) tutorial.

* Configure [static IP and domain name](/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) for your application.

* Explore other [Kubernetes Engine tutorials](/kubernetes-engine/docs/tutorials).

* Try out other Google Cloud features for yourself. Have a look at our [tutorials](/docs/tutorials).
