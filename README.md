# Testing kind


## direnv

Install

```shell
brew install kind kubectl
```

Create environment configuration files:

```shell
touch .env{,rc}
```

Allow environment configuration files:

```shell
direnv allow
```


## Install kind & dependencies

```shell
brew install kind kubectl
```

## Create cluster

```shell
kind create cluster \
    --name kindtest \
    --kubeconfig .kubeconfig \
    --wait 3m
```

Output:

```txt
Creating cluster "kindtest" ...
 ✓ Ensuring node image (kindest/node:v1.31.0)
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Waiting ≤ 3m0s for control-plane = Ready ⏳ 
 • Ready after 25s 💚
Set kubectl context to "kind-kindtest"
You can now use your cluster with:

kubectl cluster-info --context kind-kindtest --kubeconfig .kubeconfig
```

!!!tip
If you run `kind create cluster` with **`--retain`** it will prevent cleanup, and then you can run `kind export logs` to dump lots of cluster logs, then `kind delete cluster` to cleanup the cluster.
If you share the logs we can see more detail about why kubelet is unhealthy.
!!!

!!!note  [Pod errors due to “too many open files](https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files)
This issue may be caused by **ulimit**.
I have updated the ulimit for `max_user_watches` and `max_user_instances` to a higher value and now the kind cluster is coming up without any problem.

```shell
echo fs.inotify.max_user_watches=655360 | sudo tee -a /etc/sysctl.conf
echo fs.inotify.max_user_instances=1280 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
!!!

Verify `kubeconfig` has been create

```shell
cat .kubeconfig 
```

Output:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ****
    server: https://127.0.0.1:46493
  name: kind-kindtest
contexts:
- context:
    cluster: kind-kindtest
    user: kind-kindtest
  name: kind-kindtest
current-context: kind-kindtest
kind: Config
preferences: {}
users:
- name: kind-kindtest
  user:
    client-certificate-data: ****
    client-key-data: ****
```

Set `KUBECONFIG` environment variable

```shell
export KUBECONFIG=".kubeconfig"
```

Get clusters list

```shell
$ kind get clusters           
kindtest
```

Get all nodes list

```shell
kind get nodes -n kindtest
kindtest-control-plane
```

Get cluster informations

```shell
kubectl cluster-info
```

Output:

```txt
Kubernetes control plane is running at https://127.0.0.1:46493
CoreDNS is running at https://127.0.0.1:46493/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Flux

### Install Flux CLI

```
brew install fluxcd/tap/flux
```

### Configurer l'authentification GitHub:

```
cat < EOS >> .env
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
EOS
source .env
```

Token needs R/W access on Administration & Contents

Check K8s cluster:

```
flux check --pre
► checking prerequisites
✔ Kubernetes 1.31.0 >=1.28.0-0
✔ prerequisites checks passed
```

### Deploy Flux into the cluster:

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

Output:

```
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/silopolis/fleet-infra.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("bd995167e4d8e59956d3e3c45d8101d7eec09118")
► pushing component manifests to "https://github.com/silopolis/fleet-infra.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBATatoe/4Gnk2IfqtS4CmyKjMzuGWg6g+/LdyRt+4BwmWlq7GT77F7Q8WqFFcIGImamBvUz9ACQLRiW1isM9hP8/wStX4GBAX9RWIBOaJnE8JAWDCwOuD2SnMK/3+oeabA==
✔ configured deploy key "flux-system-main-flux-system-./clusters/my-cluster" for "https://github.com/silopolis/fleet-infra"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("6d968a9a0a31f4e4d4015435a88df8d0c59d30fd")
► pushing sync manifests to "https://github.com/silopolis/fleet-infra.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

The bootstrap command above does the following:

* *Creates a Git repository* `fleet-infra` on your GitHub account.
* *Adds Flux component manifests* to the repository.
* *Deploys Flux Components* to your Kubernetes Cluster.
* *Configures Flux components* to track the path /clusters/my-cluster/ in the repository.

### Clone Git repo

```
git clone https://github.com/$GITHUB_USER/fleet-infra ../fleet-infra
cd ../fleet-infra
```

### Add podinfo repository to Flux

```
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=1m \
  --export > ../fleet-infra/clusters/my-cluster/podinfo-source.yaml
```

