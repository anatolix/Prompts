# Self-hosted Vellum-ассистент с нуля — инструкция

По мотивам живого прогона 20–21 сентября 2026 (VM ai.anatolix.net, ассистент Juno).
Источник: chat-readable.md в этом же каталоге.

**Архитектура:** одна VM = один Vellum, без docker.
**Стек:** HostVDS KVM (Ubuntu 24.04) → bun + vellum CLI → systemd user unit → nginx + Let's Encrypt → публичный HTTPS → клиенты (web / iOS / Telegram) → OpenRouter как LLM → Vellum Platform для кредитных фич (поиск, генерация картинок).

Инструкция написана так, чтобы её мог выполнить и человек по шагам, и AI-агент:
каждый шаг — команды + проверка; все встреченные грабли описаны там, где встречаются,
и сведены в таблицу в конце.

**Соглашения:** `<имя>` — имя ассистента (у нас `juno`), `<host>.<домен>` — его публичный
хостнейм. Всё выполняется под пользователем `vellum` (NOPASSWD sudo), если не сказано иначе.

---

## Шаг 0. Что понадобится заранее

- Аккаунт HostVDS с openrc.sh (OpenStack-креды тенанта; `chmod 600`, внутри OS_PASSWORD).
- SSH-ключпара ed25519 (пабкик пойдёт в VM и в GitHub).
- Поддомен, который можно делегировать на IP VM.
- Ключ OpenRouter (LLM-трафик).
- Аккаунт Vellum Platform (для кредитных фич: поиск, картинки).
- Telegram-бот от @BotFather (если нужен Telegram).

---

## Шаг 1. VM на HostVDS

Регион `eu-north1` (Рига) — основная площадка. Flavor: **hostvds-16** (4c/16G/160G) —
проверен на Juno; минимально рабочий — hostvds-8 (2c/8G/80G) + swap.

```sh
source openrc.sh
export OS_REGION_NAME=eu-north1
openstack keypair create --public-key ~/.ssh/id_ed25519.pub <имя>
openstack server create \
  --flavor 84651434-65f0-4e4f-867c-181a80d9dc43 \   # hostvds-16
  --image 33baaf7f-11fe-49e4-b866-4f82c7c1f35b \    # Ubuntu-24.04
  --network <Internet-NN-id> \                       # любая Internet-*, НЕ RESERVE-*/NOTWORKING-*
  --key-name <имя> \
  --user-data cloud-init.yaml \
  <vm-name>
```

**cloud-init.yaml:** пользователь `vellum` (твой публичный ключ, `sudo: ALL=(ALL) NOPASSWD:ALL`,
`lock_passwd: true`), root выключен, `ssh_pwauth: false`, package_upgrade, ufw 22/80/443,
fail2ban, swap 8G.

### ⚠ Грабли 1: security group — VM поднимется мёртвой

Стоковая OpenStack-группа `default` пускает только трафик своих членов. VM будет ACTIVE,
cloud-init отработает, а SSH снаружи — таймаут. **Сразу после создания, до первого SSH:**

```sh
openstack security group rule create --proto tcp --dst-port 22  default
openstack security group rule create --proto tcp --dst-port 80  default
openstack security group rule create --proto tcp --dst-port 443 default
openstack security group rule create --proto icmp default
```

Диагностика: `openstack console log show <vm>` — если cloud-init завершился, а снаружи
таймаут, это security group, а не VM.

**Проверка:** `ssh vellum@<ip>` проходит.

---

## Шаг 2. Установка Vellum

```sh
# bun строго v1.3.11 — версия привязана к платформе 0.12.2
curl -fsSL https://bun.sh/install | bash -s bun-v1.3.11
export PATH=$HOME/.bun/bin:$PATH   # и в ~/.bashrc
bun install -g vellum@0.12.2
loginctl enable-linger vellum
vellum hatch --name <имя> --remote local -d
```

Гейтвей и qdrant слушают только 127.0.0.1 (7830, 6333) — так и должно быть.

Systemd user unit `~/.config/systemd/user/vellum-<имя>.service`:

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=%h/.bun/bin/vellum wake <имя>
ExecStop=%h/.bun/bin/vellum sleep <имя>
TimeoutStartSec=300
Environment=PATH=%h/.bun/bin:/usr/local/sbin:/usr/sbin:/usr/local/bin:/usr/bin:/bin
```

### ⚠ Грабли 2: PATH юнита обязан содержать /usr/sbin

Без него vellum не находит системный nginx, сообщает «nginx is not installed»
(вводит в заблуждение — nginx установлен) и публичный edge не поднимается.

```sh
systemctl --user enable --now vellum-<имя>.service
```

**Проверка:** reboot → ассистент возвращается сам (~1 мин), `assistant status` зелёный.

---

## Шаг 3. Домен и публичный HTTPS

1. DNS: A-запись `<host>.<домен>` → IP VM. Дождаться резолва.
2. `assistant config set ingress.publicBaseUrl https://<host>.<домен>`
   — vellum поднимет свой nginx edge на 127.0.0.1:7840.
