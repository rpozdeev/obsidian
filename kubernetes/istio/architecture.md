# Архитектура интеграции Istio и Kubernetes

| **Компонент Istio**       | **Роль**                                                                               | **Интеграция с Kubernetes**                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **istiod**                | Управляет маршрутизацией, настройками безопасности и сертификатами.                    | Запускается как Deployment в istio-system, использует VirtualService, DestinationRule, Gateway. |
| **Envoy Proxy (Sidecar)** | Перехватывает трафик между сервисами, применяет политики безопасности и маршрутизации. | Встраивается в **каждый под**, перехватывает трафик через iptables.                             |
| **Ingress Gateway**       | Управляет входящим трафиком, заменяя стандартный Ingress.                              | Разворачивается как Service (LoadBalancer/NodePort).                                            |
| **Egress Gateway**        | Контролирует выходящий трафик из кластера.                                             | Ограничивает доступ сервисов к внешним ресурсам.                                                |

# Использование CRD для расширения Kubernetes

| **CRD Istio**       | **Описание**                            |
| ------------------- | --------------------------------------- |
| VirtualService      | Определяет маршрутизацию HTTP/TCP.      |
| DestinationRule     | Управляет балансировкой нагрузки, мTLS. |
| Gateway             | Настраивает входящий/исходящий трафик.  |
| AuthorizationPolicy | Определяет политики безопасности.       |
```shell
kubectl get crds  grep istio
```

# Инъекция Sidecar-прокси (Envoy)

Istio автоматически добавляет **sidecar-прокси (Envoy)** к каждому поду для контроля трафика.

**Как работает перехват трафика?**

- **iptables** перенаправляет весь трафик пода через Envoy
- **Envoy** применяет правила маршрутизации и политики безопасности
- **Istiod** управляет конфигурацией и мониторингом

```bash
kubectl label namespace default istio-injection=enabled
```

Проверка
```bash
kubectl get pods -n default -o jsonpath='{.items[*].spec.containers[*].name}'
```

# Управление входящим и исходящим трафиком
#ingressgateway #egressgateway

## Ingress Gateway (входящий трафик)
Istio заменяет стандартный Ingress, предоставляя более гибкие настройки.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "example.com"
```

## Egress Gateway (исходящий трафик)

Позволяет контролировать выходящий трафик из кластера. 
### Пример **ServiceEntry** для доступа к внешнему API:

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
