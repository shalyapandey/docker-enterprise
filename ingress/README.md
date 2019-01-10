# Production Ingress Deployment in UCP 3.0

## Overview


Ingress is an API object that manages external access to the services in a cluster, typically HTTP. Ingress requires to be deployed separately in UCP 3.1. You can deploy a variety of ingress controllers but the supported one is the [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx). The following instructions provide highly-available production deployment of the controller.


## Prerequisite


- UCP 3.1.X deployed and properly configured
- Two or Three Dedicated Infra Nodes deployed as UCP worker nodes
- An external load-balancer fronting these nodes with an associated VIP that resolves application DNS (e.g `*.app.docker.mycompany.com`)



### Step 1: Labeling the Infrastructure Nodes

To deploy the NGINX controller on the infra nodes, we need to make sure they're properly labeled. Make sure you have identified 2 or 3 nodes to be the infra nodes that will host the ingress controller. In the following example, we want to use all three nodes: `dockeree-worker-linux-1,dockeree-worker-linux-2,dockeree-worker-linux-3`

```
 🐳  → kubectl get nodes
NAME                           STATUS    ROLES     AGE       VERSION
dockeree-worker-linux-1    Ready     <none>    5d        v1.8.11-docker-8d637ae
dockeree-worker-linux-2    Ready     <none>    5d        v1.8.11-docker-8d637ae
dockeree-worker-linux-3    Ready     <none>    5d        v1.8.11-docker-8d637ae

```

We will use the label `infra.role=ingress` to label the nodes:

```
🐳  → kubectl label node dockeree-worker-linux-1 infra.role=ingress
node "dockeree-worker-linux-1" labeled
🐳  → kubectl label node dockeree-worker-linux-2 infra.role=ingress
node "dockeree-worker-linux-2" labeled
🐳  → kubectl label node dockeree-worker-linux-3 infra.role=ingress
node "dockeree-worker-linux-3" labeled

```


### Step 2: Create the Dedicated Namespace

We will create a dedicated namespace for all things related to infrastructure deployment like the `nginx-controller`. We will also need a service account for the controller to be able to talk to the Kubernetes API. We will then create an RBAC policy to only allow these deployments on the dedicated nodes we have labelled above.

To create a namespace and a service account, simply use the following [YAML file](config/ns-and-sa.yaml) and apply it via CLI or the UI.

```
🐳  → cat ns-and-sa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: infra
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-service-account
  namespace: infra
```

Creating the namespace and service account via `kubectl`:

```
🐳  → kubectl apply -f ns-and-sa.yaml
namespace "infra" created
serviceaccount "nginx-ingress-service-account" created

```


### Step 3: Create RBAC Policy

We need to apply an RBAC role-binding for NGINX controller to access the API Server. We do that with the following [RBAC yaml config](config/ingress-rbac.yaml):


```
🐳  → cat ingress-rbac.yaml
## Source: https://github.com/nginxinc/kubernetes-ingress/blob/master/deployments/rbac/rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-ingress-cluster-role
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
  - update
  - create
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  verbs:
  - update
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-ingress-cluster-rb
subjects:
- kind: ServiceAccount
  name: nginx-ingress-service-account
  namespace: ingress
roleRef:
  kind: ClusterRole
  name: nginx-ingress-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

Then you can apply it with `kubectl`:

```
🐳  → kubectl apply -f ingress-rbac.yaml
```

### Step 4: Deploy NGINX Controller

Now it's time to deploy the NGINX Controller. Note that in the following template, we're using hostPorts for the controller ports. This will expose the host port ( we're picking a port in a high range using `hostPort: 38080`) directly into these nodes. 


```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
  namespace: infra
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        image: gcr.io/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
      nodeSelector:
       infra.role: ingress
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: infra
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: infra
  labels:
    app: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: infra
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: infra
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: infra
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
      annotations:
        prometheus.io/port: '10254'
        prometheus.io/scrape: 'true'
    spec:
      initContainers:
      - command:
        - sh
        - -c
        - sysctl -w net.core.somaxconn=32768; sysctl -w net.ipv4.ip_local_port_range="1024 65535"
        image: alpine:3.6
        imagePullPolicy: IfNotPresent
        name: sysctl
        securityContext:
          privileged: true
      serviceAccountName: nginx-ingress-service-account 
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.10.2
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --annotations-prefix=nginx.ingress.kubernetes.io
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
            hostPort: 38080
            protocol: TCP
          - name: https
            containerPort: 443
            hostPort: 38
            protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
      nodeSelector:
        infra.role: ingress
