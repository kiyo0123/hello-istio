# Istio hello world
Istioのサンプルこちらを参考に。
https://github.com/istio/istio/tree/master/samples/helloworld

## Setup GKE cluster
Make sure that you have the GKE API enabled. 
```
gcloud services enable container.googleapis.com
```
```
gcloud config set compute/zone asia-northeast1-a
```


Create a new cluster by using following command. 
```
gcloud container clusters create hello-istio \
    --cluster-version=latest \
    --num-nodes 4 
```

grant admin permissions in the cluster to the current user. 
```
kubectl create clusterrolebinding cluster-admin-binding \
   -clusterrole=cluster-admin \
   -user=(gcloud config get-value core/account)
```

## Install Istio
``` 
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.2 sh -
```

```
cd ./istio-*
export PATH=$PWD/bin:$PATH
```

```
kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```

Make sure istio component successfully installed.
```
kubectl get svc -n istio-system
```
```
kubectl get pods -n istio-system
```

## Sidecar Injection
[Automatic sidecar injection](https://istio.io/docs/setup/kubernetes/sidecar-injection/#automatic-sidecar-injection) を有効にしていない場合は、istioctl コマンドを使って、sidecar が挿入されたyamlファイルを生成する。元のファイルは純Kubernetes。

```
istioctl kube-inject -f helloworld.yaml -o helloworld-istio.yaml
```

自動インジェクションを有効にするには以下のようにする。
```
kubectl label namespace default istio-injection=enabled
```


## Deploy Application
Istioのsidecarが挿入されたyamlファイルをデプロイする。
```
kubectl create -f helloworld-istio.yaml
```

成功すると以下のようなメッセージが出力されて、KubernetesのService、Deploymentsオブジェクトが生成される。
```
service "helloworld" created
deployment.extensions "helloworld-v1" created
deployment.extensions "helloworld-v2" created
```

続けてIstioのオブジェクトをデプロイする。
 ```
 kubectl create -f networking/helloworld-gateway.yaml
 ```

成功すると以下のようなメッセージが出力されて、Gateway、VirtualServiceオブジェクトが作成されたことが分かる。
```
gateway.networking.istio.io "helloworld-gateway" created
virtualservice.networking.istio.io "helloworld" created
```

istioclt コマンドを使って、virtualservices がデプロイされていることを確認。
```
$ istioctl get virtualservices helloworld
VIRTUAL-SERVICE NAME   GATEWAYS             HOSTS     #HTTP     #TCP      NAMESPACE   AGE
helloworld             helloworld-gateway   *             1        0      default     1m
```


## Verify Application running

外部からのアクセスエンドポイントはistio-ingressgateway。
```
kubectl get svc istio-ingressgateway -n istio-system
```

```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

```
curl http://$GATEWAY_URL/hello
```

```
$ while true; do sleep 2; curl http://$GATEWAY_URL/hello; done
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
...
```
v1とv2が交互に呼び出されているのは、
それぞれpodが1つずつ実行されているのだが、例えばこれをv2を3つにしてみるとどうなるのか？

```
kubectl scale deployments helloworld-v2 --replicas=3
kubectl get deployments
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
helloworld-v1    1         1         1            1           18m
helloworld-v2    3         3         3            2           18m
```

バージョンに関係なく、pod全体に対して負荷をうまく分散していることが分かる。
```
io$ while true; do sleep 2; curl http://$GATEWAY_URL/hello; done
Hello version: v2, instance: helloworld-v2-f46787f6c-rtjx4
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v2, instance: helloworld-v2-f46787f6c-ppf8l
Hello version: v2, instance: helloworld-v2-f46787f6c-rtjx4
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v2, instance: helloworld-v2-f46787f6c-ppf8l
Hello version: v2, instance: helloworld-v2-f46787f6c-rtjx4
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
Hello version: v2, instance: helloworld-v2-f46787f6c-ppf8l
Hello version: v2, instance: helloworld-v2-f46787f6c-rtjx4
Hello version: v1, instance: helloworld-v1-797889fb89-sbbxr
Hello version: v2, instance: helloworld-v2-f46787f6c-7zzlb
...
```


## Autoscale the services
```
kubectl autoscale deployment helloworld-v1 --cpu-percent=50 --min=1 --max=10
kubectl autoscale deployment helloworld-v2 --cpu-percent=50 --min=1 --max=10
```

HPA(Horizontal Pod Autoscaler)の情報を確認する。
```
$ kubectl get hpa
NAME            REFERENCE                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
helloworld-v1   Deployment/helloworld-v1   43%/50%         1         10        1          1d
helloworld-v2   Deployment/helloworld-v2   36%/50%         1         10        1          1d
istio-pilot     Deployment/istio-pilot     <unknown>/80%   1         5         0          32d
```

## Generate load
TODO
普通にMacから 

``` 
while true; do curl http://$GATEWAY_URL/hello; done

```



```
while true; do wget -q -O- http://$GATEWAY_URL/hello; done
```

```
$ kubectl get hpa --watch
NAME            REFERENCE                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
helloworld-v1   Deployment/helloworld-v1   1%/50%          1         10        1          32m
helloworld-v2   Deployment/helloworld-v2   2%/50%          1         10        1          31m
istio-pilot     Deployment/istio-pilot     <unknown>/80%   1         5         0          31d
helloworld-v1   Deployment/helloworld-v1   109%/50%   1         10        1         32m
helloworld-v2   Deployment/helloworld-v2   118%/50%   1         10        1         32m
helloworld-v1   Deployment/helloworld-v1   109%/50%   1         10        3         32m
helloworld-v2   Deployment/helloworld-v2   118%/50%   1         10        3         32m
helloworld-v1   Deployment/helloworld-v1   174%/50%   1         10        3         33m
helloworld-v2   Deployment/helloworld-v2   180%/50%   1         10        3         33m
```



```
./loadgen.sh &
./loadgen.sh & # run it twice to generate lots of load
```

上記だと全然負荷がかからないのでこれの代わりに
```
ab -n 1000 -c 100 http://$GATEWAY_URL/hello
```
これでもだめなので、別の方法

```
kubectl run -i --tty load-generator --image=busybox /bin/sh 
> while true; do wget -q -O- http://$GATEWAY_URL/hello; done
```


二分ほど待つと
```
kubectl get deployments
```