Commit and push `podinfo-source.yaml` to `fleet-infra` repository:

```
git add clusters/my-cluster/podinfo-source.yaml
git commit -m "Add podinfo Flux GitRepository"
git push
```

### Deploy podinfo application

Configure Flux to build and apply the kustomize directory located in the podinfo repository.

Use the flux create command to create a Kustomization that applies the podinfo deployment.

```
flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml
```

Commit and push the Kustomization manifest to the repository:

```
git add -A && git commit -m "Add podinfo Kustomization"
git push
```

Use the flux get command to watch the podinfo app:

```
flux get kustomizations --watch
NAME       	REVISION          	SUSPENDED	READY	MESSAGE                              
flux-system	main@sha1:7b750d67	False    	True 	Applied revision: main@sha1:7b750d67	
flux-system	main@sha1:7b750d67	False	Unknown	Reconciliation in progress	
flux-system	main@sha1:7b750d67	False	Unknown	Reconciliation in progress	
flux-system	main@sha1:7b750d67	False	Unknown	Reconciliation in progress	
flux-system	main@sha1:7b750d67	False	Unknown	Reconciliation in progress	
podinfo		False	False	waiting to be reconciled	
podinfo		False	False	waiting to be reconciled	
flux-system	main@sha1:7b750d67	False	True	Applied revision: main@sha1:8e8b74df	
flux-system	main@sha1:8e8b74df	False	True	Applied revision: main@sha1:8e8b74df	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	Unknown	Reconciliation in progress	
podinfo		False	True	Applied revision: master@sha1:08238ead	
podinfo	master@sha1:08238ead	False	True	Applied revision: master@sha1:08238ead
```

```
flux get kustomizations        
NAME       	REVISION            	SUSPENDED	READY	MESSAGE                                
flux-system	main@sha1:8e8b74df  	False    	True 	Applied revision: main@sha1:8e8b74df  	
podinfo    	master@sha1:08238ead	False    	True 	Applied revision: master@sha1:08238ead
```

Check podinfo has been deployed on the cluster:

```
kubectl get svc,deploy,pod,hpa -n default -o wide
```

```
NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE     SELECTOR
service/bar-service          ClusterIP      10.96.226.31    <none>        8080/TCP            152m    app=bar
service/foo-bar-lb-service   LoadBalancer   10.96.34.238    172.19.0.5    5678:31018/TCP      80m     app=http-echo
service/foo-service          ClusterIP      10.96.122.95    <none>        8080/TCP            152m    app=foo
service/kubernetes           ClusterIP      10.96.0.1       <none>        443/TCP             176m    <none>
service/podinfo              ClusterIP      10.96.252.163   <none>        9898/TCP,9999/TCP   4m42s   app=podinfo

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                               SELECTOR
deployment.apps/podinfo   2/2     2            2           4m42s   podinfod     ghcr.io/stefanprodan/podinfo:6.7.0   app=podinfo

NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
pod/bar-app                    1/1     Running   0          152m    10.244.1.3   kind-ingress-worker    <none>           <none>
pod/bar-lb                     1/1     Running   0          80m     10.244.2.4   kind-ingress-worker2   <none>           <none>
pod/foo-app                    1/1     Running   0          152m    10.244.2.3   kind-ingress-worker2   <none>           <none>
pod/foo-lb                     1/1     Running   0          80m     10.244.1.4   kind-ingress-worker    <none>           <none>
pod/podinfo-7599545d7f-47xx8   1/1     Running   0          4m27s   10.244.1.7   kind-ingress-worker    <none>           <none>
pod/podinfo-7599545d7f-lnmdz   1/1     Running   0          4m42s   10.244.2.7   kind-ingress-worker2   <none>           <none>

NAME                                          REFERENCE            TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/podinfo   Deployment/podinfo   cpu: <unknown>/99%   2         4         2          4m42s
```

Customize podinfo deployment

To customize a deployment from a *repository you don’t control*, you can use Flux **in-line patches**.

```
cat << EOP >> podinfo-kustomization.yaml
  patches:
    - patch: |-
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: podinfo
        spec:
          minReplicas: 3             
      target:
        name: podinfo
        kind: HorizontalPodAutoscaler
EOP
```

## Capacitor

Create Capacitor Kustomization 

