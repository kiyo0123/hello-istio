# hello-istio
こちらを参考に。
https://github.com/istio/istio/tree/master/samples/helloworld

## Sidecar Injection
(Automatic sidecar injection)[https://istio.io/docs/setup/kubernetes/sidecar-injection/#automatic-sidecar-injection] を有効にしていない場合は、istioctl コマンドを使って、sidecar が挿入されたyamlファイルを生成する。

```
istioctl kube-inject -f helloworld.yaml -o hello-world-istion.yaml
```

## Deploy Application

```
kubectl create -f helloworld-istio.yaml
```

```
$ kubectl create -f helloworld-istio.yaml
service "helloworld" created
deployment.extensions "helloworld-v1" created
deployment.extensions "helloworld-v2" created
gateway.networking.istio.io "helloworld-gateway" created
virtualservice.networking.istio.io "helloworld" created
```

virtualservices がデプロイされていることを確認。
```
$ istioctl get virtualservices helloworld
VIRTUAL-SERVICE NAME   GATEWAYS             HOSTS     #HTTP     #TCP      NAMESPACE   AGE
helloworld             helloworld-gateway   *             1        0      default     1m
```


## Verify Application running

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

バージョンに関係なく、pod全体に対して負荷をうまく分散している。
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
kubectl get hpa
```

## Generate load
TODO
普通にMacから 
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




