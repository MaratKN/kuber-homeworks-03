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


Создаем deployment.yaml

root@DebianNew:~/.kube# nano deployment.yaml 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
```

Затем 

root@DebianNew:~/.kube# kubectl apply -f deployment.yaml

И ловим ошибку

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/1_v2.png)

Видим, что порт 80 уже занят

Добавляем в deployment.yaml 

```
env:
        - name: HTTP_PORT
          value: "7080"
```

root@DebianNew:~/.kube# nano deployment.yaml 

Получаем

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "7080"
```

root@DebianNew:~/.kube# kubectl apply -f deployment.yaml

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/2.png)

Меняем количество реплик с 1 на 2

root@DebianNew:~/.kube# nano deployment.yaml 

root@DebianNew:~/.kube# kubectl apply -f deployment.yaml

deployment.apps/multitool configured

root@DebianNew:~/.kube# kubectl get pods -A

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/3.png)

Создаем Service, который обеспечит доступ до реплик приложений из предыдущих пунктов

root@DebianNew:~/.kube# nano deployment-svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: deployment-svc
spec:
  selector:
    app: multitool
  ports:
  - name: for-nginx
    port: 80
    targetPort: 80
  - name: for-multitool
    port: 7080
    targetPort: 7080
```


root@DebianNew:~/.kube# kubectl apply -f deployment-svc.yaml 

service/deployment-svc created

root@DebianNew:~/.kube# kubectl get svc -o wide 

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/4.png)

Создаём отдельный Pod с приложением multitool и убеждаемся с помощью curl, что из пода есть доступ до приложений из предыдущих пунктов

root@DebianNew:~/.kube# nano multitool-app.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: multitool
  name: multitool-app
  namespace: default
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool
    ports:
    - containerPort: 8080
    env:
      - name: HTTP_PORT
        value: "7080"
```

root@DebianNew:~/.kube# kubectl apply -f multitool-app.yaml

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/5.png)

root@DebianNew:~/.kube# kubectl exec multitool-app -- curl 10.1.83.204:80 
root@DebianNew:~/.kube# kubectl exec multitool-app -- curl 10.1.83.203:80 

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/6.png)
![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/7.png)

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.


Создадим deployment-init.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-app
  name: nginx-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      initContainers:
      - name: init-nginx-svc
        image: busybox
        command: ['sh', '-c', 'until nslookup nginx-svc.default.svc.cluster.local; do echo waiting for nginx-svc; sleep 5; done;']
```

root@DebianNew:~/.kube# nano deployment-init.yaml

root@DebianNew:~/.kube# kubectl apply -f deployment-init.yaml 

deployment.apps/nginx-app created

root@DebianNew:~/.kube# kubectl get pods -o wide

Nginx не стартует

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/8.png)

Создадим Service nginx-svc.yaml и убедимся, что init запустился

root@DebianNew:~/.kube# nano nginx-svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - name: http-port
    port: 80
    protocol: TCP
    targetPort: 80
```

root@DebianNew:~/.kube# kubectl apply -f nginx-svc.yaml 

root@DebianNew:~/.kube# kubectl get pods -o wide

![alt text](https://github.com/MaratKN/kuber-homeworks-03/blob/main/9.png)

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
