# solution.md

## 1. Суммарный объем данных по IP (bash one‑liner)

``` bash
awk '{sum[$1]+=$10} END {for (ip in sum) printf "%d bytes for %s\n", sum[ip], ip}' access.log
```

------------------------------------------------------------------------

## 2. Все ESTABLISHED TCP соединения через lsof

``` bash
lsof -n -i tcp -s tcp:established
```

------------------------------------------------------------------------

## 3. Причина недоступности сервиса

По графикам:

-   около **13:00** (12:58?)
-   **MySQL connections достигают max_connections ≈ 3000**
-   **Threads running ≈ 3000**
-   **Questions/sec падает почти до 0**

### Причина

База данных достигла **лимита подключений (`max_connections`)**, новые
подключения не принимаются → приложение не может выполнять запросы.

### Время недоступности

Примерно **13:00 -- 13:20**.

------------------------------------------------------------------------

## 4. Проблемы Dockerfile

Исходный:

``` dockerfile
FROM ubuntu:latest
MAINTAINER MyCompany
COPY . /var/www/html
RUN apt-get update -y
RUN apt-get install -y nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Ошибки

**1. `ubuntu:latest`** - нестабильная версия\
использовать фиксированную

    FROM ubuntu:22.04

**2. `MAINTAINER`** - устаревшая инструкция\
заменить на

    LABEL maintainer="MyCompany"

**3. Лишние слои**

    RUN apt-get update
    RUN apt-get install

объединить

**4. Нет очистки apt cache** - увеличивает размер образа

**5. COPY до установки пакетов** - ломает cache

### Исправленный Dockerfile

``` dockerfile
FROM ubuntu:22.04

LABEL maintainer="MyCompany"

RUN apt-get update  && apt-get install -y nginx  && rm -rf /var/lib/apt/lists/*

COPY . /var/www/html

CMD ["nginx", "-g", "daemon off;"]
```

------------------------------------------------------------------------

## 5. Nginx не отвечает --- действия

**1.  Проверить процесс**

    systemctl status nginx
    ps aux | grep nginx


**2.  Проверить порт**

    ss -tulnp | grep :80

**3.  Проверить логи**

    tail -f /var/log/nginx/error.log
    journalctl -u nginx

**4.  Проверить ресурсы**

    top
    free -m
    df -h

**5.  Проверить конфиг**

    nginx -t

**6.  При необходимости**

    systemctl restart nginx

**7.  Проверить:**

-   backend
-   firewall
-   последние деплои
-   нагрузку

Цель -- **быстро восстановить сервис**, затем RCA.

------------------------------------------------------------------------

## 6. Развертывание в Kubernetes

### Deployment

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app-image
        ports:
        - containerPort: 80
```

### Service

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### Применение

    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml

### Масштабирование

    kubectl scale deployment my-app --replicas=5

------------------------------------------------------------------------

## 7. Сообщение заказчика: «Проект не работает»

1.  Подтвердить получение и начать проверку.
2.  Проверить мониторинг и алерты.
3.  Определить масштаб проблемы.
4.  Проверить:
    -   сервисы
    -   логи
    -   ресурсы
    -   сеть
    -   последние релизы.
5.  Быстрое восстановление:
    -   рестарт
    -   rollback
    -   масштабирование.
6.  Сообщать статус заказчику.
7.  После восстановления --- **RCA и предотвращение повторения**.
