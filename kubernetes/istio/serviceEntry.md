# Основная структура ServiceEntry
#serviceEntry

ServiceEntry – это ресурс API Istio, который **расширяет сервис-меш, добавляя в него внешние или нестандартные сервисы**. Он позволяет управлять трафиком к этим сервисам и применять политики Istio.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: my-service-entry
spec:
  hosts: []
  addresses: []
  location: MESH_EXTERNAL
  resolution: NONE
  ports: []
  endpoints: []
  exportTo: []
  workloadSelector: {}
```

| **Поле**         | **Тип**  | **Обязательное?** | **Описание**                                                                               |
| ---------------- | -------- | ----------------- | ------------------------------------------------------------------------------------------ |
| hosts            | []string | ✅ Да              | Список **доменных имен** сервиса (например, api.example.com).                              |
| addresses        | []string | ❌ Нет             | **IP-адреса**, которые Istio будет перехватывать (например, 192.168.1.100/32).             |
| location         | string   | ✅ Да              | Определяет, является ли сервис **внутренним (MESH_INTERNAL) или внешним (MESH_EXTERNAL)**. |
| resolution       | string   | ✅ Да              | Как Istio будет определять **IP-адрес сервиса**.                                           |
| ports            | []object | ✅ Да              | **Список портов**, которые использует сервис.                                              |
| endpoints        | []object | ❌ Нет             | Явно заданные **IP-адреса сервисов**.                                                      |
| exportTo         | []string | ❌ Нет             | Позволяет ограничить **экспорт** ServiceEntry в определенные namespaces.                   |
| workloadSelector | object   | ❌ Нет             | Позволяет привязать ServiceEntry **к конкретным workload’ам** внутри кластера.             |

# Описание полей структуры
## hosts: (Обязательное поле)

Определяет, под какими **доменными именами** сервис будет доступен в сетевом меше Istio.

```yaml
spec:
  hosts:
  - api.example.com
```

**❗Важно**: Если resolution: STATIC, то домен не резолвится в IP, а используется указанный endpoints:.

## addresses: (Опциональное поле)

Задает **список IP-адресов**, которые Istio будет **перехватывать**. Это полезно при resolution: STATIC.

```yaml
spec:
  addresses:
  - 192.168.1.100/32
```

Если его не указать, трафик может идти **мимо Istio**.

## location: (Обязательное поле)

Определяет, является ли сервис **внутренним или внешним**.

|**Значение**|**Что делает?**|
|---|---|
|MESH_INTERNAL|Сервис **находится внутри кластера** (например, нестандартный Service).|
|MESH_EXTERNAL|Сервис **находится за пределами кластера** (например, внешние API).|
```yaml
spec:
  location: MESH_EXTERNAL
```

## resolution: (Обязательное поле)

Определяет, **как Istio будет находить IP-адрес сервиса**.

| **Значение** | **Описание**                                       |
| ------------ | -------------------------------------------------- |
| NONE         | Istio не выполняет DNS-резолвинг.                  |
| DNS          | Istio использует DNS-резолвинг при каждом запросе. |
| STATIC       | IP-адреса сервиса **заданы явно** в endpoints:.    |
```yaml
spec:
  resolution: DNS
```

```yaml
spec:
  resolution: STATIC
  endpoints:
  - address: 192.168.1.100
```

## ports: (Обязательное поле)

Список **портов, через которые работает сервис**.

**HTTP и HTTPS**
```yaml
spec:
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
```

**TCP-сервис, например, PostgreSQL**
```yaml
spec:
  ports:
  - number: 5432
    name: tcp
    protocol: TCP
```

## endpoints: (Опциональное поле)

Явно указывает **IP-адреса конечных точек**.

```yaml
spec:
  endpoints:
  - address: 192.168.1.100
    ports:
      http: 80
  - address: 192.168.1.101
    ports:
      http: 80
```

## exportTo: (Опциональное поле)

Ограничивает **видимость** ServiceEntry в определенных namespaces.

|**Значение**|**Описание**|
|---|---|
|"*"|Доступен всем (по умолчанию).|
|"."|Доступен **только в текущем namespace**.|
|["ns-1", "ns-2"]|Доступен только в этих **namespace**.|
```yaml
spec:
  exportTo:
  - "istio-system"
```

## workloadSelector: (Опциональное поле)

Привязывает ServiceEntry **к конкретным подам**.

```yaml
spec:
  workloadSelector:
    labels:
      app: my-app
```

# Примеры

## общий пример

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: my-external-service
spec:
  hosts:
  - api.example.com
  addresses:
  - 192.168.1.100/32
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  endpoints:
  - address: 192.168.1.100
    ports:
      https: 443
  exportTo:
  - "default"
  workloadSelector:
    labels:
      app: my-app
```

## доступ к внешнему API

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.example.com # Внешний сервис
  location: MESH_EXTERNAL # Вне кластера Kubernetes
  ports:
  - number: 443
    name: https
    protocol: HTTPS
```

## для подключения к внешней базе **PostgreSQL (порт 5432)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-db
spec:
  hosts:
  - db.example.com
  location: MESH_EXTERNAL
  ports:
  - number: 5432
    name: tcp
    protocol: TCP
```

## resolution: STATIC

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-service-static
spec:
  addresses:
  - 192.168.1.100/32   # Фиксированный IP-адрес сервиса
  hosts:
  - my-external-service.local  # Виртуальный DNS-алиас
  location: MESH_EXTERNAL  # Указывает, что сервис за пределами кластера
  resolution: STATIC  # Используем фиксированный IP-адрес
  ports:
  - number: 8080
    name: http
    protocol: HTTP
  endpoints:
  - address: 192.168.1.100  # Назначаем IP-адрес сервису
    ports:
      http: 8080
```

## несколькими IP (балансировка нагрузки)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-service-static-multiple
spec:
  addresses:
  - 192.168.1.100/32
  - 192.168.1.101/32
  hosts:
  - my-external-service.local
  location: MESH_EXTERNAL
  resolution: STATIC
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  endpoints:
  - address: 192.168.1.100
    ports:
      https: 443
  - address: 192.168.1.101
    ports:
      https: 443
```

# Записки

addresses: в ServiceEntry **указывает на IP-адреса**, которые Istio **должно считать частью сервис-меша**. Этот параметр полезен для:

- **Оптимизации маршрутизации** – Istio сможет **перехватывать** трафик к этим IP напрямую.
- **Более строгого контроля трафика** – трафик к указанным IP можно **ограничить через правила безопасности** (AuthorizationPolicy).
- **Обхода конфликтов с другими сервисами** – если в кластере уже есть другой сервис с тем же доменным именем (hosts:), addresses: помогает явно задать IP.

addresses: обязателен в двух случаях:
- Если ServiceEntry используется с MESH_INTERNAL (т.е. **как внутренний сервис**).
- Если в ServiceEntry используется **STATIC-резолвинг** (resolution: STATIC).

Если не указать addresses:, Istio **будет считать сервис виртуальным и не сможет перехватывать трафик на уровне iptables.** Трафик к IP 192.168.1.100 **не будет автоматически перехватываться** Istio, и пакет может пойти напрямую **мимо service mesh**.