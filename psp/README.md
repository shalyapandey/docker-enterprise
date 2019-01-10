# Pod Security Policies in UCP 3.1

[Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/), or PSP, enable fine-grained authorization of pod creation and updates. It is a native Kubernetes feature implemented as an [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).

Controls you can implement with PSP:

- Running of priviledged containers
- Usage of host namespaces
- Usage of host networking and ports
- Usage of volume types
- Usage of the host filesystem
- The user and group IDs of the container
- Restricting escalation to root privileges
- White list of FlexVolume drivers
- Linux Capibilitites
- The SELinux context of the container

Let's go through the steps to enable and use PSP in UCP 3.1:

1. Enable it as an Admission Controller. You need to edit the UCP config file. **WARNING: This operation can be disruptive to running applications! You will see why in the following steps. PSP will disable any new pod or updaing existing deployments without the proper set-up after you enable it. So make sure you look into PSP's impact before you enable it!**

    First get the current configuration file:

    ```
    # Assuming you downloaded UCP client bundle, and you're in the directory containing the certs
    curl --cacert ca.pem --cert cert.pem --key key.pem https://UCP_HOST/api/ucp/config-toml > ucp-config.toml

    ```

    Then edit `ucp-config.toml` to add PSP as a custom kube api server flag option which is under the cluster config section in the config. You must ensure that any custom flags do not conflict with configuration options already set by UCP. 


    ```
    [cluster_config]

    <...SNIPPET...>
    custom_kube_api_server_flags = ["--enable-admission-plugins=PodSecurityPolicy"]
    ```

    Then apply the changes using the following command:

    ```
    curl --cacert ca.pem --cert cert.pem --key key.pem --upload-file ucp-config.toml https://UCP_HOST/api/ucp/config-toml
    ```


2. Confirm you can see the config as an added option to the kube api server container:

    ```
    $ 🐳  → docker ps -a | grep ucp-kube-apiserver
    ip-10-56-43-25/ucp-kube-apiserver                                                                                                             docker/ucp-hyperkube:3.0.2                                       37 minutes ago ago       10.56.43.25:6443->6443/tcp                                                                  Up 37 minutes                           "/bin/apiserver_entr…"
    ip-10-56-24-174/ucp-kube-apiserver                                                                                                            docker/ucp-hyperkube:3.0.2                                       37 minutes ago ago       10.56.24.174:6443->6443/tcp                                                                 Up 37 minutes                           "/bin/apiserver_entr…"
    ip-10-56-0-143/ucp-kube-apiserver                                                                                                             docker/ucp-hyperkube:3.0.2                                       37 minutes ago ago       10.56.0.143:6443->6443/tcp                                                                  Up 37 minutes                           "/bin/apiserver_entr…"

    🐳  → docker inspect ip-10-56-43-25/ucp-kube-apiserver | jq .[0].Config.Cmd
    [
    "/bin/apiserver_entrypoint.sh",
    "--enable-admission-plugins=Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota,PodNodeSelector,UCPAuthorization,CheckImageSigning,UCPNodeSelector",
    "--advertise-address=172.33.0.224",
    "--allow-privileged=true",
    "--anonymous-auth=false",
    "--requestheader-username-headers=X-Remote-User",
    "--requestheader-group-headers=X-Remote-Group",
    "--requestheader-extra-headers-prefix=X-Remote-Extra-",
    "--requestheader-client-ca-file=/var/lib/docker/ucp/ucp-node-certs/ca.pem",
    "--authorization-mode=RBAC,Webhook",
    "--authorization-webhook-cache-authorized-ttl=0s",
    "--authorization-webhook-cache-unauthorized-ttl=0s",
    "--authorization-webhook-config-file=/etc/authorization_config.cfg",
    "--audit-policy-file=/etc/audit_policy.yaml",
    "--audit-webhook-config-file=/etc/audit_webhook.cfg",
    "--client-ca-file=/var/lib/docker/ucp/ucp-node-certs/ca.pem",
    "--etcd-servers=https://172.33.0.224:12379",
    "--etcd-cafile=/var/lib/docker/ucp/ucp-node-certs/ca.pem",
    "--etcd-certfile=/var/lib/docker/ucp/ucp-node-certs/cert.pem",
    "--etcd-keyfile=/var/lib/docker/ucp/ucp-node-certs/key.pem",
    "--experimental-encryption-provider-config=/var/lib/docker/ucp/ucp-node-certs/encryption.cfg",
    "--insecure-port=0",
    "--kubelet-certificate-authority=/var/lib/docker/ucp/ucp-node-certs/ca.pem",
    "--kubelet-client-certificate=/var/lib/docker/ucp/ucp-node-certs/cert.pem",
    "--kubelet-client-key=/var/lib/docker/ucp/ucp-node-certs/key.pem",
    "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
    "--runtime-config=admissionregistration.k8s.io/v1alpha1",
    "--secure-port=12388",
    "--service-account-key-file=/var/lib/docker/ucp/ucp-node-certs/service_account_key.pem",
    "--service-cluster-ip-range=10.96.0.0/16",
    "--service-node-port-range=32768-35535",
    "--tls-cert-file=/var/lib/docker/ucp/ucp-node-certs/cert.pem",
    "--tls-private-key-file=/var/lib/docker/ucp/ucp-node-certs/key.pem",
    "--proxy-client-cert-file=/var/lib/docker/ucp/ucp-node-certs/cert.pem",
    "--proxy-client-key-file=/var/lib/docker/ucp/ucp-node-certs/key.pem",
    "--cloud-provider=aws",
    "--bind-address=0.0.0.0",
    "--enable-admission-plugins=PodSecurityPolicy"
    ]

    ```

