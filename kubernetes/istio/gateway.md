---
tags:
  - gateway
  - istiogateway 
  - ingressgateway
---

# Istio Gateway для управления входящим трафиком

**Gateway** в Istio — это объект, который управляет входящим (Ingress) и исходящим (Egress) трафиком **на уровне сети**, предоставляя возможности маршрутизации, балансировки нагрузки, а также поддержку TLS и HTTP/HTTPS.

## Основные возможности

- Прием **внешнего трафика** в кластер Kubernetes (Ingress).
- Управление **исходящим трафиком** из кластера (Egress).
- **TLS-терминация** (разгрузка HTTPS).
- **Маршрутизация** и балансировка нагрузки.
- Интеграция с **VirtualService** для гибкого контроля трафика.

В отличие от стандартного **Ingress** в Kubernetes, **Gateway в Istio** управляет трафиком через **Envoy Proxy**, что дает **больше возможностей** для маршрутизации.

**Ключевые отличия от Kubernetes Ingress:**

| **Функция**           | **Kubernetes Ingress** | **Istio Gateway + VirtualService**         |
| --------------------- | ---------------------- | ------------------------------------------ |
| Управление HTTP/HTTPS | ✅ Да                   | ✅ Да                                       |
| Балансировка нагрузки | ❌ Ограниченная         | ✅ Гибкая (по заголовкам, версиям, cookies) |
| A/B-тестирование      | ❌ Нет                  | ✅ Да                                       |
| Канареечные релизы    | ❌ Нет                  | ✅ Да                                       |
| Fault Injection       | ❌ Нет                  | ✅ Да                                       |
| Интеграция с Istio    | ❌ Нет                  | ✅ Полная                                   |

## Установка Gateway в Istio

### Создание Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway  # Используем Istio Ingress Gateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"  # Принимаем запросы с любых доменов
```

### Создание VirtualService, связанный с Gateway

Gateway **только принимает трафик**, но маршрутизация на сервисы идет через **VirtualService**.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"  # Принимаем запросы со всех доменов
  gateways:
    - bookinfo-gateway  # Привязываем к Gateway
  http:
    - match:
        - uri:
            prefix: "/productpage"  # Маршрутизируем /productpage
      route:
        - destination:
            host: productpage  # Направляем трафик в сервис productpage
            port:
              number: 9080
```

## Полное описание всех полей Gateway в Istio

**Gateway** в Istio управляет **входящим и исходящим трафиком** на уровне сети, предоставляя гибкую маршрутизацию и защиту.

### Общая структура Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway  # Выбираем Istio Ingress Gateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: my-tls-secret
      hosts:
        - "example.com"
```

#### Основные секции
- selector – указывает, к какому Pod (Ingress/Egress) применять Gateway.
- servers – определяет, на каких портах слушать трафик.
- hosts – список доменов, которые обрабатывает Gateway.
- tls – конфигурация TLS (SSL).

##### spec.selector – выбор Gateway
```yaml
selector:
  istio: ingressgateway
```

- Определяет **Ingress/Egress Gateway**, к которому применяется конфигурация.
- Значение `istio: ingressgateway` означает, что конфигурация применяется к istio-ingressgateway.

##### spec.servers – настройка серверов

```yaml
servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
      - "example.com"
      - "*.example.com"
```

- port.number – номер порта (обычно 80, 443 или кастомный).
- port.name – название порта (должно быть http, https, tcp, tls).
- port.protocol – протокол (HTTP, HTTPS, TCP, TLS, GRPC).
- hosts – список **доменов**, для которых работает Gateway ("*" означает все домены).

##### spec.servers[].tls – настройка HTTPS (TLS)

```yaml
tls:
  mode: SIMPLE
  credentialName: my-tls-secret
```

- mode – режим TLS:
	- SIMPLE – стандартная TLS-терминация.
	- PASSTHROUGH – передача TLS без расшифровки (для mTLS).
	- MUTUAL – мьютуальная TLS-аутентификация.
	- AUTO_PASSTHROUGH – автоматический passthrough для HTTPS.
- credentialName – имя **Kubernetes Secret** с сертификатом.

##### spec.servers[].hosts – домены, обрабатываемые Gateway

```yaml
hosts:
  - "example.com"
  - "*.example.com"
```

- Определяет **список доменов**, для которых действует Gateway.
- Можно указать **поддомены** ("*.example.com" – все поддомены example.com).
- "*" – обрабатывает трафик **с любых доменов** (небезопасно).

##### spec.servers[].port.protocol – поддерживаемые протоколы

```yaml
protocol: HTTP
```

Определяет **тип трафика**, который принимает Gateway:
- HTTP – обычный HTTP.
- HTTPS – защищенный трафик (требует tls).
- TCP – TCP-соединение.
- TLS – защищенный трафик без расшифровки (mTLS).
- GRPC – трафик для gRPC-сервисов.

Пример 