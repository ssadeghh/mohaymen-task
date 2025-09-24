# پروژه ELK Stack با Kubernetes
این پروژه یک پیاده‌سازی ساده از ELK Stack (Elasticsearch + Logstash + Kibana) روی Kubernetes است که داده‌های JSON را دریافت، پردازش و در Kibana قابل مشاهده می‌کند. همچنین شامل Sidecar Archiver برای فشرده‌سازی لاگ‌هاست.

## 🔹 پیش‌نیازها
قبل از اجرای پروژه باید موارد زیر روی سیستم نصب و آماده باشند:

Minikube
 یا هر کلاستر Kubernetes دیگر

kubectl

دسترسی به Docker (برای pull کردن ایمیج‌های ELK)

دایرکتوری‌های مورد نیاز روی هاست:
```bash
/home/docker/data
/home/docker/data/archive
/home/docker/esdata
```

## 🔹 ساختار پروژه
```bash
.
├── data/                           # دایرکتوری برای فایل‌های JSON ورودی
├── elasticsearch.yaml              # StatefulSet Elasticsearch
├── elasticsearch-service.yaml      # سرویس ClusterIP برای Elasticsearch
├── kibana.yaml                      # Deployment Kibana
├── kibana-service.yaml              # NodePort سرویس Kibana
├── logstash-configmap.yaml          # ConfigMap شامل pipeline Logstash
├── logstash-deployment.yaml         # Deployment Logstash با Sidecar Archiver
├── logstash-service.yaml            # سرویس برای Logstash
```

## 🔹 Kubernetes resources
### 1. Elasticsearch

- نوع: ```StatefulSet```

- ایمیج: ```docker.elastic.co/elasticsearch/elasticsearch:8.11.1```

- پورت: ```9200```

- volumes: ```/usr/share/elasticsearch/data``` → ```/home/docker/esdata``` روی هاست

- تنظیمات:

  - ```discovery.type=single-node``` → حالت تک‌نود

  - امنیت غیر فعال (```xpack.security.enabled=false``` و ```xpack.security.http.ssl.enabled=false```)

  - حافظه ```JVM```: ```-Xms1g -Xmx1g```

### 2. Elasticsearch Service

- نوع: ```ClusterIP```

- پورت: ```9200```

- دسترسی از داخل کلاستر

------- 
### 3. Kibana

- نوع: ```Deployment```

- ایمیج: ```docker.elastic.co/kibana/kibana:8.11.1```

- پورت: ```5601```

- اتصال به ```Elasticsearch```: ```ELASTICSEARCH_HOSTS=http://elasticsearch:9200```

### 4. Kibana Service

- نوع: ```NodePort``` → برای دسترسی از خارج کلاستر

- پورت: ```5601```


### 5. Logstash

- نوع: ```Deployment```

- ایمیج اصلی: ```docker.elastic.co/logstash/logstash:8.11.1```

- ```Sidecar Archiver```:

  - ایمیج: ```alpine:3.20```

  - وظیفه: فشرده‌سازی لاگ‌های ```JSON``` و ذخیره در ```/archive/compressed```

- حجم‌ها:

  - ```/usr/share/logstash/pipeline``` → ```pipeline``` از ```ConfigMap```

  - ```/data``` → داده‌های ورودی ```JSON```

  - ```/archive``` → فایل‌های خروجی و آرشیو

### 6. Logstash ConfigMap

- مسیر ```pipeline: /data/*.json```

- ```code```: ```multiline``` برای ```JSON``` چند خطی

- ```filter```:

  - ```parse JSON```

  - حذف رویدادهای ناقص

  - حذف فیلدهای اضافی (```message```, ```host```, ```path```, ```@version```)

- ```output```:

  - ```Elasticsearch: index``` با الگوی ```json-logs-%{+YYYY.MM.dd}```

  - ```stdout``` برای دیباگ

  - فایل ```JSON``` در ```/archive/logs-%{+YYYY-MM-dd}.json```

### 7. Logstash Service

- نوع: ```ClusterIP```

- پورت: ```5044```

- اتصال به ```Logstash``` از داخل کلاستر

## 🔹 راه‌اندازی پروژه
1. ایجاد ```namespace```:
```bash
kubectl create namespace elk
```

2. deploy ```Elasticsearch```:
```bash
kubectl apply -f elasticsearch.yaml
kubectl apply -f elasticsearch-service.yaml
```

3. deploy ```Kibana```:
```bash
kubectl apply -f kibana.yaml
kubectl apply -f kibana-service.yaml
```

4. deploy ```Logstash```:
```bash
kubectl apply -f logstash-configmap.yaml
kubectl apply -f logstash-deployment.yaml
kubectl apply -f logstash-service.yaml
```

5. بررسی وضعیت ```Pod```‌ها:
```bash
kubectl get pods -n elk
```

6. دسترسی به ```Kibana```:
```bash
kubectl get svc -n elk kibana
```

## 🔹 نحوه استفاده
- فایل‌های ```JSON``` خود را در مسیر ```/home/docker/data/``` قرار دهید.

- ```Logstash``` به صورت خودکار آن‌ها را پردازش و به ```Elasticsearch``` می‌فرستد.

- ```Kibana``` برای مشاهده داده‌ها: ایجاد ```Index Pattern json-logs-*``` و بررسی لاگ‌ها.

- ```Sidecar Archiver``` فایل‌ها را به صورت فشرده در ```/archive/compressed``` ذخیره می‌کند.


## 🔹 نتیجه

بعد از اجرای پروژه:

- ```Elasticsearch``` داده‌ها را ذخیره می‌کند.

- ```Logstash``` داده‌های ```JSON``` را ```parse```، فیلتر و ذخیره می‌کند.

- ```Kibana``` امکان مشاهده و تحلیل داده‌ها را فراهم می‌کند.

- فایل‌های لاگ آرشیو شده در ```/archive/compressed``` قابل دسترسی هستند.