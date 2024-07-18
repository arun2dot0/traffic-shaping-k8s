### Traffic shapping in k8s using Istio ###

You may want to traffic shaping due to various reasons
- Resource allocation:  
Developers can simulate various network conditions and bandwidth limitations, helping them test how applications perform under different scenarios.
- Performance testing:  
By shaping traffic, developers can assess how their applications behave under constrained network conditions, identifying potential bottlenecks or performance issues early in the development process.
- Realistic environment simulation:  
 Traffic shaping helps create a more realistic representation of production-like conditions, allowing developers to catch and address potential issues before they reach later stages.

Here are how you can do this with K8s and Istio . I am doing this in mac
and have installed Docker desktop with k8s integration . Example may be differ
slightly with your setup . 

_Note : I am not pulling the image and building it locally , this will be ideal for development
testing as you don't want to be pulling from registry . However change the image pull behavior
for production setups_ 
![Screenshot 2024-07-13 at 4 36 49 PM](https://github.com/user-attachments/assets/ca0486bb-8fb5-4923-b738-8eddd797fce8)


### Steps ###


- Step 1 - Install Istio , prometheus and kiali

This example uses the Istio service mesh implementation.
There are also other service mesh implementations that can do the same

```
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/addons/prometheus.yaml

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/addons/kiali.yaml

```

- Step 2  - Build the docker images

I am using the nodejs-multistage-example and doing build of two docker images with different names . This can be two different images with different versions of
your application
```
docker build -t  my-service-1 .   
docker build -t  my-service-2 .   

```

- Step 3 - Apply the deployment

If you have used another name fpr the image update the deployment file accordingly

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f traffic-shaping.yaml

DEBUG
kubectl get deployments
kubectl get pods
kubectl get services

```

- Step 4 - Run the traffic stimulator

```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://myapp-service; done"
```


- Step 4 - Open the Istio Kiali console in different tab

```
istioctl dashboard kiali
```

- Step 5 - Validate

You can validate the traffic flow as per your specifications
Also other metrics and errors with your deployment are available with prometheus


### Logs for reference ###

```
username@my-pc traffic-shaping-k8s % kubectl apply -f deployment.yaml
deployment.apps/myapp-v1 created
deployment.apps/myapp-v2 created
username@my-pc traffic-shaping-k8s % kubectl get pods
NAME                        READY   STATUS            RESTARTS   AGE
myapp-v1-54f7bd99dd-gqhcj   0/2     PodInitializing   0          9s
myapp-v2-b457864bb-pbnr5    0/2     PodInitializing   0          9s
username@my-pc traffic-shaping-k8s % kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
myapp-v1-54f7bd99dd-gqhcj   2/2     Running   0          13s
myapp-v2-b457864bb-pbnr5    2/2     Running   0          13s
username@my-pc traffic-shaping-k8s % kubectl apply -f service.yaml
service/myapp-service created
username@my-pc traffic-shaping-k8s % kubectl apply -f traffic-shaping.yaml
virtualservice.networking.istio.io/myapp-virtualservice unchanged
destinationrule.networking.istio.io/myapp-destinationrule unchanged
username@my-pc traffic-shaping-k8s % kubectl get services
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP   21h
myapp-service   ClusterIP   10.96.141.170   <none>        80/TCP    9s
username@my-pc traffic-shaping-k8s % kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://myapp-service; done"

```
