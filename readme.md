# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.


```bash
# первая версия
cat deployment.yaml
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   name: nginx
#   labels:
#     app: nginx-multitool
# spec:
#   replicas: 1
#   selector:
#     matchLabels:
#       app: nginx-multitool
#   template:
#     metadata:
#       labels:
#         app: nginx-multitool
#     spec:
#       containers:
#       - name: nginx
#         image: nginx:1.14.0
#       - name: multitool
#         image: wbitt/network-multitool:latest

kubectl apply -f deployment.yaml

kubectl get po
NAME                     READY   STATUS             RESTARTS        AGE
nginx-54fbf7f458-j4k72   1/2     CrashLoopBackOff   5 (2m15s ago)   6m5s

# пробую разобраться с причиной
kubectl describe deploy nginx 
Name:                   nginx
Namespace:              default
CreationTimestamp:      Mon, 31 Aug 2026 18:07:39 +0300
Labels:                 app=nginx-multitool
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx-multitool
Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx-multitool
  Containers:
   nginx:
    Image:        nginx:1.14.0
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
   multitool:
    Image:         wbitt/network-multitool:latest
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      False   MinimumReplicasUnavailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-54fbf7f458 (1/1 replicas created)
Events:          <none>

kubectl logs nginx-54fbf7f458-j4k72 -c nginx
kubectl logs nginx-54fbf7f458-j4k72 -c multitool
The directory /usr/share/nginx/html is not mounted.
Therefore, over-writing the default index.html filewith some useful information:
WBITT Network MultiTool (with NGINX) - nginx-54fbf7f458-j4k72 - 10.1.128.214 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
2026/08/31 15:19:19 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
2026/08/31 15:19:19 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
2026/08/31 15:19:19 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
2026/08/31 15:19:19 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
2026/08/31 15:19:19 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
2026/08/31 15:19:19 [emerg] 1#1: still could not bind()
nginx: [emerg] still could not bind()
```
Видно что занят под. Пробуем у мультитула поставить другой
```bash
# kubectl get po -w
kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-79d57f9b6b-xtd9t   2/2     Running   0          51s
```
А теперь скаллируем
```bash
kubectl scale deploy nginx --replicas=2
kubectl get po   
NAME                     READY   STATUS    RESTARTS  AGE
nginx-79d57f9b6b-sphrl   2/2     Running   0  14s
nginx-79d57f9b6b-xtd9t   2/2     Running   0  2m16s
```

Сервис, который обеспечит доступ до приложений, описан [тут](./service.yaml)
```bash
kubectl apply -f service.yaml   
# service/nginx-multitool-service created

kubectl get service
# NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
# kubernetes                ClusterIP   10.152.183.1     <none>        443/TCP                       13h
# nginx-multitool-service   NodePort    10.152.183.239   <none>        80:30910/TCP,8080:32601/TCP   2m42s
```

Теперь содадим [отдельный Pod](./pod.yaml) с приложением multitool и убедимся с помощью curl, что из пода есть доступ до приложений из п.1
```bash
kubectl apply -f pod.yaml 

kubectl exec -it pod-with-multitool -- bash
pod-with-multitool:/# curl nginx-multitool-service
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# <style>
#     body {
#         width: 35em;
#         margin: 0 auto;
#         font-family: Tahoma, Verdana, Arial, sans-serif;
#     }
# </style>
# </head>
# <body>
# <h1>Welcome to nginx!</h1>
# <p>If you see this page, the nginx web server is successfully installed and
# working. Further configuration is required.</p>

# <p>For online documentation and support please refer to
# <a href="http://nginx.org/">nginx.org</a>.<br/>
# Commercial support is available at
# <a href="http://nginx.com/">nginx.com</a>.</p>

# <p><em>Thank you for using nginx.</em></p>
# </body>
# </html>
```

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

```bash
kubectl create namespace task02
# namespace/task02 created
kubectl apply -f task02/deployment.yaml -n task02
# deployment.apps/nginx created
kubectl get po -n task02
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-8547d867cd-gqtl8   1/1     Running   0          32s

kubectl describe po nginx-8547d867cd-gqtl8 -n task02
Name:             nginx-8547d867cd-gqtl8
Namespace:        task02
Priority:         0
Service Account:  default
Node:             microk8s/10.0.1.20
Start Time:       Tue, 01 Sep 2026 06:40:11 +0300
Labels:           app=task02
                  pod-template-hash=8547d867cd
Annotations:      cni.projectcalico.org/containerID: a4ab94fcdb5bdf73f6d457f5a80b7430912190f70ed1613f04d2624dcfefd804
                  cni.projectcalico.org/podIP: 10.1.128.221/32
                  cni.projectcalico.org/podIPs: 10.1.128.221/32
Status:           Running
IP:               10.1.128.221
IPs:
  IP:           10.1.128.221
Controlled By:  ReplicaSet/nginx-8547d867cd
Init Containers:
  delay:
    Container ID:  containerd://d2cabc22bb791cbab4904be4bf25c5484e32e82e8122f84f2784389adad8c792
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:dc2d74b28e4cf8984fa52af1f39bc7c3d9c73760b41a74d629f5d11b1ab28616
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      15
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 01 Sep 2026 06:40:15 +0300
      Finished:     Tue, 01 Sep 2026 06:40:30 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vkmq8 (ro)
Containers:
  nginx:
    Container ID:   containerd://d681704a4db589405f26f22d4119cdd24212f3946bf160af55793bd3912b43cf
    Image:          nginx:1.14.0
    Image ID:       docker.io/library/nginx@sha256:8b600a4d029481cc5b459f1380b30ff6cb98e27544fc02370de836e397e34030
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 01 Sep 2026 06:40:31 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vkmq8 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-vkmq8:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  89s   default-scheduler  Successfully assigned task02/nginx-8547d867cd-gqtl8 to microk8s
  Normal  Pulling    89s   kubelet            spec.initContainers{delay}: Pulling image "busybox"
  Normal  Pulled     86s   kubelet            spec.initContainers{delay}: Successfully pulled image "busybox" in 2.882s (2.882s including waiting).Image size: 2236931 bytes.
  Normal  Created    86s   kubelet            spec.initContainers{delay}: Container created
  Normal  Started    86s   kubelet            spec.initContainers{delay}: Container started
  Normal  Pulled     70s   kubelet            spec.containers{nginx}: Container image "nginx:1.14.0" already present on machine and can be accessedby the pod
  Normal  Created    70s   kubelet            spec.containers{nginx}: Container created
  Normal  Started    70s   kubelet            spec.containers{nginx}: Container started
```
------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------