3. Системный nginx: 80→443 редирект; 443→127.0.0.1:7840 с websocket Upgrade map
   и `client_max_body_size 200m`.
4. `certbot --nginx -d <host>.<домен>` — сертификат Let's Encrypt.

**Проверка:** https://<host>.<домен> отдаёт веб-клиент.

---

## Шаг 4. LLM и память — критично, без этого фон мёртв

```sh
# OpenRouter-ключ — через секретную форму, не в чат:
assistant credentials prompt   # service: openrouter, field: api_key

# БЕЗ ЭТОГО все фоновые LLM-вызовы (память, ретроспективы) дедlock'ятся:
assistant config set llm.defaultProvider.provider openrouter

# Локальные бесплатные эмбеддинги (ONNX, мультиязычные, 1024-dim):
assistant config set memory.embeddings.provider local
assistant config set memory.embeddings.localModel Xenova/multilingual-e5-large
assistant config set memory.qdrant.vectorSize 1024
```

Рекомендуется дешёвый профиль для фоновой памяти: `llm.profiles.bg-cheap` =
`openrouter/deepseek/deepseek-v4-flash` (effort none, thinking off, maxTokens 8192),
приколоченный к call-sites: memoryV3SelectL2, recall, memoryRouter, memoryConsolidation,
filingAgent, memoryExtraction. Дорогая модель остаётся только на разговоре.

**Проверка:** `assistant config get llm` — defaultProvider = openrouter.

---

## Шаг 5. Клиенты

### Web
Уже работает после шага 3.

### iOS / desktop pairing
```sh
sudo -u vellum env HOME=/home/vellum PATH=/home/vellum/.bun/bin:$PATH \
  vellum pair <имя> --url https://<host>.<домен>
```
⚠ systemd user unit PATH не содержит bun — без явного `env PATH=...` бинарь не найдётся.
Ссылка живёт 10 минут.

### Telegram-бот
Порядок важен. **Никогда не дёргать setWebhook руками** — он гоняется с reconcileTelegramWebhook.

1. `assistant credentials prompt --service telegram --field bot_token` — секретная форма.
2. `assistant credentials set --service telegram --field webhook_secret "$(cat /proc/sys/kernel/random/uuid)" --generated`
3. `assistant config set ingress.publicBaseUrl https://<host>.<домен>` — этот config set
   триггерит reconcile, который сам вызовет setWebhook.
4. Проверка: `assistant channels get telegram` → `ready: true, webhook_delivery: OK`.
5. Привязка личности: guardian-verify выдаёт deep-link в бота; подтверждение приходит
   через самого бота (пуллинга нет). Если код протух — `resend` на той же сессии.

### GitHub (опционально)
Пабкик ed25519 в GitHub; после каждой смены хоста:
`ssh-keyscan github.com >> ~/.ssh/known_hosts`.

---

## Шаг 6. Vellum Platform (кредиты: поиск, картинки)

Штатный путь содержит три бага. Рабочая процедура целиком:

### 6.1. Base URL
```sh
assistant config set platform.baseUrl https://platform.vellum.ai
```
⚠ Без этого логин уходит в `localhost:8000` — дефолт после установки.

### 6.2. Логин (loopback-copy трюк)

⚠ Сигнал `show_platform_login` не обрабатывает НИ ОДИН клиент — окно логина не всплывёт
никогда (проверено по исходнику `platform-routes.ts`). Единственный путь — CLI.

⚠ bash-инструмент ассистента убивает всё дерево процессов в конце вызова — даже
`setsid nohup` не спасает. Долгоживущий листенер логина запускать только через systemd:

```sh
sudo -u vellum env XDG_RUNTIME_DIR=/run/user/1000 systemd-run --user \
  --unit=vellum-login --collect \
  env PATH=/home/vellum/.bun/bin:/usr/local/bin:/usr/bin:/bin \
  /home/vellum/.bun/bin/vellum login --force
journalctl --user -u vellum-login --no-pager | tail -1   # забрать authorize URL
```

Листенер живёт ~5 минут. Порядок:
1. Открыть authorize URL в браузере (можно с телефона), залогиниться.
2. Браузер редиректит на `http://127.0.0.1:<port>/auth/callback?code=...&state=...` —
   страница НЕ откроется, это норма. Скопировать адрес целиком.
