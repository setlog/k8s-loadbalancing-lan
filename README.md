## Preface

Kubernetes has already become a common technology for everyone who has a challenge to ship and run the software in the cloud. And every cloud provider helps to setup the infrastructure as easy as possible for us, devop engineers. We just start a kubernetes service, wait a couple of seconds and Voila! It has an external IP that we can assign a DNS name to and access the endpoint as if no magic happened. Have you tried the same in your LAN? If no, give it a try. Just to understand that not everything what a cloud provider does for you can be taken for granted. Let us demonstrate with a simple example, what challenges you are facing and how you can master them.

We are not going to explain here, how to set up a kubernetes cluster in your LAN, but how to deploy a simple website in a ready-made cluster and access it by a DNS name in your intranet.

## Software Prerequisites

You need an unmanaged kubernetes cluster running in your intranet on several machines. We tested this guide with the k8s-version 1.17.2 on one master and two workers.

## Let us fail first

We deploy a simple nginx deployment with two server replicas and a load balancer.

![Nginx](images/nginx-server.png "nginx-server")

nginx-server.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web

---

apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: web
spec:
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: website
        image: nginx
        ports:
          - containerPort: 80
```

Now we are going to create this deployment by executing `kubectl`.

```sh
kubectl create -f nginx-server.yaml
```

On a managed cluster you would expect the load balancer to obtain an external IP address from the cloud provider. This will not happen in an unmanaged cluster in your LAN. It remains `pending`.

```sh
NAME    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx   LoadBalancer   10.96.34.70   <pending>     80:32593/TCP   5m15s
```

What now? You can still access the service by forwarding the port to the localhost:

```sh
  kubectl port-forward -n web service/nginx 8080:80
```

You can call the website in the browser under `http://localhost:8080` now. But only you, not your colleagues in the office. 

This approach is not what we want. We need a stable IP address in LAN.

### Be like a cloud provider

What we need is a DHCP server and the installed MetalLB infrastructure (https://metallb.universe.tf). 

Since MetalLB runs in a dedicated namespace, we create namespace `metallb-system`:

```sh
kubectl create namespace metallb-system
```

...and deploy MetalLB in our cluster:

```sh
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
```sh
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.9.3/manifests/metallb.yaml
```

Let's take a look at the pods and make sure they are running:

- `kubectl get pod -n metallb-system`
```sh
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-5696cd5dcb-9hjss   1/1     Running   0          75s
pod/speaker-56rsl                 1/1     Running   0          75s
pod/speaker-5ctv5                 1/1     Running   0          75s
```

If so, we are ready to configure our cluster for allocating static ip addresses by load balancers.

#### Configure MetalLB

MetalLB supports two configurations of announcing service IP's: Layer2 and BGP. We used Layer2 Configuration because it is easy and works well in most network landscapes (see more information under https://metallb.universe.tf/configuration).

Let's assume you want to access your website under 192.168.1.100. In that case create following ConfigMap and put this IP in `addresses` as a range:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.100-192.168.1.100
```

Of course, you can put a real range of ip addresses here, but we want to keep it simple and unambiguous now.

After deploying this YAML with `kubectl create -f metallb-config.yaml`, the load balance is able to obtain a real IP address from the specified range:

```
NAME                         TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
service/nginx                LoadBalancer   10.96.34.70   192.168.1.100   80:32593/TCP   57m
```

Now you should be able to access your website via webbrowser `http://192.168.1.100`

![Website](images/welcome-nginx.png "welcome-nginx")

### DNS record

Since the website has a static IP address, it is possible to create an A-record on the DNS server, let say `nginx.your-domain.lan -> 192.168.1.100`, and access the website in web browser under `http://nginx.your-domain.lan`.

That is what you wanted, didn't you?

### Cleaning up

If you are ready and don't want to use the nginx server and MetalLB, you can remove the deployments with `kubectl` again:

```sh
kubectl delete namespace metallb-system
kubectl delete namespace web
```
