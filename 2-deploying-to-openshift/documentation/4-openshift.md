# Lab 4 - Deploying to OpenShift


## Kubernetes deployment configuration

Now we examine the **deployment** and **service** yaml. The yamls do contain the deployment of the container to a **Pod** and creation of the **Services** to access the **Authors mircoservice** in the Kubernetes Cluster. 

In the following image we see the relevant dependencies for this lab.

![authors-java-service-pod-container](images/authors-java-service-pod-container.png)

### 1. Deployment

The deployment will deploy the container to a Pod in Kubernetes.
For more details we use the [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) for Pods.

> A Pod is the basic building block of Kubernetes–the smallest and simplest unit in the Kubernetes object model that you create or deploy. A Pod represents processes running on your Cluster .


Let's start with the **deployment yaml**. For more details we use will the [Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) for deployments.

Definition of the ```kind``` defines this is a ```Deployment``` configuration.

```yml
kind: Deployment
apiVersion: apps/v1beta1
metadata:
  name: authors
```

Inside the ```spec``` section, we give the deployment a app name and version label.

```yml
spec:
  ...
  template:
    metadata:
      labels:
        app: authors
        version: v1
```

Then we define a ```name``` for the container and we provide the concret container ```image``` location, e.g. where the container can be found in the **Container Registry**. 

_NOTE:_ We will replace ```authors:1``` later with the IBM Container Registry information. 

The ```containerPort``` depends on the port definition inside our **Dockerfile** or better in our **server.xml**.

Now we should remember the usage of **HealthEndpoint** class for our **Authors**, here we see the ```livenessProbe``` definition.


```yml
spec:
      containers:
      - name: authors
        image: authors:1
        ports:
        - containerPort: 3000
        livenessProbe:
```

This is the full [deployment.yaml](../authors-java-jee/deployment/deployment.yaml) file.

```yaml
kind: Deployment
apiVersion: apps/v1beta1
metadata:
  name: authors
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: authors
        version: v1
    spec:
      containers:
      - name: authors
        image: authors:1
        ports:
        - containerPort: 3000
        livenessProbe:
          exec:
            command: ["sh", "-c", "curl -s http://localhost:3000/"]
          initialDelaySeconds: 20
        readinessProbe:
          exec:
            command: ["sh", "-c", "curl -s http://localhost:3000/health | grep -q authors"]
          initialDelaySeconds: 40
      restartPolicy: Always
```

### 2. Service

After the definition of the **Pod** we need to define how to access the Pod, therefor we use a **service** in Kubernetes. For more details we use the [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/) for service.

> A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service. The set of Pods targeted by a Service is (usually) determined by a Label Selector.

In the service we map the **NodePort** of the cluster to the port 3000 of the **Authors** service running in the **authors** Pod, as we can see in the following picture. 

_Note:_ Later we get the actual port for the service using the command line: ```nodeport=$(kubectl get svc authors --ignore-not-found --output 'jsonpath={.spec.ports[*].nodePort}')```.

![authors-java-service-pod-container](images/authors-java-service-pod-container.png)

In the [service.yaml](../authors-java-jee/deployment/service.yaml) we find our selector to the Pod **authors**. If the service is deployed, it is possible that our **Articles** service can find the **Authors** service.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: authors
  labels:
    app: authors
spec:
  selector:
    app: authors
  ports:
    - port: 3000
      name: http
  type: NodePort
---
```

## Container Registry in OpenShift

> [OpenShift Container Platform](https://docs.openshift.com/container-platform/3.11/architecture/infrastructure_components/image_registry.html) provides an integrated container image registry called OpenShift Container Registry (OCR) that adds the ability to automatically provision new image repositories on demand. This provides users with a built-in location for their application builds to push the resulting images.

Open the OpenShift Container Registry (OCR) in the IBM Cloud

1. Logon to IBM Cloud web console

2. Select **Open Shift** in the menu

![Select Open Shift in the menu](images/os-registry-01.png)

3. Chose **Clusters** and click on your **OpenShift cluster**

![Chose Clusters and click on your OpenShift cluster](images/os-registry-02.png)

4. Open the **OpenShift web console**

![Open the OpenShift web console](images/os-registry-03.png)

5. Select in **My Projects** the **default** project

![Select in My Projects the default project](images/os-registry-04.png)

6. Expand in **Overview** the **DEPLOYMENT registry-console** and click **Routes - External Traffic**

![Expand in Overview the DEPLOYMENT registry-console and click Routes - External Traffic](images/os-registry-05.png)

7. In the container registry you will find later the **authors** image and you can click on the latest label.

![In the container registry you will find later the authors image](images/os-registry-06.png)

8. In the container image details you will find the command, how you can pull the docker image to your local PC ```sudo docker pull docker-registry.default.svc:5000/cloud-native-starter/authors:latest```

![docker images details](images/os-registry-07.png)

## Deploy to OpenShift

**Push code and build image**

```
$ cd ${ROOT_FOLDER}/2-deploying-to-openshift
$ oc new-project cloud-native-starter
$ oc new-build --name authors --binary --strategy docker
$ oc start-build authors --from-dir=.
```

**Deploy microservice**

```
$ cd ${ROOT_FOLDER}/deploying-to-openshift/deployment
$ oc apply -f deployment-os.yaml
$ oc apply -f service.yaml
$ oc expose svc/authors
$ open http://$(oc get route authors -o jsonpath={.spec.host})/openapi/ui/
$ curl -X GET "http://$(oc get route authors -o jsonpath={.spec.host})/api/v1/getauthor?name=Niklas%20Heidloff" -H "accept: application/json"
```