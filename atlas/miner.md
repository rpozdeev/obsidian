---
id: miner
aliases: []
tags:
  - atlas
  - miner
---

Приложение подписывается на события kubernetes и сохраняет данные о Deployment, StatefulSet, DaemonSet в базу дынных.

Данные:

- id
- name
- namespace
- replicas
- created_at
- labels
- pods
  - name
  - node
  - containers
    - name
    - image
    - limits
      - cpu
      - memory
    - requests
      - cpu
      - memory
- status

Эти данные необходимы для расчета стоимости приложения

- [ ] добавить в расчет стоимость pv

