список всех образов
```bash
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec['initContainers', 'containers'][*].image}" |\ tr -s '[[:space:]]' '\n' |\ sort |\ uniq -c
```
