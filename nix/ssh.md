---
id: ssh
aliases:
  - ssh
tags:
  - ssh
  - nix
---

# Конфигурация ssh

## Генерация ключа

```bash
ssh-keygen -t ed25519 -f ~/.ssh/new_key -C “new key”
```

## Скопировать ключ на сервер

Автоматически

```bash
ssh-copy-id  admin@server
```

Вручную

```bash
mkdir .ssh
touch .ssh/authorized_keys
cat .ssh/new_key > .ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
chmod 700 ~/.ssh
chmod 644 ~/.ssh/*
chmod 600 ~/.ssh/id_* ~/.ssh/authorized_keys
```

Безопасная конфигурация sshd
`/etc/ssh/sshd_config`

```ini
# Основные настройки
Port 2222                        # Используйте нестандартный порт для снижения риска атак
Protocol 2                       # Применяем только SSH протокол версии 2
PermitRootLogin no               # Запрещаем вход для root
MaxAuthTries 3                   # Ограничиваем количество попыток входа
MaxSessions 2                    # Ограничиваем количество сессий на одного пользователя

# Разрешаем вход только по ключам
PasswordAuthentication no        # Только аутентификация по ключам

# Запрещаем пустые пароли
PermitEmptyPasswords no

# Включаем аутентификацию по ключам
PubkeyAuthentication yes

# Отключаем аутентификацию по имени хоста
HostbasedAuthentication no

# Отключаем Challenge-Response аутентификацию
ChallengeResponseAuthentication no

# Разрешаем только определённых пользователей (замените "youruser" на нужное имя пользователя)
AllowUsers youruser
# AllowGroups sshusers           # Можно использовать группы, если требуется

# Настройка алгоритмов шифрования и обмена ключами
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com
MACs hmac-sha2-512,hmac-sha2-256

# Уровень логирования для аудита
LogLevel VERBOSE                 # Включаем детализированное логирование

# Ограничиваем перенаправление TCP для предотвращения нежелательных туннелей
AllowTcpForwarding no            # Запрещаем TCP форвардинг
X11Forwarding no                 # Запрещаем X11 форвардинг

# Отключаем перенаправление агента для безопасности
AllowAgentForwarding no            # Запрещаем перенаправление SSH-агента


# Таймауты для снижения риска атак
ClientAliveInterval 300          # Интервал проверки активности клиента (сек.)
ClientAliveCountMax 2            # Максимальное число проверок активности клиента
LoginGraceTime 30s               # Время на авторизацию до разрыва подключения

# Защита от атак методом перенаправления DNS
UseDNS no                        # Не используем DNS для проверки клиента

# Принудительное использование современных методов шифрования
IgnoreRhosts yes
```

## Конфигурация ssh (клиент)

`~/.ssh/config`

```ini
# Общие параметры для всех подключений
Host *
    ForwardAgent no               # Отключаем перенаправление агента
    ForwardX11 no                 # Отключаем перенаправление X11
    AddKeysToAgent yes            # Автоматически добавляем ключи в агент
    UseKeychain yes               # Используем связку ключей на macOS
    ServerAliveInterval 60        # Интервал для отправки keepalive-запросов
    ServerAliveCountMax 3         # Максимальное количество keepalive-запросов до разрыва соединения
    Compression yes               # Включаем сжатие для экономии трафика
    ControlMaster auto            # Включаем мультиплексирование соединений для ускорения
    ControlPath ~/.ssh/control_%h_%p_%r
    ControlPersist 10m            # Сохраняем соединение открытым на 10 минут

# Конкретный сервер с нестандартным портом и указанием конкретного ключа
Host myserver
    HostName 192.168.1.10         # IP-адрес или доменное имя сервера
    User youruser                 # Имя пользователя для подключения
    Port 2222                     # Нестандартный порт для подключения
    IdentityFile ~/.ssh/mykey     # Используем конкретный ключ для этого подключения

# Группа серверов с общими настройками
Host workservers
    HostName work.example.com
    User workuser
    Port 2200
    IdentityFile ~/.ssh/work_rsa  # Ключ для рабочих серверов
    ProxyJump bastion.example.com # Подключение через сервер-бастион

# Пример шаблона для определённого домена
Host *.example.com
    User commonuser
    IdentityFile ~/.ssh/example_rsa
    StrictHostKeyChecking ask     # Спрашивать подтверждение для новых хостов
```

## SSH-Agent

```bash
# запуск ssh-agent
eval "$(ssh-agent -s)"

# добавление ключа в ssh-agent:
ssh-add ~/.ssh/id_rsa

# проверка загруженных ключей:
ssh-add -l

# удаление ключей из агента:
ssh-add -d ~/.ssh/id_rsa     # Удаляет указанный ключ
ssh-add -D                    # Удаляет все ключи из памяти агента`
```