```

You can deploy it using `kubectl` and check if pods are deployed successfully:

```
🐳  → kubectl apply -f nginx-ingress-deployment.yaml
deployment.extensions "default-http-backend" created
service "default-http-backend" created
configmap "nginx-configuration" created
configmap "tcp-services" created
configmap "udp-services" created
deployment.extensions "nginx-ingress-controller" created

 🐳  → kubectl get pod -n infra -o wide
NAME                                        READY     STATUS    RESTARTS   AGE       IP                NODE
default-http-backend-7ff9774865-hsj46       1/1       Running   0          1m        192.168.145.6     dockeree-worker-linux-1
default-http-backend-7ff9774865-kcqhj       1/1       Running   0          1m        192.168.116.145   dockeree-worker-linux-3
default-http-backend-7ff9774865-xq566       1/1       Running   0          1m        192.168.247.210   dockeree-worker-linux-2
nginx-ingress-controller-6b987cbbc6-4qqz8   1/1       Running   0          1m        192.168.145.7     dockeree-worker-linux-1
nginx-ingress-controller-6b987cbbc6-h6rmg   1/1       Running   0          1m        192.168.116.146   dockeree-worker-linux-3
nginx-ingress-controller-6b987cbbc6-hkw86   1/1       Running   0          1m        192.168.247.211   dockeree-worker-linux-2
```

### Step 5: Deploy an Application and its Ingress Rule

Now that the Controller is successfully deployed, we can start creating Ingress objects to expose our applications externally using it. Let's take an example deployment for the following application. The deployment yaml consists of a `Service` and `Deployment` objects that deploy the application with the Docker image `ehazlett/docker-demo` and create a service associated with it.


```
🐳  → cat dockerdemo.yaml

kind: Service
apiVersion: v1
metadata:
  namespace: default
  name: docker-demo-svc
spec:
  selector:
    app: dockerdemo
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  namespace: default
  name: dockerdemo-deploy
  labels:
    app: dockerdemo
spec:
  selector:
    matchLabels:
      app: dockerdemo
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: dockerdemo
    spec:
      containers:
      - image: ehazlett/docker-demo
        name: docker-demo-container
        env:
        - name: app
          value: dockerdemo
        ports:
        - containerPort: 8080
```


Now let's say we want to expose this application with a valid URL using `dockerdemo.app.docker.example.com`. We have to create an `Ingress` rule do that. The Ingress rule gets picked up by the NGINX controller and is used to generate the proper configuration for NGINX for it to do the proper load-balancing. Here's the ingress rule looks like:

```
🐳  → cat dockerdemo.ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dockerdemo-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: dockerdemo.app.docker.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: docker-demo-svc
          servicePort: 8080
```

```
 🐳  → kubectl apply -f dockerdemo.ingress.yaml
```



You can apply the ingress rules separately or combine them with the application deployment yaml file. Either way will work!

```


 🐳  → kubectl get ingress
NAME                          HOSTS                                          ADDRESS   PORTS     AGE
dockerdemo-ingress   dockerdemo.app.docker.example.com             80        7d
```

Assuming you have already registered a DNS record for your application pointing to the external load-balancer fronting the `infra` nodes, you should be able to access your application using the URL.
