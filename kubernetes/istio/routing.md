---
tags:
  - virtualservice
  - "#destinationRule"
---
# VirtualService

VirtualService управляет входящим трафиком и направляет его в зависимости от **URI, заголовков, веса и версии приложения**.

## Основные поля VirtualService

| **Поле**         | **Описание**                                                                     |
| ---------------- | -------------------------------------------------------------------------------- |
| hosts            | К какому сервису применяется                                                     |
| gateways         | Используемые **Istio Gateway** (если не указан, значит внутренняя маршрутизация) |
| http / tcp / tls | Определение правил маршрутизации                                                 |
| route            | Список Destination, куда пойдет трафик                                           |
| match            | Условия (заголовки, пути, методы и т. д.)                                        |

## Пример VirtualService (Маршрутизация по версиям)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app.default.svc.cluster.local  # Применяется к сервису my-app
  http:
    - match:
        - headers:
            user-agent:
              regex: ".*Chrome.*"  # Трафик от Chrome направлять в v2
      route:
        - destination:
            host: my-app.default.svc.cluster.local
            subset: v2
    - route:  
        - destination:
            host: my-app.default.svc.cluster.local
            subset: v1  # Остальной трафик идет в v1
```

## VirtualService — Поля и описание

### Основные поля VirtualService

| **Поле** | **Тип**     | **Описание**                                                               |
| -------- | ----------- | -------------------------------------------------------------------------- |
| hosts    | []string    | **Список сервисов**, к которым применяется VirtualService                  |
| gateways | []string    | **Список Istio Gateway**, через которые можно направлять трафик            |
| http     | []HTTPRoute | **Правила маршрутизации HTTP**                                             |
| tcp      | []TCPRoute  | **Правила маршрутизации TCP**                                              |
| tls      | []TLSRoute  | **Правила маршрутизации TLS**                                              |
| exportTo | []string    | **Где можно использовать VirtualService** (по умолчанию *, т.е. глобально) |

### Поля http (HTTPRoute)

| **Поле**   | **Тип**                | **Описание**                                                |
| ---------- | ---------------------- | ----------------------------------------------------------- |
| match      | []HTTPMatchRequest     | **Фильтр входящих запросов** (по заголовкам, методам, пути) |
| route      | []HTTPRouteDestination | **Куда отправлять запросы**                                 |
| redirect   | HTTPRedirect           | **Перенаправление HTTP (301/302)**                          |
| rewrite    | HTTPRewrite            | **Изменение URL перед отправкой на бэкенд**                 |
| timeout    | Duration               | **Таймаут запроса** (например, 5s)                          |
| retries    | HTTPRetry              | **Политика повторных попыток**                              |
| fault      | HTTPFaultInjection     | **Искусственное создание ошибок**                           |
| corsPolicy | CORSPolicy             | **Настройки CORS**                                          |

### Поля match (HTTPMatchRequest)

| **Поле**    | **Тип**                | **Описание**                                |
| ----------- | ---------------------- | ------------------------------------------- |
| uri         | StringMatch            | **Фильтр по пути (regex, prefix, exact)**   |
| method      | StringMatch            | **Фильтр по HTTP-методу (GET, POST, etc.)** |
| headers     | map[string]StringMatch | **Фильтр по заголовкам**                    |
| queryParams | map[string]StringMatch | **Фильтр по параметрам URL**                |

#### Пример фильтрации запроса по заголовку и методу

```yaml
match:
  - headers:
      user-agent:
        regex: ".*Firefox.*"
    method:
      exact: GET
```

### Поля route (HTTPRouteDestination)

| **Поле**    | **Тип**     | **Описание**                                   |
| ----------- | ----------- | ---------------------------------------------- |
| destination | Destination | **Конечный сервис** (куда направляется трафик) |
| weight      | int         | **Вес (в процентах) при балансировке**         |

#### Пример маршрутизации 80% трафика в v1, 20% — в v2

```yaml
route:
  - destination:
      host: my-app.default.svc.cluster.local
      subset: v1
    weight: 80
  - destination:
      host: my-app.default.svc.cluster.local
      subset: v2
    weight: 20