3. Apply the PSP policies. There are good examples [here](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) and [here](https://docs.bitnami.com/kubernetes/how-to/secure-kubernetes-cluster-psp/). For example , the following policy prevents any pod from running as root. Remember that PSP are not namespaced! So they can be shared across multiple namespaces in your clusters.


    ```
    🐳  → cat restrict-root.yml

    apiVersion: extensions/v1beta1
    kind: PodSecurityPolicy
    metadata:
    name: restrict-root
    spec:
    privileged: false
    runAsUser:
        rule: MustRunAsNonRoot
    seLinux:
        rule: RunAsAny
    fsGroup:
        rule: RunAsAny
    supplementalGroups:
        rule: RunAsAny
    volumes:
    - '*'
    ```
    ```
    🐳  → kubectl apply -f restrict-root.yml
    podsecuritypolicy.extensions "restrict-root" configured
    $ kubectl get psp
    NAME            DATA      CAPS      SELINUX    RUNASUSER          FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
    restrict-root   false               RunAsAny   MustRunAsNonRoot   RunAsAny   RunAsAny   false            *

    ```
4. Before you can launch any pod with this policy, you need to give access to the pod to use the policy. By default, once PSP is enabled, a pod MUST match a Pod Security Policy they have access to. Hence, before a pod is launched directly or through a deployment or replicaset, we need to give it permission to use a PSP. We do that using a RoleBinding that associates the pod's serviceaccount to the PSP. We need to use the pod's service account because this will ensure that regardless how the pod got launched (as a pod, or through a replicaset or deployment) it still would have access to the PSP. The following steps are needed to do that:

    - Create a `Role` to give access to **use** the `restrict-root` PSP that we created. The `Role` needs to be created in a namespace. In this case, I'm using a random namespace called `blue`, which is the namespace that we intend to launch the pod in.

    ```
    🐳  → cat role.yml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: Role
    metadata:
    name: psp-role
    namespace: blue
    rules:
    - apiGroups:
        - ""
        resources:
        - podsecuritypolicy
        resourceName:
        - restrict-root
        verbs:
        - use
    ```
    ```
    🐳  → kubectl apply -f role.yml
    ```
    - Create a `RoleBinding` to associate the `psp-role` to a service account. In this example i'm using the `default` serviceaccount of the `blue` namespace. If you want to use a custom service account, you can.

    ```
    🐳  → cat rolebinding.yml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
        name: blue:psp-role
        namespace: blue
    subjects:
        - kind: ServiceAccount
          name: default
          apiGroup: ""
    roleRef:
        kind: Role 
        name: psp-role 
        apiGroup: ""
    ```
    ```
    🐳  → kubectl apply -f rolebinding.yml
    ```
5. Now we can deploy an sample application to test if the PSP is in effect. Let's take a look at this docker demo deployment.

```
🐳  → cat deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: blue
  name: dockerdemo-deployment
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
      serviceAccountName: default
      containers:
      - image: ehazlett/docker-demo
        name: docker-demo-container
        env:
        - name: app
          value: dockerdemo
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```
```
🐳  → kubectl apply -f deployment.yml
```

Let's check if the deployment worked as expeted,

```
🐳  → kubectl get pod -n blue
NAME                                    READY     STATUS                       RESTARTS   AGE
dockerdemo-deployment-7f84d6b89-hmjf9   0/1       CreateContainerConfigError   0          49s

 🐳  → kubectl describe pod dockerdemo-deployment-7f84d6b89-hmjf9 -n blue
Name:               dockerdemo-deployment-7f84d6b89-hmjf9
Namespace:          blue
Priority:           0
PriorityClassName:  <none>
Node:               ip-172-33-37-245.us-west-2.compute.internal/172.33.37.245
Start Time:         Thu, 10 Jan 2019 04:59:28 +0000
Labels:             app=dockerdemo
                    pod-template-hash=394082645
Annotations:        kubernetes.io/psp=restrict-root
Status:             Pending
IP:                 192.168.219.211
Controlled By:      ReplicaSet/dockerdemo-deployment-7f84d6b89
Containers:
  docker-demo-container:
    Container ID:
    Image:          ehazlett/docker-demo
    Image ID:
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       CreateContainerConfigError
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:     250m
      memory:  64Mi
    Environment:
      app:  dockerdemo
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-qjsnm (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  default-token-qjsnm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-qjsnm
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age               From                                                  Message
  ----     ------     ----              ----                                                  -------
  Normal   Scheduled  1m                default-scheduler                                     Successfully assigned blue/dockerdemo-deployment-7f84d6b89-hmjf9 to ip-172-33-37-245.us-west-2.compute.internal
  Normal   Pulling    10s (x7 over 1m)  kubelet, ip-172-33-37-245.us-west-2.compute.internal  pulling image "ehazlett/docker-demo"
  Normal   Pulled     9s (x7 over 1m)   kubelet, ip-172-33-37-245.us-west-2.compute.internal  Successfully pulled image "ehazlett/docker-demo"
  Warning  Failed     9s (x7 over 1m)   kubelet, ip-172-33-37-245.us-west-2.compute.internal  Error: container has runAsNonRoot and image will run as root

```

You can see from the `Events` section that the pod was not created because it uses an image that runs as root. The fact that we're seeing this error means that PSP kicked in and that the pod has the proper access to use PSP policies.

6. Let's fix this issue by setting a pod secruity context to run the container as non-root. The new deployment file looks like this:

```
 🐳  → cat deployment.non-root.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: blue
  name: dockerdemo-deployment
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
      serviceAccountName: default
      containers:
      - image: ehazlett/docker-demo
        name: docker-demo-container
        env:
        - name: app
          value: dockerdemo
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        securityContext:
          runAsUser: 1000
```

After deploying the new YAML, if we take a look at the pods, we can see that the pod successfully deployed.

```
🐳  → kubectl get pods -n blue
NAME                                     READY     STATUS    RESTARTS   AGE
dockerdemo-deployment-649d94df7f-kgg7t   1/1       Running   0          29s
```

 Any other deployments that are launched in that namespace with the default service account should be able to access and use that PSP. 

 ## Multiple Policies

 If a user or a service account has access to multiplpe PSPs, the Pod Security Policy Admission Controller will choose the first policy _alphabetically_. Which means you either limit the number of PSPs a user/service account has access to to only 1 so that it can be the only one that can be used. Alternatively, you can name your PSP in a way that would ensure that the PSP with more priviledges is selected first.
