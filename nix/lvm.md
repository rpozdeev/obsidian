---
id: lvm
aliases: []
tags:
  - nix
  - lvm
---

# Logical Volume Manager

![[lvm.png]]

## Создание lvm

- Создание физического тома (Physical Volume, PV):

```bash
pvcreate /dev/sdX
```

- Создание группы томов (Volume Group, VG):

```bash
vgcreate имя_группы /dev/sdX
vgcreate my_vg /dev/sda1
```

- Создание логического тома (Logical Volume, LV):

```bash
lvcreate -L размер -n имя_логического_тома имя_группы
lvcreate -L 10G -n my_lv my_vg
```

## Управление LVM

- Список всех физических томов: `pvdisplay`
- Список всех групп томов: `vgdisplay`
- Список всех логических томов: `lvdisplay`

### Расширение логического тома

- Увеличение логического тома:

```bash
lvextend -L +размер имя_логического_тома
lvextend -L +5G /dev/my_vg/my_lv
```

- После расширения требуется обновить файловую систему:
  - Для ext4: `resize2fs /dev/my_vg/my_lv`
  - Для xfs: `xfs_growfs /mnt/точка_монтирования`

### Уменьшение логического тома

- Отмонтировать том:

```bash
umount /mnt/точка_монтирования
```

- Уменьшить файловую систему:

```bash
resize2fs /dev/my_vg/my_lv размер
resize2fs /dev/my_vg/my_lv 5G
```

- Уменьшить размер логического тома:

```bash
lvreduce -L размер /dev/my_vg/my_lv
```

### Удаление логического тома

- Отмонтировать том:

```bash
umount /mnt/точка_монтирования
```

- Удалить логический том:

```bash
lvremove /dev/my_vg/my_lv
```

### Удаление группы томов

```bash
vgremove имя_группы
```

### Удаление физического тома

- Убедитесь, что физический том не используется

```bash
vgreduce имя_группы /dev/sdX
```

- Удалите физический том:

```bash
pvremove /dev/sdX
```

## Дополнительные команды

- Просмотр структуры LVM:

```bash
lsblk
```

- Проверка статуса устройств:

```bash
pvs
vgs
lvs
```
