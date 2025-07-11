---
marp: true
---

# Распределенное логирование с Jaeger
`Jaeger` — это система с открытым исходным кодом для распределенного трейсинга, разработанная для мониторинга и отладки микросервисных архитектур. Она помогает отслеживать запросы, проходящие через различные сервисы, и выявлять узкие места и проблемы производительности.

---
# Основные компоненты Jaeger
`Agent`: Легковесный процесс, который собирает трейсинг-данные от приложений и отправляет их в Collector.
`Collector`: Принимает данные от агентов и сохраняет их в хранилище (например, Elasticsearch, Cassandra).
`Query`: Веб-интерфейс для поиска и визуализации трейсинг-данных.
`Ingester`: Компонент, который читает данные из Kafka и записывает их в хранилище.

---
# Основные возможности Jaeger
`Трейсинг запросов`: Отслеживание пути запроса через различные микросервисы.
`Анализ производительности`: Выявление узких мест и проблем производительности.
`Поиск и визуализация`: Поиск и визуализация трейсинг-данных через веб-интерфейс.
`Поддержка различных хранилищ`: Поддержка хранения данных в Elasticsearch, Cassandra, Kafka и других системах.

---
# Развертывание Jaeger в Kubernetes
Для развертывания Jaeger в Kubernetes можно использовать Helm Chart или манифесты YAML. Рассмотрим пример развертывания с использованием манифестов YAML.

---
Шаг 1: Создание пространства имен
Создайте пространство имен для Jaeger:
```bash
kubectl create namespace observability
```
---
Шаг 2: Развертывание Jaeger
Создайте манифест для развертывания Jaeger:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-agent
  namespace: observability
spec:
  ports:
  - name: jaeger-udp
    port: 6831
    protocol: UDP
  - name: jaeger-udp-sampling
    port: 5778
    protocol: UDP
  selector:
    app: jaeger
```
---
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:1.22
        ports:
        - containerPort: 6831
          protocol: UDP
        - containerPort: 5778
          protocol: UDP
      - name: jaeger-collector
        image: jaegertracing/jaeger-collector:1.22
        ports:
        - containerPort: 14267
        - containerPort: 14268
        - containerPort: 14250
      - name: jaeger-query
        image: jaegertracing/jaeger-query:1.22
        ports:
        - containerPort: 16686
```
---
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: observability
spec:
  ports:
  - name: http
    port: 80
    targetPort: 16686
  selector:
    app: jaeger
  type: LoadBalancer
```
---
Шаг 3: Применение манифестов
Примените манифесты для развертывания Jaeger:
```bash
kubectl apply -f jaeger-deployment.yaml
```

---
# Интеграция Jaeger с приложениями

Пример на Python с использованием библиотеки jaeger-client:

```python
from jaeger_client import Config
import logging

def init_jaeger_tracer(service_name='my-service'):
    logging.getLogger('').handlers = []
    logging.basicConfig(format='%(message)s', level=logging.DEBUG)

    config = Config(
        config={
            'sampler': {'type': 'const', 'param': 1},
            'logging': True,
        },
        service_name=service_name,
    )
    return config.initialize_tracer()

tracer = init_jaeger_tracer()
# Пример использования трейсера
with tracer.start_span('my-operation') as span:
    span.log_kv({'event': 'test message', 'life': 42})
```
<!--
Для интеграции Jaeger с вашими приложениями, вам нужно добавить клиентские библиотеки Jaeger в ваш код и настроить их для отправки трейсинг-данных в Jaeger Agent.
-->
---
# Заключение
Jaeger предоставляет мощные инструменты для распределенного трейсинга, которые помогают отслеживать и анализировать запросы в микросервисных архитектурах. Развертывание Jaeger в Kubernetes и интеграция с вашими приложениями позволяет улучшить мониторинг и отладку, выявлять узкие места и повышать производительность ваших систем.

**Links**
https://medium.com/jaegertracing/jaeger-tracing-a-friendly-guide-for-beginners-7b53a4a568ca