# Kubernetes guide <!-- omit in toc -->

[⬅️ Back to index](.)

## Contents <!-- omit in toc -->

- [References](#references)
- [Components](#components)
  - [Pods and Deployments](#pods-and-deployments)
    - [Deployment example](#deployment-example)
  - [Services](#services)
    - [Service example](#service-example)
    - [Service with external ip](#service-with-external-ip)
  - [Volumes](#volumes)
    - [HostPath example](#hostpath-example)
    - [PersistentVolumeClaim example](#persistentvolumeclaim-example)
    - [NFS example](#nfs-example)
  - [Secrets and ConfigMaps](#secrets-and-configmaps)
    - [Secret example](#secret-example)
    - [ConfigMap example](#configmap-example)
  - [Tensorflow Jobs](#tensorflow-jobs)

## References

For a more detailed guide on Kubernetes concepts and components, refer to the [official documentation](https://kubernetes.io/docs/concepts/) or, if you prefer a video guide, there is the Tech with Nana [Kubernetes crash course](https://youtu.be/X48VuDVv0do) or [the detailed playlist](https://youtube.com/playlist?list=PLy7NrYWoggjwPggqtFsI_zMAwvG0SqYCb).

Since Kubernetes is in active developement, the syntax may change. Refer to the official API documentation for the correct syntax: [Kubernetes API syntax v1.26](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/)

Another useful reference is the [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).

## Components

### Pods and Deployments

Pods are the smallest unit of deployment in Kubernetes. A pod is an abstraction over containers that allows Kubernetes to be run with different container runtimes such as Docker, Containerd and many others.

Even though pods are the smallest unit of deployment, they are not meant to be deployed directly. Instead, they are meant to be deployed through a Deployment, which is a higher level abstraction that provides more features such as rolling updates, rollbacks, and scaling.

#### Deployment example

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx          # deployment name
spec:
  replicas: 1               # number of pods to be created
  selector:
    matchLabels:
      app: nginx-test       # pod label to match
  template:
    metadata:
      labels:
        app: nginx-test     # pod label to be matched
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

The above example is a yaml file which describes a deployment named `test-nginx` with one pod. This pod is labeled with `app: nginx-test` and has only 1 container named `nginx-container` which uses the `nginx` image.

In order to actually create this deployment, we need to run the following command:

```bash
kubectl apply -f deployment.yaml
```

This will create the deployment and the pod. To check if the deployment was created successfully, run:

```bash
kubectl get deployments
```

which will show the deployment we just created:

```bash
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
test-nginx   1/1     1            1           2m
```

To check if the pod was created successfully, run:

```bash
kubectl get pods
```

which will show the pod we just created:

```bash
NAME                          READY   STATUS    RESTARTS   AGE
test-nginx-5f7b8b9b7b-2j2xg   1/1     Running   0          2m
```

To check the logs of the pod, run:

```bash
kubectl logs test-nginx-5f7b8b9b7b-2j2xg
```

To delete the deployment, run:

```bash
kubectl delete -f deployment.yaml
```

### Services

Services are a way to expose pods to the outside world.

There are 3 types of services:

- ClusterIP: Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType.
- NodePort: Exposes the service on each Node’s IP at a static port. A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting `NodeExternalIP:NodePort`.
- LoadBalancer: Exposes the service externally using a cloud provider’s load balancer. NodePort and ClusterIP services, to which the external load balancer will route, are automatically created.

#### Service example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service   # service name
spec:
  type: NodePort
  selector:
    app: nginx-test     # pod selector
  ports:
    - port: 80          # ClusterIP port
      targetPort: 80    # pod port
      nodePort: 31234   # node port (optional)
```

The above example is a yaml file which describes a service named `nginx-service` of type `NodePort` which exposes the port 80 of the pods labeled with `app: nginx-test` on the port 31234 of the node they'll be running on.

In order to actually create this service, we need to add this yaml to a new file and apply it or we can just add it at the end of the deployment yaml file like this:

```yaml
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx          # deployment name
spec:
  replicas: 1               # number of pods to be created
  selector:
    matchLabels:
      app: nginx-test       # pod label to match
  template:
    metadata:
      labels:
        app: nginx-test     # pod label to be matched
    spec:
      containers:
        - name: nginx-container
          image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service   # service name
spec:
  type: NodePort
  selector:
    app: nginx-test     # pod selector
  ports:
    - port: 80          # ClusterIP port
      targetPort: 80    # pod port
      nodePort: 31234   # node port (optional)
```

then run:

```bash
kubectl apply -f deployment.yaml
```

This will create the deployment, the pod and the service. To check if the service was created successfully, run:

```bash
kubectl get services
```

which will show the service we just created:

```bash
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        17d
nginx-service   NodePort    10.108.230.171   <none>        80:31234/TCP   12s
```

To see where the pod is running run:

```bash
kubectl get pods -o wide
```

which will show the pod we just created and the node it is running on:

```bash
NAME                                               READY   STATUS    RESTARTS         AGE   IP            NODE           NOMINATED NODE   READINESS GATES
nfs-subdir-external-provisioner-7d87bdc799-z7dlr   1/1     Running   608 (100m ago)   17d   10.244.1.57   kitt-pic4ser   <none>           <none>
test-nginx-7dfcdc54d6-8v6s2                        1/1     Running   0                40m   10.244.1.76   kitt-pic4ser   <none>           <none>
```

To see the ip address of the node run:

```bash
kubectl get nodes -o wide
```

then you can access the service by going to the ip address of the node on the port 31234. The service will redirect your request to any (in this case the only) pod that is running.

#### Service with external ip

On every service type there is the option to specify an external ip property. This lets you choose where the service is accessible from and not only from the machine the pod is running on. Note that Kubernetes doesn't validate the ip so it's responsibility of the user to make sure the ip is valid.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service   # service name
spec:
  selector:
    app: nginx-test     # pod selector
  ports:
    - port: 80          # ClusterIP port
      targetPort: 80    # pod port
  externalIPs:
    - x.x.x.x           # external ip
```

This will create a ClusterIP service which is also accessible from the ip `x.x.x.x` on port 80 (`spec.ports.port`).

### Volumes

Volumes are used to store data in a pod. There are 3 types of volumes:

- HostPath: A directory on the host node’s filesystem.
- PersistentVolumeClaim: A request for storage by a user.
- NFS: A directory on a NFS server.

The first one mounts a directory on the host into the pod. This is similar to how Docker mounts volumes but since Kubernetes schedules pods on different nodes, the directory will be the one on the node the pod is running on. This means that if the pod is restarted on a different node, the data will be lost on the pod point of view.
The second one is used to request storage space from a previously defined volume or storage class such as a NFS server.
The third one is used to mount a directory on a NFS server into the pod. This is similar to the first one but the directory is on a NFS server.

Each of these types have a use. The first one is useful to mount large quantities of files stored in directories into pods that are bound to a specific host. The second one is useful to store application data that needs to be persistent across pod restarts, such as Grafana database. The third one is useful to mount directories on a NFS server into pods and we'll use this to mount ML data into pods later.

#### HostPath example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx          # deployment name
spec:
  replicas: 1               # number of pods to be created
  selector:
    matchLabels:
      app: nginx-test       # pod label to match
  template:
    metadata:
      labels:
        app: nginx-test     # pod label to be matched
    spec:
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:     # volume mount into container
            - name: data
                            # mount path in container
              mountPath: /usr/share/nginx/html
      volumes:              # volume mount into pod
        - name: data
          hostPath:
            path: /data     # host path
```

This example uses the test nginx deployment from previous examples with volume mounts. There are 2 sections with volumes. The most external one in the yaml file (`spec.template.spec.volumes`) is the mount of the volume into the pod. It has 2 main properties: name and volume definition, in this case `hostPath`. The name is used to reference the volume in the container mount (`spec.template.spec.containers.volumeMounts`).

#### PersistentVolumeClaim example

Provided that a default storage class is defined, the following yaml file will create a PersistentVolumeClaim and a pod that mounts it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx          # deployment name
spec:
  replicas: 1               # number of pods to be created
  selector:
    matchLabels:
      app: nginx-test       # pod label to match
  template:
    metadata:
      labels:
        app: nginx-test     # pod label to be matched
    spec:
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:     # volume mount into container
            - name: data
                            # mount path in container
              mountPath: /usr/share/nginx/html
      volumes:              # volume mount into pod
        - name: data
          persistentVolumeClaim:
            claimName: test-pvc
```

Here the things to note are:

- The `spec.accessModes` property in the PersistentVolumeClaim definition. This defines how the volume can be accessed. In this case it is `ReadWriteOnce` which means that the volume can be mounted as read-write by a single node. More info on access modes [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).
- The `spec.resources.requests.storage` property in the PersistentVolumeClaim definition. This defines the size of the volume. In this case it is 1Gi.
- The pod volume mount has `persistentVolumeClaim` volume definition with the `claimName` property set to the name of the PersistentVolumeClaim.

#### NFS example

Provided there is an NFS server already running, the following yaml file will create a pod that mounts a directory on the NFS server.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx          # deployment name
spec:
  replicas: 1               # number of pods to be created
  selector:
    matchLabels:
      app: nginx-test       # pod label to match
  template:
    metadata:
      labels:
        app: nginx-test     # pod label to be matched
    spec:
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:     # volume mount into container
            - name: nfs-test
                            # mount path in container
              mountPath: /mnt/test
      volumes:              # volume mount into pod
        - name: nfs-test
          nfs:
            server: kitt2.polito.it # NFS server
            path: /export/kubernetes/test # path on the NFS server
```

### Secrets and ConfigMaps

Secrets and ConfigMaps are used to store sensitive data such as passwords and configurations. They are stored in the Kubernetes API and are mounted as files in the pod. The difference between the two is that ConfigMaps are used to store non-sensitive data such as configuration files or key-value pairs.

#### Secret example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
data:
  username: dXNlcm5hbWU=
  password: cGFzc3dvcmQ=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx-container
          image: nginx
          env:
            - name: USERNAME_VAR
              valueFrom:
                secretKeyRef:
                  name: test-secret
                  key: username
            - name: PASSWORD_VAR
              valueFrom:
                secretKeyRef:
                  name: test-secret
                  key: password
```

This example creates a secret with 2 key-value pairs and a pod that mounts the secret as environment variables. The values in secrets are base64 encoded.

Another useful example of secret usage is for certificate files. The following example creates a secret and mounts it as a volume in the pod.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
data:
  secret.file: |
    c29tZXN1cGVyc2VjcmV0IGZpbGUgY29udGVudHMgbm9ib2R5IHNob3VsZCBzZWU=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:
            - name: secret-volume
              mountPath: /etc/secret-volume
      volumes:
        - name: secret-volume
          secret:
            secretName: test-secret
```

If you have a certificate file in your local machine, you can create a secret with it using the following command:

```bash
kubectl create secret generic test-secret --from-file=secret.file=/path/to/certificate/file
```

where test-secret is the name of the secret and secret.file is the name of the file in the secret. If the key is not specified, the name of the file will be used.

#### ConfigMap example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
data:
  key1: value1
  key2: value2
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap-file
data:
  file1: |
    some
    multiline
    text
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx-container
          image: nginx
          env:
            - name: KEY1_VAR
              valueFrom:
                configMapKeyRef:
                  name: test-configmap
                  key: key1
            - name: KEY2_VAR
              valueFrom:
                configMapKeyRef:
                  name: test-configmap
                  key: key2
          volumeMounts:
            - name: configmap-volume
              mountPath: /etc/configmap-volume
      volumes:
        - name: configmap-volume
          configMap:
            name: test-configmap-file
```

ConfigMaps are the same as Secrets but they are not base64 encoded. They can be used to store configuration files or key-value pairs.

### Tensorflow Jobs