```
# ./clusters/my-cluster/capacitor-kustomization.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: capacitor
  namespace: flux-system
spec:
  interval: 12h
  url: oci://ghcr.io/gimlet-io/capacitor-manifests
  ref:
    semver: ">=0.1.0"
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: capacitor
  namespace: flux-system
spec:
  targetNamespace: flux-system
  interval: 1h
  retryInterval: 2m
  timeout: 5m
  wait: true
  prune: true
  path: "./"
  sourceRef:
    kind: OCIRepository
    name: capacitor
```

Commit and push Capacitor Kustomization

```
git add -A
git commit -m "Add Capacitor UI Kustomization"
```

Wait for Capacitor to be deployed

```
flux get kustomizations -n flux-system --watch       
NAME       	REVISION          	SUSPENDED	READY	MESSAGE                              
flux-system	main@sha1:957ee967	False    	True 	Applied revision: main@sha1:957ee967	
podinfo	master@sha1:08238ead	False	True	Applied revision: master@sha1:08238ead	
capacitor		False	Unknown	Reconciliation in progress	
...
capacitor		False	True	Applied revision: v0.4.5@sha256:0f30e8b0	
capacitor	v0.4.5@sha256:0f30e8b0	False	True	Applied revision: v0.4.5@sha256:0f30e8b0	
capacitor	v0.4.5@sha256:0f30e8b0	False	Unknown	Reconciliation in progress	
...
capacitor	v0.4.5@sha256:0f30e8b0	False	True	Applied revision: v0.4.5@sha256:0f30e8b0
```

Check deployed resources

```
kubectl get svc,deploy,pod,hpa -n flux-system -o wide      
NAME                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/capacitor                 ClusterIP   10.96.238.57    <none>        9000/TCP   6m26s   app.kubernetes.io/instance=capacitor,app.kubernetes.io/name=onechart
service/notification-controller   ClusterIP   10.96.15.164    <none>        80/TCP     6h50m   app=notification-controller
service/source-controller         ClusterIP   10.96.142.83    <none>        80/TCP     6h50m   app=source-controller
service/webhook-receiver          ClusterIP   10.96.122.137   <none>        80/TCP     6h50m   app=notification-controller

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                                          SELECTOR
deployment.apps/capacitor                 1/1     1            1           6m26s   capacitor    ghcr.io/gimlet-io/capacitor:v0.4.5              app.kubernetes.io/instance=capacitor,app.kubernetes.io/name=onechart
deployment.apps/helm-controller           1/1     1            1           6h50m   manager      ghcr.io/fluxcd/helm-controller:v1.1.0           app=helm-controller
deployment.apps/kustomize-controller      1/1     1            1           6h50m   manager      ghcr.io/fluxcd/kustomize-controller:v1.4.0      app=kustomize-controller
deployment.apps/notification-controller   1/1     1            1           6h50m   manager      ghcr.io/fluxcd/notification-controller:v1.4.0   app=notification-controller
deployment.apps/source-controller         1/1     1            1           6h50m   manager      ghcr.io/fluxcd/source-controller:v1.4.1         app=source-controller

NAME                                           READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
pod/capacitor-6ddcbf67ff-r769k                 1/1     Running   0          6m26s   10.244.2.8   kind-ingress-worker2   <none>           <none>
pod/helm-controller-7f788c795c-gmtxg           1/1     Running   0          6h50m   10.244.1.6   kind-ingress-worker    <none>           <none>
pod/kustomize-controller-b4f45fff6-ckvr9       1/1     Running   0          6h50m   10.244.1.5   kind-ingress-worker    <none>           <none>
pod/notification-controller-556b8867f8-64fph   1/1     Running   0          6h50m   10.244.2.5   kind-ingress-worker2   <none>           <none>
pod/source-controller-77d6cd56c9-rfn68         1/1     Running   0          6h50m   10.244.2.6   kind-ingress-worker2   <none>           <none>
```

Use port-forwarding to access Capacitor UI

```
kubectl -n flux-system port-forward svc/capacitor 9000:9000
```

Access Capacitor UI

```
xdg-open http://localhost:9000
```

Create an Ingress for Capacitor

```
```

Commit and push

```
```


## Weave GitOps


Install `gitops` CLI

```


