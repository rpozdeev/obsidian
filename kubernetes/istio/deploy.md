---
tags:
  - canary_deployment
  - a/b_testing
  - fault_injection
TQ_short_mode: false
---
# Канареечный релиз (Canary Deployment)

Канареечный релиз — это **поэтапное** развертывание новой версии сервиса. Мы сначала отправляем **малую часть трафика** на новую версию (v2), а потом постепенно увеличиваем нагрузку.

## Пример VirtualService для канареечного релиза

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90  # 90% трафика идет на v1
        - destination:
            host: reviews
            subset: v2
          weight: 10  # 10% трафика идет на v2 (канарейка)
```

## Пример DestinationRule для версий сервиса

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

# A/B тестирование (по заголовкам или кукам)

A/B тестирование позволяет **направлять определенных пользователей** на разные версии сервиса. Это удобно, если нужно тестировать новую версию на **определенных пользователей**.

## Пример DestinationRule для версий сервиса

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

## Пример VirtualService с A/B тестированием по заголовку

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: test-user
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
```

- Если **заголовок end-user: test-user → юзер попадает на v2.
- Все остальные → v1.

# Имитация отказов (Fault Injection)

**Fault Injection** — это механизм Istio, который позволяет **искусственно добавлять задержки и ошибки в сервисы**, чтобы тестировать **отказоустойчивость** приложения.

## Добавляем Fault Injection (Задержка 5 секунд)

Допустим, мы хотим проверить, **как отреагирует frontend**, если сервис reviews начнет отвечать с задержкой **5 секунд**.
Создадим **VirtualService**, который добавит задержку для **50% запросов**.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - fault:
        delay:
          fixedDelay: 5s  # Добавляем задержку 5 секунд
          percentage:
            value: 50  # 50% запросов будет задерживаться
      route:
        - destination:
            host: reviews
            subset: v1
```

## Добавляем Fault Injection (Возвращаем 500 ошибку)

Протестируем, **как отреагирует frontend**, если сервис reviews начнет возвращать **ошибку 500**.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - fault:
        abort:
          httpStatus: 500  # 50% запросов вернет 500
          percentage:
            value: 50
      route:
        - destination:
            host: reviews
            subset: v1
```

## Стратегии Fault Injection

### Delay Injection

#### Фиксированная задержка (fixedDelay)

```yaml
fault:
  delay:
    fixedDelay: 3s
    percentage:
      value: 50  # 50% запросов задерживается
```

- Проверка, как клиентские сервисы реагируют на долгий отклик.
- Тестирование таймаутов (timeout в DestinationRule).

#### Задержка с вероятностью 100%

```yaml
fault:
  delay:
    fixedDelay: 5s
    percentage:
      value: 100  # Все запросы задерживаются
```

- Имитация медленного ответа **всего сервиса**.
- Тестирование **Circuit Breaker** (чтобы убедиться, что трафик не идет на сломанный сервис).

### Abort Injection

#### Генерация HTTP 500 для 50% запросов

```yaml
fault:
  abort:
    httpStatus: 500
    percentage:
      value: 50
```

- Тестирование, как **frontend** реагирует на ошибки бекенда.
- Проверка, срабатывают ли **ретраи** в DestinationRule.

#### Возвращение разных HTTP-ошибок

```yaml
fault:
  abort:
    httpStatus: 403  # Доступ запрещен
    percentage:
      value: 30  # 30% запросов получат 403
```

- Тестирование поведения API при **403 (Forbidden)**, **404 (Not Found)**, **503 (Service Unavailable)**.
- Проверка, как клиентские сервисы **обрабатывают разные типы ошибок**.

#### Полное отключение сервиса (100% отказ)

```yaml
fault:
  abort:
    httpStatus: 503  # Service Unavailable
    percentage:
      value: 100  # Все запросы будут получать 503
```

- Симуляция полного падения сервиса.
- Тестирование **автоматического переключения** на другой сервис (если есть резервный).

#### Комбинированный Fault Injection

```yaml
fault:
  delay:
    fixedDelay: 2s
    percentage:
      value: 50  # 50% запросов задерживаются
  abort:
    httpStatus: 500
    percentage:
      value: 20  # 20% запросов сразу получают 500
```

- Проверка, **как клиенты обрабатывают смешанные проблемы** (медленный сервис + редкие ошибки).
- Имитация **нагрузки на базу данных** (когда часть запросов проходит, а часть получает отказ).

#### Fault Injection с JWT (тестирование авторизации)

```yaml
http:
  - match:
      - headers:
          authorization:
            prefix: "Bearer bad-token"
    fault:
      abort:
        httpStatus: 401  # Unauthorized
        percentage:
          value: 100
```

- Тестирование **авторизации API**.
- Проверка, как frontend обрабатывает невалидные JWT-токены.

#### Fault Injection в зависимости от User-Agent

```yaml
http:
  - match:
      - headers:
          user-agent:
            regex: ".*(iPhone|Android).*"
    fault:
      abort:
        httpStatus: 503
        percentage:
          value: 100
```

- Тестирование отказов **только для мобильных приложений**.
- Проверка, как **разные клиенты** (Web/Mobile) реагируют на сбои.
