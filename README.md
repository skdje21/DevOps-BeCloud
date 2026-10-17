# DevOps test Project

Небольшой проект из 4 сервисов, который разворачивается одной командой через Ansible на любом Linux-сервере (физическом или в ВМ). Nginx выступает reverse proxy, фронт — статика, бэк — API на Node, данные хранятся в PostgreSQL.

## Что внутри

```
Клиент → nginx:8080 → ┬─ frontend (static HTML)
                      └─ backend (/api/*) → PostgreSQL

На хосте: UFW, SSH hardening, fail2ban, unattended-upgrades
```

| Сервис | Образ | Что делает |
|--------|-------|-----------|
| nginx | nginx:alpine | Reverse proxy, единственный открытый порт (8080) |
| frontend | nginx:alpine | Отдаёт index.html с двумя кнопками |
| backend | node:18-alpine | Express API: `/api/health`, `/api/visits` |
| db | postgres:15-alpine | Таблица visits, доступна только внутри сети |

## Требования

На целевом сервере (где будет крутиться приложение):
- Ubuntu 22.04 LTS 
- SSH-доступ по ключу
- Пользователь с правами sudo

На машине, с которой запускаешь плейбук (control-машина):
- Ansible 2.14+
- Docker (опционально — ускоряет первый деплой, см. ниже)

Если своего Linux-сервера под рукой нет — поднимается в любой ВМ (UTM / VirtualBox / VMware / Parallels / что угодно). Инструкция ниже учитывает оба варианта.

## Запуск

Сначала клонируем репозиторий (на control-машине, с которой будешь запускать Ansible):

```bash
git clone https://github.com/skdje21/DevOps-BeCloud.git
cd DevOps-BeCloud
```

### Шаг 1 (только если нужна ВМ). Создать виртуалку

Если у тебя уже есть готовый Linux-сервер — пропускай, иди на шаг 3.

Параметры ВМ: Ubuntu Server 22.04, 2 CPU / 4 GB RAM / 20 GB диск, включить OpenSSH при установке. Пользователь на свой вкус — я использовал `devops`.

> Я тестировал на macOS Apple Silicon через UTM. На Windows/Linux удобнее VirtualBox или VMware.

### Шаг 2 (только если нужна ВМ). Узнать IP виртуалки

Внутри ВМ:
```bash
ip a
```
Запомнить адрес (у меня `192.168.64.2`).

### Шаг 3. Настроить SSH-доступ

С control-машины:
```bash
# если ключа ещё нет
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# закинуть ключ на сервер
ssh-copy-id devops@<IP-сервера>

# проверить что заходит без пароля
ssh devops@<IP-сервера>
```

### Шаг 4. Подправить инвентарь

Открой `ansible/inventory.ini` и укажи свой IP и имя пользователя:
```ini
[devops]
<IP-сервера> ansible_user=<user> ansible_become_password=<sudo-пароль>
```

### Шаг 5. Деплой

```bash
cd ansible
ansible-playbook -i inventory.ini playbook.yml
```

Что происходит под капотом:
1. Файлы проекта копируются на сервер (rsync)
2. Ставится Docker Engine + compose plugin
3. Образы загружаются (см. раздел «Про Docker Hub» ниже)
4. Собираются образы frontend/backend
5. Запускаются все 4 контейнера
6. Проверяется health-эндпоинт

В конце должно быть `failed=0`.

### Шаг 6. Безопасность

```bash
ansible-playbook -i inventory.ini security.yml
```

Применяет UFW, отключает root и парольный логин в SSH, ставит fail2ban и автообновления.

## Проверка что всё работает

Открыть в браузере: `http://<IP-сервера>:8080`

Там две кнопки:
- **Check API** — дёргает `/api/health`, покажет status и время с БД
- **Count Visit** — инкрементит счётчик в таблице `visits`

Или через curl:
```bash
curl http://<IP-сервера>:8080/api/health
# {"status":"ok","db_time":"2026-10-17T..."}

curl http://<IP-сервера>:8080/api/visits
# {"total_visits":"5"}
```

Посмотреть что в базе:
```bash
ssh devops@<IP-сервера>
sudo docker compose -f /home/devops/devops-test/docker-compose.yml exec db \
  psql -U app_user -d app_db -c "SELECT * FROM visits;"
```

Проверить безопасность:
```bash
sudo ufw status verbose
sudo fail2ban-client status sshd
sudo grep -E "PermitRootLogin|PasswordAuthentication|MaxAuthTries" /etc/ssh/sshd_config
```

## Про Docker Hub

У меня при деплое выяснилось, что из моей ВМ `docker pull` упирается в блокировку хаба (connection reset). Поэтому в плейбук заложил два пути загрузки образов:

1. **Если на control-машине есть Docker** — образы сохраняются через `docker save`, передаются архивом на сервер и грузятся `docker load`. Самый надёжный путь, не зависит от сети на целевой машине.
2. **Если Docker на control-машине нет** — плейбук пробует обычный `docker pull` прямо с сервера, с retry (3 попытки).

Логика переключения автоматическая — проверяется через `docker info` на control-машине. Оба пути проверены вручную, работают.

## Структура

```
devops-test/
├── docker-compose.yml           # 4 сервиса
├── frontend/
│   ├── Dockerfile
│   └── index.html
├── backend/
│   ├── Dockerfile
│   ├── app.js                   # Express API
│   └── package.json
├── nginx/
│   └── nginx.conf               # proxy_pass на frontend/backend
├── ansible/
│   ├── inventory.ini
│   ├── playbook.yml             # Docker + деплой
│   └── security.yml             # UFW / SSH / fail2ban
└── README.md
```

## Что настроено по безопасности

| Что | Как |
|-----|-----|
| UFW | deny incoming по умолчанию, открыты только 22 и 8080 |
| SSH | root-логин off, парольная аутентификация off, MaxAuthTries 3 |
| fail2ban | SSH jail: 3 неудачные попытки → бан на час |
| unattended-upgrades | включены для security-патчей |
| Docker | БД не публикует порты наружу, везде alpine-образы |

---
