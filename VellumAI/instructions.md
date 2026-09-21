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
  --flavor 45f32f0b-a79d-4c93-977c-57d2a8753d5e \   # hostvds-16 (актуально на сен 2026)
  --image 60fb7be9-02f1-48ee-ac23-fe36db7d69e2 \    # Ubuntu-24.04 (актуально на сен 2026)
  --network 008299cd-25ff-4cf2-aa9a-d6d8d605484a \  # Internet-06; НЕ RESERVE-*/NOTWORKING-*
  --key-name <имя> \
  --user-data cloud-init.yaml \
  --config-drive true \                             # ОБЯЗАТЕЛЬНО, см. грабли 2
  <vm-name>
```

⚠ **ID flavor/image протухают.** Перед созданием проверить актуальные:
`openstack flavor list` (искать hostvds-16) и `openstack image list` (искать Ubuntu-24.04).

### ⚠ Грабли 2: без --config-drive user-data не приклеивается

Без `--config-drive true` VM поднимается с `DataSourceNone` — cloud-init НЕ ВЫПОЛНЯЕТСЯ:
нет пользователя vellum, нет ключей, нет ufw. В `openstack server show` поле `user_data`
пустое. Лечения нет — VM пересоздавать с `--config-drive true`. IP при пересоздании
МЕНЯЕТСЯ — DNS делегировать только после финального создания.

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
openstack security group rule create --proto ipv6-icmp --ethertype IPv6 default
```

Правила идемпотентны — при повторе дают Conflict 409, это норма. Проверка:
`openstack security group rule list default --long`.

Диагностика: `openstack console log show <vm>` — если cloud-init завершился, а снаружи
таймаут, это security group, а не VM.

### ⚠ Грабли 3: MTU — SSH виснет на KEX

Сеть HostVDS между VM (и до части интернета) имеет реальный MTU ~1430 при eth0 mtu 1500,
а ICMP fragmentation-needed где-то глушится — PMTUD blackhole. Симптом: SSH вешается на
`expecting SSH2_MSG_KEX_ECDH_REPLY` — дефолтный KEX sntrup761 шлёт пакет ~1.2KB, который
дропается. Обход на раз: `ssh -o KexAlgorithms=curve25519-sha256` (32-байтные ключи
проходят). Постоянное лечение — MTU 1400 в netplan **на обеих VM сразу**:

```sh
sudo sed -i 's/mtu: 1500/mtu: 1400/' /etc/netplan/50-cloud-init.yaml
sudo netplan apply
ip link show eth0   # mtu 1400
```

После этого дефолтный KEX работает. Диагностика PMTUD: `ping -M do -s 1440 <ip>` —
если молчит при рабочем обычном пинге, это оно.

**Проверка шага 1:** `ssh vellum@<ip>` проходит.

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

### ⚠ Грабли 4: PATH юнита обязан содержать /usr/sbin

Без него vellum не находит системный nginx, сообщает «nginx is not installed»
(вводит в заблуждение — nginx установлен) и публичный edge не поднимается.

```sh
systemctl --user enable --now vellum-<имя>.service
```

### ⚠ Грабли 5: `assistant` CLI без VELLUM_WORKSPACE_DIR идёт не туда

CLI резолвит workspace только через переменную окружения `VELLUM_WORKSPACE_DIR`.
Без неё смотрит в `~/.vellum/workspace` и падает с
«Could not connect to the assistant at /home/vellum/.vellum/workspace/assistant.sock»
— даже если сидеть в каталоге workspace. В каждом шелле/скрипте:

```sh
export VELLUM_WORKSPACE_DIR=$HOME/.local/share/vellum/assistants/<имя>/.vellum/workspace
```

**Проверка:** reboot → ассистент возвращается сам (~1 мин), `assistant status` зелёный.

---

## Шаг 3. Домен и публичный HTTPS

1. DNS: A-запись `<host>.<домен>` → IP VM. Дождаться резолва.
2. ```sh
   assistant config set ingress.publicBaseUrl https://<host>.<домен>
   assistant config set ingress.enabled true
   systemctl --user restart vellum-<имя>.service
   ```
   ⚠ **Грабли 6:** edge на 127.0.0.1:7840 НЕ стартует только от publicBaseUrl —
   обязателен `ingress.enabled true` + рестарт демона. Без этого порт 7840 молчит,
   а журнал не пишет ничего внятного.
3. Системный nginx: 80→443 редирект; 443→127.0.0.1:7840 с websocket Upgrade map
   и `client_max_body_size 200m`.
4. `certbot --nginx -d <host>.<домен> --redirect` — сертификат Let's Encrypt.

**Проверка:** `ss -tln | grep 7840` слушает; https://<host>.<домен> редиректит на
`/assistant/`, и `https://<host>.<домен>/assistant/` отдаёт 200.

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
| 1 | VM ACTIVE, SSH таймаут | OpenStack default SG пускает только своих | Явные правила 22/80/443/icmp/ipv6-icmp (шаг 1) |
| 2 | Нет пользователя vellum, cloud-init не отработал | Нет --config-drive true | VM пересоздать с --config-drive (шаг 1) |
| 3 | SSH виснет на KEX_ECDH_REPLY | PMTUD blackhole, реальный MTU ~1430 | MTU 1400 в netplan; обход: KexAlgorithms=curve25519-sha256 (шаг 1) |
| 4 | «nginx is not installed» | Нет /usr/sbin в PATH юнита | Environment=PATH с /usr/sbin (шаг 2) |
| 5 | assistant CLI: «Could not connect... .vellum/workspace» | Нет VELLUM_WORKSPACE_DIR | export VELLUM_WORKSPACE_DIR=... (шаг 2) |
| 6 | Порт 7840 молчит после config set | Нет ingress.enabled true | config set ingress.enabled true + рестарт (шаг 3) |
| 7 | Фоновые LLM-задачи висят | defaultProvider=vellum | config set → openrouter (шаг 4) |
| 8 | Окно логина платформы не всплывает | Сигнал никем не потребляется | CLI-логин + ручной callback (шаг 6.2) |
| 9 | Логин-процесс умирает | bash убивает дерево процессов | systemd-run --user (шаг 6.2) |
| 10 | «Some credentials could not be injected» | Баг инъекции в CLI | Ручной reprovision + POST /v1/secrets (шаг 6.3) |
| 11 | `vellum pair`: command not found | Нет bun в PATH | env PATH=... vellum pair (шаг 5) |
| 12 | Telegram настроен, но молчит | Нет webhook_secret / reconcile не запущен | Креды + config set ingress.publicBaseUrl (шаг 5) |
| 13 | ensure-registration даёт другой assistant id | Платформа перевыдала запись | Использовать id из ensure-registration (шаг 6.3) |
| 14 | flavor/image ID не находится | ID протухли | openstack flavor/image list (шаг 1) |
