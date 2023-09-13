# Alfalfa Helm Chart with CoSim container

[Alfalfa](https://github.com/NREL/alfalfa) is an open source web application forged in the melting pot of Building Energy Modeling (BEM), Building Controls, and Software Engineering domain expertise.​ Alfalfa transforms a Building Energy Models (BEMs) into virtual buildings by providing industry standard building control interfaces for interacting with models as they run.​ From a software engineering perspective, Alfalfa leverages widely adopted open source products and is architected according to best practices for a robust, modular, and scalable architecture.

[CoSimAlfalfa](https://github.com/NEU-ABLE-LAB/CoSimAlfalfa/) is a container that allows one to run building simulation experiments with the digital twins of occupants.

## Introduction

This helm chart installs a Alfalfa web application instance that interacts with the CoSim pod on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager. 

## Prerequisites

- Kubernetes 1.8+ cluster: [Docker desktop with Kubernetes integration](https://docs.docker.com/desktop/kubernetes/)
- Cosim container access: [CoSimAlfalfa](https://github.com/NEU-ABLE-LAB/CoSimAlfalfa)

## Installing metrics server
To check the resources used by each pod in deployment, metrics server service is used. 
1. Download the Metrics Server deployment YAML file using `PowerShell`, you can use the Invoke-WebRequest `cmdlet`:

      ```
      Invoke-WebRequest -Uri "https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml" -OutFile ".\components.yaml"
      ```

2. Deploy Metrics Server: After downloading the YAML file, you can deploy Metrics Server to your cluster using kubectl apply. Make sure your current directory is the same as where the components.yaml file is downloaded.
   ```
   kubectl apply -f .\components.yaml
   ```
3. Verify Metrics Server is running: You can check if Metrics Server is running using kubectl get pods. This command will list all the pods running in the kube-system namespace, and you can check for the metrics-server.
   ```
   kubectl get pods -n kube-system
   ```

4. Allow kubeclt insecure tls to allow metrics server to record resource usage:

   - Get your existing deployment configuration:
      ```
      kubectl get deployment metrics-server -n kube-system -o yaml > metrics-server 
      ```
   - Edit the metrics-server-deployment.yaml file in a text editor. Find the spec.template.spec.containers.args section, and add - --kubelet-insecure-tls to the arguments.
   
      The result should look something like this:
      ```
         containers:
         - args:
         - --cert-dir=/tmp
         - --secure-port=4443
         - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
         - --kubelet-use-node-status-port
         - --metric-resolution=15s
         - --kubelet-insecure-tls  # <-- Add this line
      ```
   - Apply the updated configuration:
      ```
      kubectl apply -f metrics-server-deployment.yaml
      ```
      This change should tell the metrics server to ignore certificate errors when connecting to the kubelet's metrics endpoint. However, it will make your metrics-server connection insecure and should only be used for development purposes only.
   
5. Get resources: The kubectl top pod command should start working once the Metrics Server is up and running, and has had a few minutes to collect initial data.
   ```
   kubectl top pod -n cosim
   ```
Please note that these instructions assume that you're running a cluster where you have sufficient permissions to install add-ons. If you're using a managed Kubernetes service provided by a cloud vendor, the process to install Metrics Server may be different and you may need to refer to the vendor's documentation or contact their support for help.
## Cosim image update

**NOTE: YOU NEED TO HAVE ACCESS TO NEUABLELAB ACCOUNT ON DOCKERHUB AND COSIMALFALFA REPO ON GIT. Please contact the ABLE_LAB team for questions.**

1. Edit the [CoSimMain.py](https://github.com/NEU-ABLE-LAB/CoSimAlfalfa/blob/main/cosim/src/CoSimMain.py) file: 
   Prior to running the helm chart, the number of building models to be simulated and the number of parallel processes to be run needs to defined in the cosim container. (Sorry, We are working on automating this).

   To define the number of models and parallel processes: update the following lines of code in [CoSimMain.py](https://github.com/NEU-ABLE-LAB/CoSimAlfalfa/blob/main/cosim/src/CoSimMain.py) file:
   ```
   num_models = 10 # Total number of tasks to be done
   num_parallel_process = 5 # Tasks to be done simultaneously
   ```
   SAVE THE CHANGES!

   The `num_parallel_process` parameter here can be decided based on the number of logical cores of your system.
2. Build the docker image: Using cmd, cd into the [cosim](https://github.com/NEU-ABLE-LAB/CoSimAlfalfa/tree/main/cosim) directory and use the following command to build the image with the tag of `neuablelab/cosim`
   ```
   docker build -t neuablelab/cosim .
   ```
3. Push the docker image: Using cmd, the docker image can be pushed to neuablelab image repo with the command:
   ```
   docker push neuablelab/cosim
   ```
## Installing the Chart

To install the helm chart with the chart name `alfalfa`, you can run the following commands (below) in the root directory of this repo:
For quick install using the default settings, you can launch the cluster using on the options below. For local deployment (e.g. docker-desktop or minikube)
```
helm install simulation ./alfalfa-chart --set worker.replicas=5 --values ./alfalfa-chart/values_resources_minimal.yaml
```
**Note: `--set worker.replicas=5` is based on the number of parallel simuations to be run which was defined [Cosim image update](#cosim-image-update) section of this readme.** 
## Uninstalling the Chart

To uninstall/delete the `simulation` helm chart:
```
helm delete simulation
```
The command removes all the Kubernetes components associated with the chart and deletes the release *including* persistent volumes. See more about persistent volumes below. *Make sure to run this command before deleting the k8s cluster itself as many cloud providers retain the persistent volumes on k8s deletion. The helm delete command will remove everything. 

## Accessing alfalfa

Once you install the chart it'll take a couple of minutes to start all the containers. You can verify all the Kubernetes pods are up in running by running the following. As the pods are deployed under the namespace `cosim` (can be found in [values.yaml](alfalfa-chart/values.yaml) or [values_resources_minimal.yaml](alfalfa-chart/values_resources_minimal.yaml)): 
```
kubectl get pods -n cosim
```
example output of all pods running: 
```
NAME                       READY   STATUS             RESTARTS      AGE
cosim-zmv24                1/1     Running            0             55s
goaws-5d44c4bbf8-9rfl8     1/1     Running            0             55s
influxdb-9b977fb99-trqtw   1/1     Running            0             54s
mc-597b7cf4cb-g6r9g        0/1     CrashLoopBackOff   2 (22s ago)   55s
minio-54649779c9-8c2zw     1/1     Running            0             54s
mongo-67d886bf9c-dbndz     1/1     Running            0             55s
redis-6d97d96786-rz7m7     1/1     Running            0             55s
web-69b65844c6-q5xps       1/1     Running            0             54s
worker-57cc98d55f-2hkg6    1/1     Running            0             54s
worker-57cc98d55f-2xzxm    1/1     Running            0             54s
worker-57cc98d55f-6pjcq    1/1     Running            0             54s
worker-57cc98d55f-hxthn    1/1     Running            0             54s
worker-57cc98d55f-mx4f2    1/1     Running            0             55s
```
To get resources used by the pods, considering that you have installed the metrics server as mentioned above, you can use the top command:
```
kubectl top pods -n cosim
```
example output of resources used by pods in `cosim` namespace:
```
NAME                       CPU(cores)   MEMORY(bytes)
cosim-zmv24                235m         1421Mi
goaws-5d44c4bbf8-9rfl8     1m           11Mi
influxdb-9b977fb99-trqtw   1m           34Mi
minio-54649779c9-8c2zw     1m           326Mi
mongo-67d886bf9c-dbndz     257m         122Mi
redis-6d97d96786-rz7m7     735m         92Mi
web-69b65844c6-q5xps       244m         136Mi
worker-57cc98d55f-2hkg6    633m         959Mi
worker-57cc98d55f-2xzxm    636m         965Mi
worker-57cc98d55f-6pjcq    635m         957Mi
worker-57cc98d55f-hxthn    646m         962Mi
worker-57cc98d55f-mx4f2    632m         953Mi
```
## Persistent Volumes

This helm chart provisions persistent storage for the database (MongoDB) and the cosim container (Outputs the simulations runs dataframes in `.parquet.gzip` files). The database volume will persist throughout the life of the helm chart while it's running and will **NOT** persist if you delete the helm chart. The cosim container's volume will will persist, more specifically the path listed in `cosim.pv.spec.hostPath.path` which can be found in [values_resources_minimal.yaml](alfalfa-chart/values_resources_minimal.yaml) file.   

<!-- ## Auto Scaling

The worker pods __can__ be configured to auto-scale based on CPU threshold (default 50%). So once the aggregate CPU for all worker pods exceeds the defined threshold (in this case 50%), the Kubernetes engine will start adding additional worker pods up to the maximum specified. This is also dependent on how the Kuebernetes cluster was configured as additional VM node instances will also be added. Please refer to the notes on [aws](/aws/README.md) and [google](/google/README.md) when setting up the cluster and note the instance type and maximum nodes specified.   -->
