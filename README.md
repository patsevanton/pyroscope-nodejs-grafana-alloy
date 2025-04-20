# Запуск pyroscope. Профилирование простого nodejs проекта без Grafana Alloy

[Pyroscope](https://github.com/grafana/pyroscope) — это инструмент для непрерывного профилирования кода, который помогает находить узкие места в производительности приложений.

Основные особенности:
- Анализирует CPU, память и другие метрики в реальном времени.
- Поддерживает несколько языков (Go, Python, Java, Ruby и др.).
- Интегрируется с Grafana, Kubernetes и другими инструментами мониторинга.
- Позволяет сравнивать профили за разные периоды.
- Используется для оптимизации производительности сервисов.

В этом посте будет продемонстрирован запуск Pyroscope в docker-compose и в kubernetes.
Затем будет запущено тестовое NodeJS приложение со специально написанными медленной функцией и функцией с утечкой памяти.
Отправка данных делается самим NodeJS приложением в Pyroscope, поэтому Grafana Alloy отключен.

## Быстрый запуск kubernetes в docker-compose
### Запускаем нагрузку в:
```shell
while true; do curl http://localhost:3000/fast; curl http://localhost:3000/slow; curl http://localhost:3000/leak; done
```

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

### Устанавливаем Pyroscope в namespace 'pyroscope' используя Helm chart из репозитория Grafana
Передаем кастомные настройки из файла values_pyroscope.yaml
```shell
helm upgrade -n pyroscope --create-namespace --install pyroscope grafana/pyroscope --values values_pyroscope.yaml
```

### Устанавливаем/обновляем Grafana с настройками:
Передаем кастомные настройки из файла values_pyroscope.yaml
```shell
helm upgrade -n grafana --create-namespace --install grafana grafana/grafana -f values_grafana.yaml
```

## Запускаем nodejs-app в kubernetes
```shell
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
![](https://habrastorage.org/webt/zk/3l/at/zk3latbwghzaxdnuyezha0mx7zc.png)

![](https://habrastorage.org/webt/5r/p0/_w/5rp0_wgdcnili5ytzo-ovv1bib8.png)

Pyroscope через Grafana плагин

![](https://habrastorage.org/webt/jc/dl/jj/jcdljjhrd7qfjxcpmi3vc6jwkns.png)

![](https://habrastorage.org/webt/dc/sz/ir/dcszirjryh_ugttlhdnraq6m7fk.png)

![](https://habrastorage.org/webt/nc/qx/ve/ncqxveh6g1lo6xndhcg_mb46fwy.png)

![](https://habrastorage.org/webt/z3/ou/ud/z3ouudbzmayznipiak-5w7frvjc.png)

![](https://habrastorage.org/webt/3g/fu/87/3gfu87hmdm1h-ckgh6d6tzsellu.png)


Что мы можем сказать по поводу NodeJS кода после профилирования с помощью Pyroscope:
- Мы видим что функция leakMemoryRoute в файле ./app.js на строке 63 имеет утечку памяти и занимает 223MB.
- Мы видим что функция slowRoute в файле ./app.js на 51 строке долго отвечает 6.96 mins.