3. На VM: `curl "http://127.0.0.1:<port>/auth/callback?code=...&state=..."` → HTTP 200.
   Код одноразовый и привязан к текущему порту/листенеру.

### 6.3. ⚠ Грабли 3: инъекция ключей молча падает

После успешного логина CLI печатает «Some credentials could not be injected into the
assistant» — и `assistant platform status` остаётся пустым. Штатная инъекция сломана;
рабочее лечение — ручной reprovision + ручная инъекция:

```sh
PTOKEN=$(cat ~/.config/vellum/platform-token)
CLIENTID=$(cat ~/.config/vellum/client-id)
ORG=<organization-id>   # виден в платформенном UI / у существующего инстанса

# 1. Свежий API-ключ ассистента
curl -X POST https://platform.vellum.ai/v1/assistants/self-hosted-local/reprovision-api-key/ \
  -H "Content-Type: application/json" -H "X-Session-Token: $PTOKEN" \
  -H "Vellum-Organization-Id: $ORG" \
  -d "{\"client_installation_id\":\"$CLIENTID\",\"runtime_assistant_id\":\"<имя>\",\"client_platform\":\"cli\"}"
# → provisioning.assistant_api_key

# 2. ensure-registration → webhook_secret и АКТУАЛЬНЫЙ platform assistant id
#    (может отличаться от id, выданного при логине — использовать ЭТОТ)
curl -X POST https://platform.vellum.ai/v1/assistants/self-hosted-local/ensure-registration/ \
  -H "Content-Type: application/json" -H "X-Session-Token: $PTOKEN" \
  -H "Vellum-Organization-Id: $ORG" \
  -d "{\"client_installation_id\":\"$CLIENTID\",\"runtime_assistant_id\":\"<имя>\",\"client_platform\":\"cli\",\"public_ingress_url\":\"https://<host>.<домен>\"}"

# 3. Влить шесть секретов в гейтвей
GT=$(python3 -c "import json;print(json.load(open('$HOME/.config/vellum/assistants/<имя>/guardian-token.json'))['accessToken'])")
# для каждого из: vellum:assistant_api_key, vellum:platform_assistant_id,
#   vellum:platform_base_url (https://platform.vellum.ai), vellum:platform_organization_id,
#   vellum:platform_user_id, vellum:webhook_secret
curl -X POST http://127.0.0.1:7830/v1/secrets \
  -H "Content-Type: application/json" -H "Authorization: Bearer $GT" \
  -d '{"type":"credential","name":"<name>","value":"<value>"}'
```

**Проверка:** `assistant platform status` — API key set, org/user IDs заполнены,
callback registration available: yes. `assistant platform credits` показывает баланс.
Тестовый web_search проходит.

⚠ `Platform: false` в статусе — **норма** для self-hosted: флаг IS_PLATFORM означает
«управляется платформой», а не «подключён к ней».

---

## Шаг 7. Финал

- Full access ассистенту выдаёт человек из меню клиента.
- Промпты на аппрувы приходят в ТЕКУЩИЙ чат-уведомления инстанса, не в старые чаты.
- Мониторинг: `assistant ps`, `assistant status`, `assistant platform credits`.

---

## Таблица граблей

| # | Симптом | Причина | Лечение |
|---|---|---|---|
| 1 | VM ACTIVE, SSH таймаут | OpenStack default SG пускает только своих | Явные правила 22/80/443/icmp (шаг 1) |
| 2 | «nginx is not installed» | Нет /usr/sbin в PATH юнита | Environment=PATH с /usr/sbin (шаг 2) |
| 3 | Фоновые LLM-задачи висят | defaultProvider=vellum | config set → openrouter (шаг 4) |
| 4 | Окно логина платформы не всплывает | Сигнал никем не потребляется | CLI-логин + ручной callback (шаг 6.2) |
| 5 | Логин-процесс умирает | bash убивает дерево процессов | systemd-run --user (шаг 6.2) |
| 6 | «Some credentials could not be injected» | Баг инъекции в CLI | Ручной reprovision + POST /v1/secrets (шаг 6.3) |
| 7 | `vellum pair`: command not found | Нет bun в PATH | env PATH=... vellum pair (шаг 5) |
| 8 | Telegram настроен, но молчит | Нет webhook_secret / reconcile не запущен | Креды + config set ingress.publicBaseUrl (шаг 5) |
| 9 | ensure-registration даёт другой assistant id | Платформа перевыдала запись | Использовать id из ensure-registration (шаг 6.3) |
