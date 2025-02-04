# Install Kubernetes Cluster from Scratch

**Created with help of:** [Medium Article](https://medium.com/@rabbi.cse.sust.bd/kubernetes-cluster-setup-on-ubuntu-24-04-lts-server-c17be85e49d1)

## Table of Contents

- [Pre-requisites](#pre-requisites)
- [Network configuration](#network-configuration)
- [Install autofs](#install-autofs)
- [Remove swap](#remove-swap)
- [Enable kernel modules and configure sysctl](#enable-kernel-modules-and-configure-sysctl)
- [Install containerd](#install-containerd)
- [Install Kubeadm, Kubectl, and Kubelet](#install-kubeadm-kubectl-and-kubelet)
- [Setup Kubernetes](#setup-kubernetes)
- [Install Pod network add-on Calico](#install-pod-network-add-on-calico)
- [Sanity checks](#sanity-checks)
- [Fix taint node](#fix-taint-node)
- [Enable Nvidia support](#enable-nvidia-support)
- [Install Helm](#install-helm)
- [Install Kubernetes Dashboard using Helm](#install-kubernetes-dashboard-using-helm)
- [Add ingress-nginx](#add-ingress-nginx)
- [Add Let's Encrypt](#add-lets-encrypt)
- [OAuth2 Proxy](#oauth2-proxy)
- [Create Docker Registry](#create-docker-registry)

## Pre-requisites

This manual is based on Ubuntu 24.10 server installation. No extra packages with the exception of ssh.

**DO NOT INSTALL DOCKER!!!**

## Network configuration

| Component | IP Address |
|-----------|------------|
| Kubernetes Server | 172.16.100.9 |
| NFS Server | 172.16.100.8 |

Please adjust your NFS exports in the configs below to suit your needs. Also create your DNS entries where appropiate.

## Install autofs

    sudo apt install autofs

**Create /etc/auto.nfs**
    
    sudo bash -c 'cat << EOF > /etc/auto.nfs
    shared-media	-rw,soft,intr,rsize=32768,wsize=32768 172.16.100.8:/volume1/shared-media
    owncloud	-rw,soft,intr,rsize=32768,wsize=32768 172.16.100.8:/volume1/owncloud
    EOF'

**Add /etc/auto.nfs to /etc/auto.master**

    sudo bash -c 'cat << EOF > /etc/auto.master
    +dir:/etc/auto.master.d
    /autofs                 /etc/auto.nfs      --timeout 30
    +auto.master
    EOF'

**Add links for permanent access**

    sudo mkdir /libraries
    cd /libraries
    sudo ln -s /autofs/shared-media .
    sudo ln -s /autofs/owncloud .

**Restart services**

    sudo systemctl restart autofs.service

**Test autofs**

    ls -al /libraries/shared-media/ /libraries/owncloud/

## Remove swap

*NOTE: If swap is not disabled, kubelet will refuse to start.*

    sudo swapoff -a
    sudo sed -i '/\/swap.img.*$/d' /etc/fstab

## Enable kernel modules and configure sysctl

    sudo modprobe overlay
    sudo modprobe br_netfilter

**Configure persistent loading of modules with the following command**

    sudo tee /etc/modules-load.d/k8s.conf <<EOF
    overlay
    br_netfilter
    EOF

**Add settings to sysctl**

    sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

**Reload sysctl.configs**

    sudo sysctl --system

## Install containerd

**Add Docker's official GPG key**

    sudo apt-get -y update
    sudo apt-get -y install apt-transport-https ca-certificates curl gpg net-tools
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

**Add the repository to Apt sources**

    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

**Install containerd**

    sudo apt-get -y install containerd.io

**Configure containerd**

    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml

Set overlayfs and SystemdCgroup for containerd settings

*sudo vi /etc/containerd/config.toml*
    
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true

*or, simpeler but less control*

    sudo sed -i.bak 's/^\([[:space:]]*\)SystemdCgroup = false/\1SystemdCgroup = true/' /etc/containerd/config.toml

**Restart containerd and enable auto start**

    sudo systemctl restart containerd
    sudo systemctl enable containerd

**Enable IP forwarding (temporarily and permanently)**

    echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
    sudo sh -c "echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf"
    sudo sysctl -p

**Check status of containerd and sysctl settings**

    systemctl status containerd
    sysctl net.bridge.bridge-nf-call-ip6tables
    sysctl net.bridge.bridge-nf-call-iptables
    sysctl net.ipv4.ip_forward

## Install Kubeadm, Kubectl and Kubelet

*These instructions are for Kubernetes v1.30.*

**Download the public signing key for the Kubernetes package**

    sudo mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

**Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version**

    sudo apt-get update
    sudo apt-get -y install kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

**Enable kubelet services**

    sudo systemctl enable --now kubelet

**Check that containerd socket exits**

    ls -al /var/run/containerd/containerd.sock

## Setup Kubernetes

*Reset kubernetes to start from scratch if your stuck and want to start over*

    # sudo kubeadm reset -f

**Init Kubenetes**

    sudo kubeadm init --pod-network-cidr=10.96.0.0/16 --cri-socket=/var/run/containerd/containerd.sock --v=5

**Add kubectl config to user profile**

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    export KUBECONFIG=/etc/kubernetes/admin.conf
    sudo chmod -R 755 /etc/kubernetes/admin.conf

**Enable commandline completion**

    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc
    source ~/.bashrc
    echo "complete -F __start_kubectl kubectl" >> ~/.bashrc
    source ~/.bashrc

**Test commandline completion**

    Start typing a kubectl command (e.g., kubectl get po), 
    pressing Tab should show suggestions for possible completions.

## Install Pod network add-on Calico

*Note: Only on Master Node!!!*

    wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
    wget https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

**Set correct CIDR in custom-resources.yaml (10.96.0.0/16)**

Edit *custom-resources.yaml*
    
    # This section includes base Calico installation configuration.
    # For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
    apiVersion: operator.tigera.io/v1
    kind: Installation
    metadata:
      name: default
    spec:
      # Configures Calico networking.
      calicoNetwork:
        ipPools:
        - name: default-ipv4-ippool
          blockSize: 26
          cidr: 10.96.0.0/16
          encapsulation: VXLANCrossSubnet
          natOutgoing: Enabled
          nodeSelector: all()
    
    ---
    
    # This section configures the Calico API server.
    # For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
    apiVersion: operator.tigera.io/v1
    kind: APIServer
    metadata:
      name: default
    spec: {}

**Apply tigera-operator.yaml and custom-resources.yaml**

    kubectl create -f tigera-operator.yaml
    kubectl create -f custom-resources.yaml

**Wait until that the pods are running**

    watch kubectl get pods -n calico-system

Your desired status should look like

    # ~$ watch kubectl get pods -n calico-system
    # 
    # Every 2.0s: kubectl get pods -n calico-system master: Wed May 22 15:55:46 2024
    #  
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # calico-kube-controllers-754f966575-9856h   1/1     Running   0          2m53s
    # calico-node-wnmz5                          1/1     Running   0          2m53s
    # calico-typha-774567f868-zkjxd              1/1     Running   0          2m53s
    # csi-node-driver-7vsfm                      2/2     Running   0          2m53s

## Sanity checks

**Check that the API server runs on port 6443**

    netstat -tan | grep 6443

**Check cluster status**

    kubectl cluster-info

**Check node status**

    kubectl get nodes -o wide

## Fix taint node

*Needed in a single host setup or if controller node is also worker*

    kubectl taint nodes --all node-role.kubernetes.io/control-plane-

## Enable Nvidia support (if applicable)

*used this intall guide: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html*

**Install drivers**

    sudo ubuntu-drivers autoinstall

*If you’re mixing GPU vendors, usually for power efficiency reasons, you
may need to set the Nvidia GPU as the default.*

    sudo prime-select nvidia

**Add Nvidia repositories**

    curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | sudo apt-key add -
    curl -s -L https://nvidia.github.io/nvidia-container-runtime/$(. /etc/os-release;echo $ID$VERSION_ID)/nvidia-container-runtime.list | sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list

**Update the cache**

    sudo apt update

**Install container runtime**

    sudo apt install nvidia-container-runtime -y

**Create backup of containerd config**

*Don't skip this step!*

    sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.backup

**Configure containerd**

    sudo sed -i 's/runtime = "runc"/runtime = "nvidia"/g' /etc/containerd/config.toml

**Check containerd config**

*Depending on your setup and/or current software versions the 
last step might have failed. Make sure these settings 
are present in /etc/containerd/config.toml*

    [plugins]
        [plugins."io.containerd.grpc.v1.cri".containerd]
          default_runtime_name = "nvidia"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
                BinaryName = "/usr/bin/nvidia-container-runtime"
                CriuImagePath = ""
                CriuPath = ""
                CriuWorkPath = ""
                IoGid = 0
                IoUid = 0
                NoNewKeyring = false
                NoPivotRoot = false
                Root = ""
                ShimCgroup = ""
                SystemdCgroup = true
      [plugins."io.containerd.runtime.v1.linux"]
        no_shim = false
        runtime = "nvidia-container-runtime"
        runtime_root = ""
        shim = "containerd-shim"
        shim_debug = false

**Reboot to activate all changes**

    sudo reboot

**Check that Kubernetes has started correctly**

    kubectl get pods --all-namespace

*You should see pods starting. If they don't start, restore your /etc/containterd/config.toml
and restart kubectl and containerd.*

    cp /etc/containerd/config.toml.backup /etc/containerd/config.toml
    sudo systemctl restart containerd
    sudo systemctl restart kubelet.service 
    
**Running a container with GPU support**

    sudo ctr images pull docker.io/nvidia/cuda:11.1.1-base
    sudo ctr run --rm --gpus 0 docker.io/nvidia/cuda:11.1.1-base nvidia-smi nvidia-smi

Your expected output should be something like
    
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 560.35.03              Driver Version: 560.35.03      CUDA Version: 12.6     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  Quadro K2200                   Off |   00000000:01:00.0 Off |                  N/A |
    | 42%   47C    P8              2W /   39W |       2MiB /   4096MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
                                                                                             
    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+

**Run Nvidia test pod**

    cat << EOF > nvidia-gpu-test-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nvidia-gpu-test-pod
    spec:
      restartPolicy: Never
      containers:
        - name: cuda-container
          image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0
          resources:
            limits:
              nvidia.com/gpu: 1 # requesting 1 GPU
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
    EOF

    kubectrl apply -f nvidia-gpu-test-pod.yaml

This pod should reach the state completed
    
    kubectl get pods -n default
    NAME      READY   STATUS      RESTARTS   AGE
    nvidia-gpu-test-pod   0/1     Completed   0          74m

*If the pod is stuck in the state pending, check pod info, Eg.*

    kubectl describe pod nvidia-gpu-test-pod -n default
    ...
    Events:
    Type     Reason            Age                From               Message
    ----     ------            ----               ----               -------
    Warning  FailedScheduling  76m                default-scheduler  0/1 nodes are available: 1 
    Insufficient nvidia.com/gpu. preemption: 0/1 nodes are available: 1 No preemption 
    victims found for incoming pod.

**Check the logs of this pod**

    kubectl logs -n default nvidia-gpu-test-pod
    [Vector addition of 50000 elements]
    Copy input data from the host memory to the CUDA device
    CUDA kernel launch with 196 blocks of 256 threads
    Copy output data from the CUDA device to the host memory
    Test PASSED
    Done

## Install helm

    sudo snap install helm --classic

## Install kubernetes-dashboard using helm

    helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
    helm search repo -l kubernetes-dashboard/kubernetes-dashboard | head -3
    helm show values kubernetes-dashboard/kubernetes-dashboard --version 7.10.0 > kubernetes-dashboard-override-helm-values.yaml

**Override some helm values**

*set your own hostname*

    $ vi kubernetes-dashboard-override-helm-values.yaml
    ...
    app:
      ingress:
        enabled: true
        host:
          kubernetes-dashboard.YOUR-DOMAIN.com
        ingressClassName: nginx
    ...

**Install the helm deployment**

    helm upgrade --install \
      kubernetes-dashboard \
      kubernetes-dashboard/kubernetes-dashboard \
      --namespace kubernetes-dashboard \
      --create-namespace \
      --debug \
      --values kubernetes-dashboard-override-helm-values.yaml \
      --version 7.10.0

**Delete the helm deployment**

    helm uninstall kubernetes-dashboard --namespace kubernetes-dashboard
    helm list --namespace kubernetes-dashboard

**Check the ingress rule**

    kubectl -n kubernetes-dashboard get ingress

*Optional: Proxy port-forwarding*

    kubectl -n kubernetes-dashboard port-forward \
      service/kubernetes-dashboard-kong-proxy 8443:443 --address 0.0.0.0

Open in a browser:

    https://172.16.100.9:8443

**Create Service-Account, ClusterRole-Binding and SecretToken**

    cat << EOF > kubernetes-dashboard-service-account-clusterrole-binding-secret-token.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kubernetes-dashboard-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: v1
    kind: Secret
    type: kubernetes.io/service-account-token
    metadata:
      name: admin-user-token
      namespace: kubernetes-dashboard
      annotations:
        kubernetes.io/service-account.name: "admin-user"
    EOF
    
    kubectl apply -f kubernetes-dashboard-service-account-clusterrole-binding-secret-token.yaml

**Create login-token generator script**

    cat << EOF > get-token.sh
    #!/bin/bash
    kubectl -n kubernetes-dashboard get secret admin-user-token -o yaml | grep token: | awk {'print \$2'} | base64 -d
    EOF
  
    chmod +x get-token.sh

**Acquire a login token**

    ./get-token.sh

**Test your kubernetes-dashboard**

    open https://172.16.100.9

## Add ingress-nginx

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm search repo -l ingress-nginx/ingress-nginx | head -3
    helm show values ingress-nginx/ingress-nginx --version 4.11.3 > ingress-nginx-override-helm-values.yaml

**Edit ingress-nginx-override-helm-values.yaml and change:**
    
    dnsPolicy: ClusterFirstWithHostNet
    hostNetwork: true
    externalIPs: ["172.16.100.9"]

**Deploy ingress-nginx**

    helm upgrade --install \
      ingress-nginx \
      ingress-nginx/ingress-nginx \
      --namespace ingress-nginx \
      --create-namespace \
      --debug \
      --values ingress-nginx-override-helm-values.yaml \
      --version 4.11.3

**Delete the ingress deployment**

    helm uninstall ingress-nginx --namespace ingress-nginx
    helm list --namespace ingress-nginx

After the Lets-Encrypt section we're going to install Oauth2 (via google) to access the
kubernetes-dashboard with Google login.

## Add Lets-Encrypt

**Get helm chart for cert manager**

    helm repo add jetstack https://charts.jetstack.io --force-update
    helm search repo -l jetstack/cert-manager | head -3


**Install helm chart**

    helm install \
      cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.16.2 \
      --set crds.enabled=true

**Verify that the pods are running**

    kubectl get pods --namespace cert-manager

**Create a ClusterIssuer or Issuer for Let's Encrypt**

    cat << EOF > letsencrypt-clusterissuer.yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: ACCOUNT@YOUR-DOMAIN.com
        privateKeySecretRef:
          name: letsencrypt-prod-key
        solvers:
          - http01:
              ingress:
                class: nginx
    EOF

    kubectl apply -f letsencrypt-clusterissuer.yaml

**Add tls to kubernetes-dashboard**
    
    kubectl edit ingress -n kubernetes-dashboard kubernetes-dashboard

    metadata:
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        meta.helm.sh/release-name: kubernetes-dashboard
        meta.helm.sh/release-namespace: kubernetes-dashboard
        nginx.ingress.kubernetes.io/backend-protocol: HTTPS
        nginx.ingress.kubernetes.io/ssl-redirect: 'true'
    spec:
      tls:
      - hosts:
          - kubernetes-dashboard.YOUR-DOMAIN.com
        secretName: kubernetes-dashboard-tls

*Check certificate issuing*

    kubectl describe certificaterequest kubernetes-dashboard-tls-1 -n kubernetes-dashboard

*Check the Challenge Resources*

    kubectl get challenges -n kubernetes-dashboard
    kubectl describe challenge <challenge-name> -n kubernetes-dashboard

*Check Cert-Manager Logs:*

    kubectl logs -l app=cert-manager -n cert-manager

*Monitor Events:*

    kubectl get events -n kubernetes-dashboard --sort-by='.metadata.creationTimestamp'

## Oauth2 Proxy

Make sure you have created a OAuth 2.0 Client IDs

    login at https://console.cloud.google.com/ 
    -> APIs & Services
    -> + Create Credentials
    -> Oauth Client ID

Fill in the form:

    application type: Web application
    name: kubernetes-dashboard
    Authorized JavaScript origins: https://kubernetes-dashboard.YOUR-DOMAIN.com
    Authorized redirect URIs: https://kubernetes-dashboard.YOUR-DOMAIN.com/oauth2/auth

Get the token for the admin-dashboard we'll need it in the next step

    kubectl get secret admin-user-token -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d

    example output: eyJhbGc...qHWXa7eUBg

Next we're going to edit the ingress for the kubernetes-dashboard

    kubectl edit -n kubernetes-dashboard ingress kubernetes-dashboard

And we're going to add the following annotations (use the token from 'admin-user-token')

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        meta.helm.sh/release-name: kubernetes-dashboard
        meta.helm.sh/release-namespace: kubernetes-dashboard
        nginx.ingress.kubernetes.io/auth-requeste-headers: x-auth-request-user,x-auth-request-email,authorization
        nginx.ingress.kubernetes.io/auth-signin: https://kubernetes-dashboard.YOUR-DOMAIN.com/oauth2/start?rd=$escaped_request_uri
        nginx.ingress.kubernetes.io/auth-url: https://kubernetes-dashboard.YOUR-DOMAIN.com/oauth2/auth
        nginx.ingress.kubernetes.io/backend-protocol: HTTPS
        nginx.ingress.kubernetes.io/configuration-snippet: |
          proxy_set_header Authorization "Bearer eyJhbGc...qHWXa7eUBg";
        nginx.ingress.kubernetes.io/ssl-redirect: "true"

These annotations will send a login token after the oauth2 redirect/login has been successful.
Now we're ready to deploy the oauth2 proxy. There is a configmap (for valid emails), deployment (oauth2 proxy), service and ingress.

Configure valid email accounts configmap:

    cat << EOF > oauth2-proxy-valid-email.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: oauth2-proxy-authenticated-emails
      namespace: kubernetes-dashboard 
    data:
      authenticated-emails.txt: |
        YOUR-ACCOUNT@gmail.com
        *@YOUR-DOMAIN.com
    EOF
    
    kubectl apply -f oauth2-proxy-valid-email.yaml

Configure Deployment:

    cat << EOF > oauth2-proxy-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: oauth2-proxy
      name: oauth2-proxy
      namespace: kubernetes-dashboard 
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: oauth2-proxy
      template:
        metadata:
          labels:
            app: oauth2-proxy
        spec:
          serviceAccountName: dashboard-user  # Add this line to use the 'dashboard-user' ServiceAccount
          containers:
            - args:
                - -provider=google
                - -upstream=file:///dev/null
                - -http-address=0.0.0.0:4180
                - -client-id=<GOOGLE client id>
                - -client-secret=<GOOGLE client secret>
                - -cookie-domain=kubernetes-dashboard.YOUR-DOMAIN.com
                - -email-domain=YOUR-DOMAIN.com
                - -authenticated-emails-file=/etc/oauth2-proxy/authenticated-emails.txt 
                - -whitelist-domain=YOUR-DOMAIN.com
                - -cookie-refresh=1h
                - -cookie-secret=Sa7b....5oTA
                - --pass-access-token=true
                - --set-authorization-header=true
                - --pass-user-headers=true
                - --pass-authorization-header=true # Forward the Authorization header
              env:
                - name: OAUTH2_PROXY_PASS_AUTHORIZATION_HEADER
                  value: "true"
                - name: OAUTH2_PROXY_SET_AUTHORIZATION_HEADER
                  value: "true"
              image: quay.io/pusher/oauth2_proxy:v4.1.0-amd64
              name: oauth2-proxy
              ports:
                - containerPort: 4180
                  protocol: TCP
              volumeMounts:
                - name: auth-email-file
                  mountPath: /etc/oauth2-proxy/authenticated-emails.txt
                  subPath: authenticated-emails.txt
          volumes:
            - name: auth-email-file
              configMap:
                name: oauth2-proxy-authenticated-emails
    EOF

    kubectl apply -f oauth2-proxy-deployment.yaml

Configure service and ingress:

    cat << EOF > oauth2-proxy-service-ingress.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: oauth2-proxy
      name: oauth2-proxy
      namespace: kubernetes-dashboard
    spec:
      ports:
        - name: http
          port: 4180
          protocol: TCP
          targetPort: 4180
      selector:
        app: oauth2-proxy
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: oauth2-proxy
      namespace: kubernetes-dashboard 
      annotations:
        kubernetes.io/ingress.class: "nginx"
    spec:
      rules:
        - host: kubernetes-dashboard.YOUR-DOMAIN.com
          http:
            paths:
              - path: /oauth2
                pathType: Prefix
                backend:
                  service:
                    name: oauth2-proxy
                    port:
                      number: 4180
      tls:
        - hosts:
            - kubernetes-dashboard.YOUR-DOMAIN.com
          secretName: tls-secret
    EOF

    kubectl apply -f oauth2-proxy-service-ingress.yaml

Test by opening an incognito browser and login to the dashboard

    https://kubernetes-dashboard.YOUR-DOMAIN.com

Verify that an invalid email address (with SUCCESSFUL google login) is denied.

## Create Docker Registry

**Create namespace**

    cat << EOF > docker-registry-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: docker-registry
    EOF

    kubectl apply -f docker-registry-namespace.yaml
    
**Create PersistentVolumeClaim**

    cat << EOF > docker-registry-persistent-volume-claim.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: docker-registry-pvc
      namespace: docker-registry
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 128Gi
    EOF

    kubectl apply -f docker-registry-persistent-volume-claim.yaml

**Create PersistentVolume**

    cat << EOF > docker-registry-persistent-volume.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: docker-registry-pv
    spec:
      capacity:
        storage: 128Gi
      accessModes:
        - ReadWriteOnce
      nfs:
        path: /volume1/shared-media/kubernetes/docker-registry
        server: 172.16.100.8
    EOF

    kubectl apply -f docker-registry-persistent-volume.yaml

**Create an htpasswd secret**

*install apache2-utils if it's not already installed*

    sudo apt-get install apache2-utils  # On Ubuntu/Debian
    
*Create the htpasswd file using bcrypt*

    htpasswd -cB ./htpasswd registry
    New password: ........
    Re-type new password: ........
    Adding password for user registry
    
**Add Helm charts**
    
    helm repo add phntom https://phntom.kix.co.il/charts/
    helm search repo -l phntom/docker-registry  | head -3
    helm show values phntom/docker-registry --version 1.10.0 > docker-registry-override-helm-values.yaml

**Configure overrides**

    vim docker-registry-override-helm-values.yaml
 
*Replace htpasswd with the content of prior created htpasswd file*

    ingress:
      enabled: true 
      path: /
      hosts:
        - registry.YOUR-DOMAIN.com
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/backend-protocol: HTTPS
        nginx.ingress.kubernetes.io/ssl-redirect: 'true'
        kubernetes.io/ingress.class: nginx
        kubernetes.io/tls-acme: "true"
      tls:
        - secretName: registry.YOUR-DOMAIN.com
          hosts:
            - registry.YOUR-DOMAIN.com
    persistence:
      accessMode: 'ReadWriteOnce'
      enabled: true
      existingClaim: docker-registry-pvc
    tlsSecretName: registry.YOUR-DOMAIN.com
    secrets:
      haSharedSecret: ""
      htpasswd: "registry:$2y$05$U....Ksq"

**Deploy helm chart**

    helm install docker-registry-integ phntom/docker-registry -f docker-registry-override-helm-values.yaml --namespace docker-registry 
