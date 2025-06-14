# Запуск pyroscope. Профилирование простого nodejs проекта c Grafana Alloy

[Pyroscope](https://github.com/grafana/pyroscope) — это инструмент для непрерывного профилирования кода, который помогает находить узкие места в производительности приложений.

Основные особенности:
- Анализирует CPU, память и другие метрики в реальном времени.
- Поддерживает несколько языков (Go, Python, Java, Ruby и др.).
- Интегрируется с Grafana, Kubernetes и другими инструментами мониторинга.
- Позволяет сравнивать профили за разные периоды.
- Используется для оптимизации производительности сервисов.

В этом посте будет продемонстрирован запуск Pyroscope в kubernetes.
Затем будет запущено тестовое NodeJS приложение со специально написанными медленной функцией и функцией с утечкой памяти.
Отправка данных делается Grafana Alloy приложением c отправкой в Pyroscope

## Быстрый запуск kubernetes в Yandex Cloud
### Устанавливаем kubernetes
```shell
git clone https://github.com/patsevanton/pyroscope-nodejs
export YC_FOLDER_ID='ваша_folder_id'
terraform apply
yc managed-kubernetes cluster get-credentials --id xxxx --force
```

### Добавляем репозиторий Helm charts от Grafana в локальный список репозиториев
```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Устанавливаем Alloy в namespace alloy используя Helm chart из репозитория Grafana
Передаем кастомные настройки из файла values_alloy.yaml
```shell
helm upgrade -n alloy --create-namespace --install alloy grafana/alloy --values values_alloy.yaml
```

### Устанавливаем Pyroscope в namespace pyroscope используя Helm chart из репозитория Grafana
Передаем кастомные настройки из файла values_pyroscope.yaml
```shell
helm upgrade -n pyroscope --create-namespace --install pyroscope grafana/pyroscope --values values_pyroscope.yaml
```

### Устанавливаем/обновляем Grafana с настройками:
Передаем кастомные настройки из файла values_pyroscope.yaml
```shell
helm upgrade -n grafana --create-namespace --install grafana grafana/grafana -f values_grafana.yaml
```

## Запускаем python-fast-slow в kubernetes
```shell
kubectl create namespace pyroscope-ebpf
kubectl apply -f kubernetes/python-fast-slow.yaml
kubectl apply -f kubernetes/java-fast-slow.yaml
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/ingress.yaml
```

### Генерируем нагрузку для профилирования через kubernetes ingress:
```shell
while true; do curl nodejs-app.apatsev.org.ru/fast; curl nodejs-app.apatsev.org.ru/slow; curl nodejs-app.apatsev.org.ru/leak; done
```

### Смотрим сколько pod занимают память
```shell
k top pod
NAME                          CPU(cores)   MEMORY(bytes)   
nodejs-app-77f7b96899-7cff6   19m          1652Mi          
nodejs-app-77f7b96899-hvfrl   32m          1676Mi 
```

### Скриншоты
Собственный UI Pyroscope:


Pyroscope через Grafana плагин


Что мы можем сказать по поводу NodeJS кода после профилирования с помощью Pyroscope:
- Мы видим что функция leakMemoryRoute в файле ./app.js на строке 63 имеет утечку памяти и занимает 223MB.
- Мы видим что функция slowRoute в файле ./app.js на 51 строке долго отвечает 6.96 mins.