```

# DestinationRule — балансировка и политики

После того, как VirtualService направил трафик, **DestinationRule** определяет поведение на уровне подов.
## Основные поля DestinationRule

| **Поле**      | **Описание**                            |
| ------------- | --------------------------------------- |
| host          | К какому сервису применяется            |
| subsets       | Определяет версии (по labels)           |
| trafficPolicy | Балансировка, таймауты, политики сессии |

## Пример DestinationRule (Балансировка и политики отказа)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app.default.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN  # Балансировка по кругу
    connectionPool:
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

- Определяет **v1 и v2** как разные версии приложения.
- **Балансировка ROUND_ROBIN** между подами.
- **Лимит запросов** и защита от перегрузки.
- **Outlier Detection** (если под падает 5 раз подряд — его исключают).

## DestinationRule — Поля и описание

### Основные поля DestinationRule

| **Поле**      | **Тип**       | **Описание**                                                         |
| ------------- | ------------- | -------------------------------------------------------------------- |
| host          | string        | **Сервис, для которого создается правило**                           |
| subsets       | []Subset      | **Версии сервиса (по label’ам)**                                     |
| trafficPolicy | TrafficPolicy | **Настройки балансировки, отказоустойчивости и политики соединений** |

### Поля subsets (Subset)

Определяет **версии сервиса**, которые можно использовать в маршрутизации.

| **Поле**      | **Тип**           | **Описание**                       |
| ------------- | ----------------- | ---------------------------------- |
| name          | string            | **Название версии** (например, v1) |
| labels        | map[string]string | **Метка (label) для выбора подов** |
| trafficPolicy | TrafficPolicy     | **Свои правила для этой версии**   |

#### Пример определения версии v1 и v2

```yaml
subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Поля trafficPolicy (TrafficPolicy)

| **Поле**         | **Тип**                | **Описание**                        |
| ---------------- | ---------------------- | ----------------------------------- |
| loadBalancer     | LoadBalancerSettings   | **Стратегия балансировки**          |
| connectionPool   | ConnectionPoolSettings | **Лимиты на количество соединений** |
| outlierDetection | OutlierDetection       | **Защита от падения подов**         |

### Поля loadBalancer (LoadBalancerSettings)

| **Значение** | **Описание**                             |
| ------------ | ---------------------------------------- |
| ROUND_ROBIN  | **Балансировка по кругу** (по умолчанию) |
| LEAST_CONN   | **Наименьшее количество соединений**     |
| RANDOM       | **Случайное распределение**              |
| PASSTHROUGH  | **Без балансировки, напрямую в под**     |

#### Пример балансировки по наименьшему числу соединений

```yaml
trafficPolicy:
  loadBalancer:
    simple: LEAST_CONN
```

### Поля connectionPool (ConnectionPoolSettings)

| **Поле**                 | **Описание**                        |
| ------------------------ | ----------------------------------- |
| http1MaxPendingRequests  | **Максимум запросов в очереди**     |
| maxRequestsPerConnection | **Максимум запросов на соединение** |

```yaml
connectionPool:
  http:
    http1MaxPendingRequests: 10
    maxRequestsPerConnection: 1
```

### Поля outlierDetection (OutlierDetection)

| **Поле**          | **Описание**                                    |
| ----------------- | ----------------------------------------------- |
| consecutiveErrors | **Сколько ошибок подряд считается “проблемой”** |
| interval          | **Как часто проверять поды**                    |
| baseEjectionTime  | **Как долго исключать под**                     |

#### Пример исключения подов после 5 ошибок

```yaml
outlierDetection:
  consecutiveErrors: 5
  interval: 10s
  baseEjectionTime: 30s
```

# Как VirtualService и DestinationRule работают вместе

1. **VirtualService** выбирает, **куда** направить трафик (по subset).
2. **DestinationRule** управляет поведением трафика внутри выбранного сервиса.

**Пример схемы:**

```
Client → VirtualService (Маршрутизация) → DestinationRule (Балансировка)
```

# Проверка маршрутизации в Istio

Проверить, как Istio направляет трафик:

```bash
kubectl get virtualservice,destinationrule
```

Проверить маршруты в Istio:

```bash
istioctl proxy-config routes $(kubectl get pod -l app=my-app -o jsonpath='{.items[0].metadata.name}') -o json
```

Проверить балансировку:

```bash
for i in {1..10}; do curl -s http://my-app.default.svc.cluster.local; done
```

Теперь можно пробовать **канарейку, A/B тестирование и fault injection!**

