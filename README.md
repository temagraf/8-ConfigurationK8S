# Домашнее задание к занятию «Конфигурация приложений»

## Цель задания

В тестовой среде Kubernetes необходимо создать конфигурацию и продемонстрировать работу приложения.

## Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (MicroK8s)
2. Установленный локальный kubectl
3. Редактор YAML-файлов с подключенным GitHub-репозиторием

## Задание 1: Создать Deployment приложения и решить возникшую проблему с помощью ConfigMap

1) Создать Deployment приложения, состоящего из контейнеров nginx и multitool.
2) Решить возникшую проблему с помощью ConfigMap.
3) Продемонстрировать, что pod стартовал и оба конейнера работают.
4) Сделать простую веб-страницу и подключить её к Nginx с помощью ConfigMap. Подключить Service и показать вывод curl или в браузере.
5) Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Решение

1. Создан Deployment с двумя контейнерами:  
    nginx (порт 80)  
    multitool (порт 8080)  

### Создаем файл deployment.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
```
Применяем и видим конфликт портов. 

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/1-1%20конфликт%20портов.png)

### 2. Исправление проблемы с помощью ConfigMap

### Создаем configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-multitool-config
data:
  nginx.conf: |
    events {
      worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      server {
        listen 80;
        server_name localhost;

        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Welcome to Nginx</title>
    </head>
    <body>
        <h1>Welcome to Nginx on Kubernetes!</h1>
        <p>This page is served via ConfigMap</p>
    </body>
    </html>
```

### Обновляем файл deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: index-html
          mountPath: /usr/share/nginx/html/
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        ports:
        - containerPort: 8080
          name: multitool
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-multitool-config
      - name: index-html
        configMap:
          name: nginx-multitool-config
```

### Применяем конфигурации

```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
```

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/1-2%20ConfigMap.png)

### Проверяем статус пода

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/1-3%20статус%20пода.png)


Создание Service и проверка доступности веб-страницы

Создаем файл service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
spec:
  selector:
    app: nginx-multitool
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: multitool
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

### Применяем Service

```bash
kubectl apply -f service.yaml
kubectl get services
```
### Проверка доступности подов

```bash
kubectl get pods -o wide
```
### Проверка доступа к веб-странице через IP пода

```bash
curl 10.1.46.6:80
```
### Результат:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to Nginx</title>
</head>
<body>
    <h1>Welcome to Nginx on Kubernetes!</h1>
    <p>This page is served via ConfigMap</p>
</body>
</html>
```

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/1-4-5.png) 


### Проверка работы multitool

```bash
curl 10.1.46.6:80
```
### Проверка работы multitool:

```text
WBITT Network MultiTool (with NGINX) - nginx-multitool-6fdf78bb44-lfbmg - 10.1.46.6 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/1-4-2.png)


### Использованные манифесты

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: index-html
          mountPath: /usr/share/nginx/html/
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        ports:
        - containerPort: 8080
          name: multitool
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-multitool-config
      - name: index-html
        configMap:
          name: nginx-multitool-config
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
spec:
  selector:
    app: nginx-multitool
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: multitool
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-multitool-config
data:
  nginx.conf: |
    events {
      worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      server {
        listen 80;
        server_name localhost;

        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Welcome to Nginx</title>
    </head>
    <body>
        <h1>Welcome to Nginx on Kubernetes!</h1>
        <p>This page is served via ConfigMap</p>
    </body>
    </html>
```

## Задание 2. Создать приложение с вашей веб-страницей, доступной по HTTPS

1) Создать Deployment приложения, состоящего из Nginx.
2) Создать собственную веб-страницу и подключить её как ConfigMap к приложению.
3) Выпустить самоподписной сертификат SSL. Создать Secret для использования сертификата.
4) Создать Ingress и необходимый Service, подключить к нему SSL в вид. Продемонстировать доступ к приложению по HTTPS.
5) Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Создание SSL-сертификата и Secret

1. Создаем SSL-сертификат

```bash
mkdir -p ssl
cd ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginx-multitool.com"
cd ..
```

2. Создаем Secret с сертификатом

```bash
kubectl create secret tls nginx-tls-secret \
  --key ssl/tls.key \
  --cert ssl/tls.crt
```
![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/2-1.png)

### Обновляем configmap.yaml для поддержки HTTPS

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-multitool-config
data:
  nginx.conf: |
    events {
      worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      server {
        listen 80;
        listen 443 ssl;
        server_name nginx-multitool.com;
        
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
        
        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>HTTPS Nginx on Kubernetes</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                margin: 40px;
                background-color: #f0f0f0;
            }
            .container {
                background-color: white;
                padding: 20px;
                border-radius: 5px;
                box-shadow: 0 0 10px rgba(0,0,0,0.1);
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to HTTPS-enabled Nginx!</h1>
            <p>This page is served securely via HTTPS</p>
        </div>
    </body>
    </html>
```

### Применяем обновленный ConfigMap

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/2-2применения%20configmap.png)

### Настройка Ingress

Создаем ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-multitool-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - nginx-multitool.com
    secretName: nginx-tls-secret
  rules:
  - host: nginx-multitool.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-multitool-service
            port:
              number: 80
```

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/2-3%20статус%20ingress.png)


### Проверка работы HTTPS

Добавляем запись в /etc/hosts

```bash
sudo bash -c 'echo "127.0.0.1 nginx-multitool.com" >> /etc/hosts'
```

Проверяем доступность по HTTPS

```bash
curl -k https://nginx-multitool.com
```

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/2-4%20результата%20curl%20с%20HTTPS.png) 

![image](https://github.com/temagraf/8-ConfigurationK8S/blob/main/555.png)


### Использованные манифесты:

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-multitool-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - nginx-multitool.com
    secretName: nginx-tls-secret
  rules:
  - host: nginx-multitool.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-multitool-service
            port:
              number: 80
```

### Обновленный configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-multitool-config
data:
  nginx.conf: |
    events {
      worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      server {
        listen 80;
        listen 443 ssl;
        server_name nginx-multitool.com;
        
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
        
        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>HTTPS Nginx on Kubernetes</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                margin: 40px;
                background-color: #f0f0f0;
            }
            .container {
                background-color: white;
                padding: 20px;
                border-radius: 5px;
                box-shadow: 0 0 10px rgba(0,0,0,0.1);
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to HTTPS-enabled Nginx!</h1>
            <p>This page is served securely via HTTPS</p>
        </div>
    </body>
    </html>
```



### Результат

- Создан и настроен SSL-сертификат  
- Настроен HTTPS через Ingress  
- Веб-страница доступна по HTTPS  
- Работает редирект с HTTP на HTTPS
