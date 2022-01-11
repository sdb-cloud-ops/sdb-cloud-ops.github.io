# Deploy SingleStore with Google Cloud Kubernetes Engine

### Prerequisites 

Make sure you have [kubectl](https://kubernetes.io/docs/tasks/tools/) and [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) installed.

Use the `gcloud` tool to configure the following default settings: your default [project](https://cloud.google.com/resource-manager/docs/creating-managing-projects?visit_id=637775161629210129-1131801549&rd=1#identifying_projects), [compute zone](https://cloud.google.com/compute/docs/regions-zones#available), and [compute region](https://cloud.google.com/compute/docs/regions-zones#available).

## Creating and connecting to the Kubernetes cluster

We will use the `gcloud containers create` command to create our `sdb-cluster` using the following optional flags

    [--network=NETWORK]
    [--subnetwork=SUBNETWORK]
    [--region=REGION | --zone=ZONE, -z ZONE] 
    [--image-type=IMAGE_TYPE] 
    [--machine-type=MACHINE_TYPE, -m MACHINE_TYPE] 
    [--node-locations=ZONE,[ZONE,…]] 
    [--enable-autoscaling]
    [--num-nodes NUM_NODES]
    [--min-nodes MIN_NODES]
    [--max-nodes MAX_NODES]

###### Example command
```
gcloud container clusters create sdb-cluster --region=us-east4 --node-locations=us-east4-a --machine-type=n2-standard-16 --image-type=cos --enable-ip-alias --create-subnetwork name=sdb-subnet --enable-autoscaling --num-nodes=4 --min-nodes=0 --max-nodes=10
```
> Note: Warnings are expected in the output at this time.

Run the `gcloud container clusters get-credentials` command to connect to the kubernetes cluster

###### Example command
```
gcloud container clusters get-credentials sdb-cluster --region us-east4 --project memsql-team-se
```
If you have kubectl correctly installed, you should now be able to run `kubectl get ns` without errors.

## Deploy SingleStore on Kubernetes

### Download the Operator image

The `memsql/operator` can be pulled from [Docker Hub](https://hub.docker.com/r/memsql/operator/tags) or from the [Red Hat container registry](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/download-the-memsql-operator.html)

###### Example with Docker
```
docker pull memsql/operator:1.2.5-83e8133a
```

### Create the Object Definition files

Download the [object definition files](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files.html) from our site, and save them in an accessible directory. The [rbac.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/rbac-yaml.html) and the [memsql-cluster-crd.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/memsql-cluster-crd-yaml.html) do not require edits.

Edit the [deployment.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/deployment-yaml.html) to reference the Docker image just pulled.

###### Example spec
```
    spec:
      serviceAccountName: memsql-operator
      containers:
        - name: memsql-operator
          image: memsql/operator:1.2.5-83e8133a
          imagePullPolicy: IfNotPresent
```

In order to edit the [memsql-cluster.yaml](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/memsql-cluster-yaml.html) you will need your license key from the customer [portal](https://auth.singlestore.com/auth/realms/memsql/protocol/openid-connect/auth?client_id=customer-portal-login&redirect_uri=https%3A%2F%2Fportal.singlestore.com%2F&state=0e422fe0-0db1-45d3-a27d-e9b27c64cd82&response_mode=fragment&response_type=code&scope=openid&nonce=4022881b-27c3-406a-b0e0-ba83cd5d9985) and a hashed version of a secure password for the admin user. 

In order to hash a secure password, we have an example python script
```
from hashlib import sha1
print("*" + sha1(sha1('secretpass'.encode('utf-8')).digest()).hexdigest().upper())
```
###### How to use the script
> Save the script in a file named `hash_password.py`, replacing `secretpass` with a secure password. Make sure it is executable with - `chmod +x hash_password.py`, then run the script with `python3 hash_password.py`. It will print the hashed password to the command line, where you can copy and paste directly to the `memsql-cluster.yaml` file within the quotes by the `adminHashedPassword:`
> <br></br>

Create a namespace for the Kubernetes Objects
```
kubectl create ns singlestore
```
Add the namespace to the object metadata
```
apiVersion: memsql.com/v1alpha1
kind: MemsqlCluster
metadata:
  name: memsql-cluster
  namespace: singlestore
```
Add the `licence_key`, `hashed_password`, and use the latest `memsql/node` image
```
spec:
  license: <license_key>
  adminHashedPassword: "<hashed_password>"
  nodeImage:
    repository: memsql/node
    tag: latest
```
Change the redundancy level to `2`
```
  redundancyLevel: 2
```
The rest of the edits to the `aggregatorSpec` and the `leafSpec` are custom depending on how many aggregators and leafs you want and how many resources you want allocated to them. 

### Create the Kubernetes Objects

Create the role-based access control
```
kubectl -n singlestore create -f rbac.yaml
```
Create the memsql custom resource definition
```
kubectl -n singlestore create -f memsql-cluster-crd.yaml
```
Create the Operator deployment
```
kubectl -n singlestore create -f deployment.yaml
```
Now you should be able to run `kubectl -n singlestore get pods` and see that the Operator pod is running.
```
NAME                               READY   STATUS    RESTARTS   AGE
memsql-operator-79874797f4-9t47r   1/1     Running   0          2m50s
```
Create the memsql custom resource
```
kubectl -n singlestore create -f memsql-cluster.yaml
```

