### Домашнее задание к занятию «Запуск приложений в K8S» Баранов Сергей

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


[deployment-01.yaml]()

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-01
  labels:
	app: nginx
spec:
  replicas: 1
  selector:
	matchLabels:
  	app: nginx_multitool
  template:
	metadata:
  	labels:
    	app: nginx_multitool
	spec:
  	containers:
  	- name: nginx
    	image: nginx:1.20
    	ports:
    	- containerPort: 80
  	- name: multitool
    	image: wbitt/network-multitool
    	env:
    	- name: HTTP_PORT
      	value: "1180"
    	- name: HTTPS_PORT
      	value: "11443"
    	ports:
    	- containerPort: 1180
      	name: mt-http
    	- containerPort: 11443
      	name: mt-https
```
```
root@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-01.yaml
deployment.apps/deployment-01 configured
root@baranov:/home/baranovsa/kube-1.3#
oot@baranov:/home/baranovsa# kubectl get pods
NAME                           	READY   STATUS   RESTARTS   AGE
deployment-01-5c9cdc8d6d-2dn6b  2/2     Running  0          70s
root@baranov:/home/baranovsa#
```

2. После запуска увеличить количество реплик работающего приложения до 2.

```
root@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-01.yaml
deployment.apps/deployment-01 configured
root@baranov:/home/baranovsa/kube-1.3#
```

3. Продемонстрировать количество подов до и после масштабирования.

```
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                             	READY   STATUS    RESTARTS      AGE
deployment-01-5c9cdc8d6d-2dn6b          2/2     Running   0      	11m
deployment-01-5c9cdc8d6d-6xlxh          2/2     Running   0      	13s
root@baranov:/home/baranovsa/kube-1.3#
```

4. Создать Service, который обеспечит доступ до реплик приложений из п.1.

```
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  ports:
    - name: nginx
      port: 80
      protocol: TCP
      targetPort: 80
    - name: miltitool
      port: 1180
      protocol: TCP
      targetPort: 1180
  selector:
    app: nginx_multitool
```

```
oot@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-01.yaml
deployment.apps/deployment-01 unchanged
service/web-svc created
root@baranov:/home/baranovsa/kube-1.3#
```

```
root@baranov:/home/baranovsa# kubectl get svc
NAME        TYPE    	CLUSTER-IP     EXTERNAL-IP    PORT(S)           AGE
kubernetes  ClusterIP   10.96.0.1      <none>         443/TCP           3d23h
web-svc     ClusterIP   10.109.205.73  <none>         80/TCP,1180/TCP   4m47s
root@baranov:/home/baranovsa#
```

5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

```
root@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-01.yaml
deployment.apps/deployment-01 unchanged
service/web-svc unchanged
pod/pod-multitool created
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                             READY  STATUS   RESTARTS   AGE
deployment-01-5c9cdc8d6d-2dn6b   2/2 	Running     0       35m
deployment-01-5c9cdc8d6d-6xlxh   2/2 	Running     0       24m
pod-multitool                	 1/1 	Running     0       19s
root@baranov:/home/baranovsa/kube-1.3#
```
```
root@baranov:/home/baranovsa/kube-1.3# kubectl exec --stdin --tty pod-multitool -- /bin/bash
pod-multitool:/# curl 10.109.205.73:1180
WBITT Network MultiTool (with NGINX) - deployment-01-5c9cdc8d6d-2dn6b - 10.244.0.4 - HTTP: 1180 , HTTPS: 11443 . (Formerly praqma/network-multitool)
pod-multitool:/#
root@baranov:/home/baranovsa/kube-1.3#
```
------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.


[deployment-02.yaml]()

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-02
  labels:
    app: nginx
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
      initContainers:
      - name: wait
        image: busybox:1.28
        command: ['sh', '-c', "until nslookup nginx.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for service; sleep 2; done"]
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.

```
root@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-02.yaml
deployment.apps/deployment-02 created
root@baranov:/home/baranovsa/kube-1.3#
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                        	 READY   STATUS     RESTARTS    AGE
deployment-02-77cfb9bc7-db4z4    0/1 	 Init:0/1   0      	33s
root@baranov:/home/baranovsa/kube-1.3#

```

3. Создать и запустить Service. Убедиться, что Init запустился.

Добавляем сервис web:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - name: web
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx
```

4. Продемонстрировать состояние пода до и после запуска сервиса.

```
root@baranov:/home/baranovsa/kube-1.3# kubectl get svc
NAME     	TYPE        CLUSTER-IP  	EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.96.0.1   	<none>        443/TCP   4d22h
nginx    	ClusterIP   10.111.84.132       <none>        80/TCP	4m34s
root@baranov:/home/baranovsa/kube-1.3#

root@baranov:/home/baranovsa/kube-1.3# kubectl apply -f deployment-02.yaml
deployment.apps/deployment-02 unchanged
service/nginx created
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                        	READY   STATUS     RESTARTS     AGE
deployment-02-77cfb9bc7-db4z4   0/1 	Init:0/1   0      	58m
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                        	READY   STATUS           RESTARTS   AGE
deployment-02-77cfb9bc7-db4z4   0/1 	PodInitializing  0          58m
root@baranov:/home/baranovsa/kube-1.3# kubectl get pods
NAME                        	READY   STATUS    RESTARTS   AGE
deployment-02-77cfb9bc7-db4z4   1/1 	Running   0          58m
root@baranov:/home/baranovsa/kube-1.3#
```

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
