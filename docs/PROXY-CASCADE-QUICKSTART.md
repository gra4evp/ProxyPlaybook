# Proxy Cascade Quickstart

Пошаговый playbook по настройке прокси и каскадного VPN. Документ обновляется по мере прохождения шагов.

## Инфраструктура

| Узел | Провайдер | IP | Статус |
|------|-----------|-----|--------|
| Нидерланды (NL) | hip.hosting | `138.124.2.114` | UFW + 3x-ui, мост для RU |
| Россия (RU) | adminvps.ru | `5.35.127.68` | UFW + 3x-ui, каскад RU→NL работает |
| Финляндия (FI) | hip.hosting, Хельсинки | — | Из РФ недоступен (консоль и SSH) |

Целевая схема (**работает**):

```
Клиент (РФ) → RU:443 Reality (admin-ru)
                 ├─ .ru / geoip:RU → direct (RU IP)
                 └─ остальное → NL:443 (admin-nl) → direct → интернет (NL IP)
```

Домен для TLS отложен — см. [вариант B (с доменом)](#вариант-b-с-доменом--vless--tls--xhttp-отложено). Сейчас: **Reality без домена** на RU.

---

## Шаг 1. Firewall (UFW)

### Что такое firewall

**Firewall (межсетевой экран)** — правила на сервере, которые решают, какой сетевой трафик **пропускать**, а какой **блокировать**.

Без firewall VPS с публичным IP постоянно сканируют боты: перебор паролей SSH, проверка открытых портов, попытки эксплуатации уязвимостей. Firewall не заменяет сильные пароли и SSH-ключи, но **отсекает лишний шум** и закрывает все порты, которые ты явно не открыл.

### UFW на Ubuntu/Debian

**UFW** (Uncomplicated Firewall) — простая обёртка над `iptables`. Политики по умолчанию после включения:

| Направление | Политика | Смысл |
|-------------|----------|--------|
| Incoming (входящие) | **deny** | Снаружи нельзя подключиться, если нет правила ALLOW |
| Outgoing (исходящие) | **allow** | Сервер сам может ходить в интернет (apt, curl и т.д.) |

### Зачем включаем

1. Закрыть все входящие порты, кроме нужных (сначала только SSH).
2. Снизить нагрузку от ботов и автоматических атак.
3. Перед установкой 3x-ui явно контролировать, какие порты открыты (панель, inbound прокси).

### Правильный порядок команд

> **Важно:** сначала разрешить SSH, **потом** включить firewall. Иначе можно потерять доступ к серверу.

```bash
# 1. Проверить текущее состояние
ufw status

# 2. Разрешить SSH (порт 22) — до включения!
ufw allow OpenSSH

# 3. Включить firewall
ufw --force enable

# 4. Проверить правила
ufw status verbose
```

`OpenSSH` — готовый профиль UFW; эквивалент `ufw allow 22/tcp`.

#### Почему `--force`

При `ufw enable` без флага скрипт спрашивает `Proceed with operation (y|n)?`. В SSH-сессии с **русской раскладкой** вместо латинской `y` может попасть кириллический символ — UFW падает с ошибкой:

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xd0 in position 0
```

`ufw --force enable` включает firewall без интерактивного подтверждения. Альтернатива: переключить раскладку на English и ввести `y`.

### Открытие портов 80 и 443

Перед установкой 3x-ui и настройкой прокси с TLS нужно разрешить стандартные веб-порты:

| Порт | Зачем |
|------|--------|
| **80/tcp** | HTTP — выпуск сертификата Let's Encrypt (ACME challenge), редирект на HTTPS |
| **443/tcp** | HTTPS — основной порт для прокси с TLS (VLESS/VMess + WebSocket и т.д.) |

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw status verbose
```

Открываем **до** установки 3x-ui, чтобы после настройки inbound не забыть про firewall.

### Как понять, на каком сервере ты работаешь

Prompt `root@235714` и hostname в логах (`235714.com`) **не показывают** страну или провайдера — это внутреннее имя хостера.

Надёжная проверка — **публичный IP**:

```bash
curl -4 ifconfig.me
```

| IP | Сервер |
|----|--------|
| `5.35.127.68` | RU (adminvps) |
| `138.124.2.114` | NL (hip.hosting) |

При установке 3x-ui IP также виден в логе Let's Encrypt и в URL панели.

### Текущий результат UFW (RU, `5.35.127.68`)

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

**Что это значит:**

- `Status: active` — firewall включён.
- `Default: deny (incoming)` — всё входящее заблокировано, кроме явных правил.
- `22/tcp (OpenSSH) ALLOW IN` — SSH доступен с любого IP (IPv4 и IPv6).
- `80/tcp`, `443/tcp` — HTTP/HTTPS открыты для сертификатов и прокси-трафика.
- `(v6)` — те же правила для IPv6.
- `disabled (routed)` — форвардинг между интерфейсами не фильтруется UFW (понадобится позже для каскада, настраивается отдельно).

### Порт панели 3x-ui в UFW *(легко забыть — в гайдах часто не говорят)*

Установщик 3x-ui задаёт **случайный порт панели** (у нас на RU: `50593`). Пока он не открыт в UFW, браузер **не достучится** до панели, хотя сервис работает.

```bash
# узнать порт: x-ui settings
ufw allow 50593/tcp
ufw status verbose
```

Порт панели **не** равен 443 — это отдельный HTTPS-порт для админки.

### Опционально: отключить ping (ICMP) *(отложено)*

Гайды иногда предлагают в `/etc/ufw/before.rules` заменить `ACCEPT` на `DROP` для ICMP (в т.ч. `echo-request`). Эффект: сервер **не пингуется** снаружи.

- **Плюс:** чуть меньше «шума» в простых сканах.
- **Минус:** не скрывает VPN; агрессивный DROP всего ICMP может мешать PMTU.
- **Компромисс:** DROP только `echo-request`, остальное оставить ACCEPT.
- **Приоритет:** низкий, можно после рабочего прокси.

```bash
nano /etc/ufw/before.rules   # секция icmp
ufw reload
```

### Чеклист UFW

- [x] SSH (`OpenSSH`) разрешён до `ufw enable`
- [x] `ufw status` показывает `Status: active`
- [x] Порты `80/tcp` и `443/tcp` открыты
- [x] Порт панели 3x-ui открыт (`50593/tcp` на RU)
- [ ] То же на NL-сервере (`138.124.2.114`), когда дойдёшь до него

---

## Шаг 2. Вход по SSH-ключам *(отложено)*

> **Статус:** пропущено на старте — сначала настраиваем прокси. Вернуться к этому шагу на всех серверах (NL, RU), когда базовый VPN работает.

### Зачем

Сейчас вход по **паролю root** — боты постоянно пробуют подобрать его. UFW закрывает лишние порты, но **порт 22 открыт для всех**. SSH-ключ:

- практически **не брутфорсят** (в отличие от пароля);
- удобнее — не вводить пароль при каждом `ssh`;
- позволяет **отключить вход по паролю** — главная защита от перебора.

### 2.1. Сгенерировать ключ на локальной машине (Windows)

В PowerShell или Git Bash **на своём ПК** (не на сервере):

```powershell
ssh-keygen -t ed25519 -C "proxy-playbook" -f "$env:USERPROFILE\.ssh\id_ed25519_proxy"
```

- На вопрос passphrase — можно Enter (пусто) или задать фразу (безопаснее).
- Появятся два файла:
  - `id_ed25519_proxy` — **приватный** ключ (никому не отдавать, не коммитить в git);
  - `id_ed25519_proxy.pub` — **публичный** ключ (его кладём на сервер).

### 2.2. Скопировать публичный ключ на сервер

**Вариант A — `ssh-copy-id`** (если есть, например в Git Bash):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_proxy.pub root@138.124.2.114
```

**Вариант B — вручную** (PowerShell + уже открытая SSH-сессия):

На **локальной** машине — вывести публичный ключ и скопировать строку:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_proxy.pub"
```

На **сервере** — добавить ключ:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys   # вставить строку целиком, сохранить
chmod 600 ~/.ssh/authorized_keys
```

### 2.3. Проверить вход по ключу

**Не закрывая** текущую SSH-сессию, открыть **новое** окно терминала:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_proxy" root@138.124.2.114
```

Если зашло **без пароля** (или только с passphrase ключа) — ключ работает.

Опционально — запись в `~/.ssh/config` на Windows (`C:\Users\<имя>\.ssh\config`):

```
Host nl-proxy
    HostName 138.124.2.114
    User root
    IdentityFile ~/.ssh/id_ed25519_proxy
```

Тогда достаточно: `ssh nl-proxy`.

### 2.4. Отключить вход по паролю *(только после проверки ключа!)*

> **Важно:** делать в **открытой** сессии, где ключ уже проверен. Иначе можно потерять доступ.

```bash
nano /etc/ssh/sshd_config
```

Изменить / убедиться:

```
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin prohibit-password
```

Применить:

```bash
sshd -t && systemctl reload sshd
```

Снова проверить вход по ключу из **нового** терминала. Старую сессию закрывать только после успешной проверки.

### 2.5. Опционально: fail2ban

Банит IP после нескольких неудачных попыток SSH (полезно, пока пароль ещё включён):

```bash
apt update && apt install -y fail2ban
systemctl enable --now fail2ban
```

### Чеклист SSH-ключи

- [ ] Ключ сгенерирован на локальной машине
- [ ] Публичный ключ добавлен в `~/.ssh/authorized_keys` на NL
- [ ] Вход по ключу проверен из нового терминала
- [ ] `PasswordAuthentication no` в `sshd_config`, `sshd` перезагружен
- [ ] То же на RU-сервере (`5.35.127.68`)
- [ ] (опционально) fail2ban установлен

---

## Шаг 3. Установка 3x-ui (RU)

> **Статус:** выполнено на RU (`5.35.127.68`), v3.7.0, панель доступна.

### Команда установки

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Переустановка / повторный запуск после обрыва SSH — нормально: установщик подхватит уже заданные username, password и WebBasePath.

### Опрос установщика — что выбирать

| Вопрос | Ответ | Зачем |
|--------|--------|--------|
| Database | **1** SQLite | Достаточно для личного использования |
| Customize Panel Port? | **n** | Случайный порт безопаснее дефолтного |
| SSL certificate | **2** Let's Encrypt for **IP** | Домена пока нет |
| Correct public IPv4? | **y** | Подтвердить IP **этого** сервера (`5.35.127.68` для RU) |
| IPv6 | Enter (пусто) | Если нет IPv6 |
| ACME port | **80** (default) | Нужен открытый `80/tcp` в UFW |

**Не путать:** на вопрос «Choose an option» вводится только цифра **1–4**, IP вводится **на следующем шаге**.

Сертификат по IP: срок **~6 дней**, продление через **acme.sh** (cron), установщик настраивает сам.

### После установки — данные для входа

```bash
x-ui settings    # port, username, password, webBasePath
x-ui status      # сервис и Xray
```

Формат URL панели:

```
https://<IP>:<PORT>/<WebBasePath>/
```

Пример для RU (порт и path — свои, смотри `x-ui settings`):

```
https://5.35.127.68:50593/<WebBasePath>/
```

> **Безопасность:** не коммить в git реальные пароли и полный WebBasePath. Храни их локально или в `.env` (в `.gitignore`).

Сброс логина/пароля — через меню `x-ui` на сервере или пункт в установщике.

### «Ошибки» при установке — что игнорировать

| Сообщение | Это проблема? |
|-----------|----------------|
| `Failed to stop x-ui.service: Unit x-ui.service not loaded` | **Нет** — первый запуск, останавливать было нечего |
| `SyntaxWarning: invalid escape sequence` при fail2ban | **Нет** — предупреждения в тестах пакета apt |
| `WARNING: Running pip as the 'root' user` | **Нет** — стандартное предупреждение pip |
| `Fail2ban installed successfully!` / `Fail2ban setup complete` | Всё ок — fail2ban + IP Limit для 3x-ui настроены установщиком |

### Логи `x-ui status` — что значат

```bash
x-ui status
```

| Строка | Смысл |
|--------|--------|
| `Active: active (running)` | Сервис работает |
| `WARNING - XRAY: core: Xray ... started` | **Не ошибка** — 3x-ui логирует старт Xray уровнем WARNING |
| `Sub server running HTTPS on [::]:2096` | Сервер подписок; для subscription-ссылок позже: `ufw allow 2096/tcp` |
| `database is locked` | Конкуренция записей SQLite (статистика / IP Limit). Редко — норма; часто — `x-ui restart`, проверить `df -h` |

Прокси (inbound) создаётся **отдельно в панели** — установка даёт только админку и Xray без клиентских подключений.

### Язык панели — почему сразу русский

3x-ui **не** определяет язык по IP сервера или стране VPS. Браузер отправляет заголовок `Accept-Language` (язык Windows/Chrome). Если первым идёт `ru` — интерфейс на русском.

Сменить: настройки панели (шестерёнка) → English. Выбор сохраняется в localStorage браузера.

### Чеклист 3x-ui (RU)

- [x] Установлен 3x-ui v3.7.0
- [x] SSL Let's Encrypt для IP (`5.35.127.68`)
- [x] Порт панели открыт в UFW
- [x] Логин/пароль сброшены на свои
- [x] Вход в панель из браузера
- [x] Создать inbound (прокси для клиента)
- [x] Установить 3x-ui на NL (`138.124.2.114`)

---

Домен для TLS **отложен** на вариант B — сейчас идём по **варианту A (без домена, Reality)** на RU. Зарубежный NL — позже для каскада и доменного inbound.

---

## Шаг 4. Inbound — два подхода

Два способа поднять клиентский прокси в 3x-ui. Выбор зависит от того, есть ли **свой домен** и **на каком сервере** висит DNS.

| | **A. Без домена** | **B. С доменом** |
|---|-------------------|------------------|
| **Протокол** | VLESS + **Reality** | VLESS + **TLS** |
| **Транспорт** | TCP (обычно) | **XHTTP** |
| **DNS** | Не нужен для inbound | A-запись на **зарубежный** сервер (NL) |
| **Сертификат inbound** | Не нужен (маскировка под чужой сайт) | Let's Encrypt на **свой** домен |
| **SSL панели** | LE для IP (уже сделано на RU) | LE для домена |
| **Сервер** | RU сейчас (`5.35.127.68`) | NL (`138.124.2.114`) — по гайду |
| **Статус** | **Текущий путь** | Отложено (+ nginx fallback) |

```
Вариант A (сейчас):  Клиент → RU:443 Reality → интернет (позже каскад → NL)

Вариант B (позже):   DNS → NL → VLESS+TLS+XHTTP:443
                     + nginx fallback-заглушка на 443 (HTTP-сайт, «менее палевно»)
```

---

### Вариант A. Без домена — VLESS + Reality *(текущий)*

**Reality** не требует своего домена и сертификата. Трафик маскируется под **чужой** реальный HTTPS-сайт (dest / SNI): для DPI это похоже на обычное TLS-подключение к известному домену.

#### RealiTLScanner — поиск подходящих dest

Сканер из экосистемы XTLS проверяет соседние IP в подсети VPS и ищет хосты с TLS, подходящие под Reality.

**Репозиторий:** [XTLS/RealiTLScanner](https://github.com/XTLS/RealiTLScanner/releases)

**Установка (RU, v0.2.3):**

```bash
wget https://github.com/XTLS/RealiTLScanner/releases/download/v0.2.3/RealiTLScanner-linux-amd64
chmod +x RealiTLScanner-linux-amd64
```

**Запуск** — сканирует подсеть вокруг указанного IP:

```bash
./RealiTLScanner-linux-amd64 --addr 5.35.127.68
```

Остановка: `Ctrl+C` (режим infinite — сканирует непрерывно).

**Предупреждение `Cannot open Country.mmdb`** — не критично, геолокация в выводе будет `geo=N/A`.

**Как читать строку результата:**

```text
feasible=true  tls="TLS 1.3"  alpn=h2  cert-domain=github.com
```

| Поле | Что искать |
|------|------------|
| `feasible=true` | Подходит для Reality |
| `tls="TLS 1.3"` | Обязательно TLS 1.3 |
| `alpn=h2` | HTTP/2 — хорошо |
| `cert-domain=...` | Домен для поля **SNI / dest** в inbound |

**Примеры из скана RU-подсети adminvps** (соседи на том же хостинге, не наш сервер):

| cert-domain | Заметка |
|-------------|---------|
| `cdnjs.cloudflare.com` | CDN, часто хороший candidat |
| `github.com` | Крупный сайт, TLS 1.3 + h2 |
| `*.vk.com`, `*.ozon.ru` | Крупные RU-сервисы |
| `bierbach.ru`, `test.domiksvinok.ru` | Чужие VPS на той же подсети — **слабый** выбор |

> **Как выбирать dest:** не брать первый попавшийся `.ru` соседа. Лучше **крупный стабильный** сайт с TLS 1.3 и h2. Часто вручную задают `www.microsoft.com:443`, `dl.google.com:443` или берут из скана CDN/глобальные домены (`github.com`, `cdnjs.cloudflare.com`).

#### RU или NL — где запускать сканер?

| Где сканер | Что находит | Когда имеет смысл |
|------------|-------------|-------------------|
| **RU** (`5.35.127.68`) | Соседи adminvps — много случайных `.ru` сайтов на VPS | Можно посмотреть `feasible` цели в локальной подсети |
| **NL** (`138.124.2.114`) | Соседи зарубежного DC | **Предпочтительно**, когда inbound на NL или exit за рубежом |

**Вывод:** для **зарубежного exit / NL-inbound** сканer логичнее гонять **с NL-сервера** — dest ближе к «нормальному» зарубежному трафику. На RU скан всё равно полезен для обучения, но многие `cert-domain` — чужие мелкие сайты на shared hosting.

Reality-**dest не обязан** быть из скана соседей: это **любой** доступный с твоего VPS сайт с TLS 1.3. Сканер помогает **найти и проверить** candidat'ов, а не заменяет здравый смысл.

#### Inbound в 3x-ui (Reality) — черновик

> Детальная пошаговая настройка полей панели — допишем после выбора dest.

1. **Inbounds → Add inbound**
2. Protocol: **VLESS**, Port: **443** (уже открыт в UFW)
3. Security: **Reality**
4. **Dest (target):** `домен:443` выбранного сайта (напр. `www.microsoft.com:443`)
5. **SNI / Server Names:** тот же домен
6. **uTLS / Fingerprint:** `chrome` (типично)
7. Сгенерировать **Short ID**, **Private key** (панель делает сама)
8. Добавить клиента → QR / ссылка в v2rayN / v2rayNG

### Чеклист Reality (RU)

- [x] RealiTLScanner скачан и запущен на RU
- [x] Dest/SNI выбран (Reality + XHTTP на :443)
- [x] Inbound VLESS + Reality создан
- [x] Каскад RU → NL ([шаг 6](#шаг-6-каскад-ru--nl-рабочая-схема))

---

### Вариант B. С доменом — VLESS + TLS + XHTTP *(отложено)*

Путь из гайда для **зарубежного сервера**, когда появится домен.

#### Предварительные условия

1. Арендовать домен.
2. **DNS A-запись → IP зарубежного сервера (NL)**, не RU:
   ```
   vpn.example.com  →  138.124.2.114
   ```
3. На NL: 3x-ui + Let's Encrypt **для домена** (вариант 1 при установке).
4. UFW: `80/tcp`, `443/tcp` (для ACME и прокси).

> В гайде домен вешают именно на **зарубежный** VPS — клиент ходит на NL по имени, сертификат валидный, при смене IP меняется только DNS.

#### Inbound — параметры (черновик)

| Поле | Значение |
|------|----------|
| Protocol | VLESS |
| Security | TLS |
| Transport | **XHTTP** |
| Domain / SNI | свой домен (`vpn.example.com`) |
| Certificate | Let's Encrypt (из панели или acme.sh) |
| Port | 443 |

#### Fallback-заглушка через nginx *(отложено — распишешь позже)*

Идея из гайда: на **443** параллельно с прокси (или через маршрутизацию) отдавать **обычный HTTP/HTTPS-сайт-заглушку** через **nginx** — при прямой проверке IP сервер выглядит как простой веб-сервер, а не «голый» VPN.

```
TODO (позже):
- [ ] nginx на 443 (или split routing с Xray)
- [ ] простая статическая страница / редирект
- [ ] согласовать порты с inbound XHTTP в 3x-ui
- [ ] не светить панель 3x-ui на том же URL
```

### Чеклист доменный путь (NL)

- [ ] Домен куплен
- [ ] A-запись на NL (`138.124.2.114`)
- [ ] 3x-ui на NL с LE для домена
- [ ] Inbound VLESS + TLS + XHTTP
- [ ] nginx fallback-заглушка
- [ ] Клиент по домену, не по IP

---

## Шаг 6. Каскад RU → NL *(рабочая схема)*

> **Статус:** проверено, работает. Примеры конфигов: [`xray-ru-cascade.json`](examples/xray-ru-cascade.json), [`xray-nl-bridge.json`](examples/xray-nl-bridge.json) (секреты — плейсхолдеры).

### Inbound и Outbound — коротко

| Термин | Кто | Аналогия |
|--------|-----|----------|
| **Inbound** | Сервер **слушает**, сюда подключаются | Дверь: ждёшь гостей |
| **Outbound** | Сервер **сам идёт** на другой сервер (как клиент) | Ты идёшь в чужую дверь |

**Правило каскада:** outbound на сервере X содержит **ссылку клиента inbound на сервере Y**, куда X подключается.

```
Телефон ──► RU inbound (admin-ru)
RU outbound ──► NL inbound (admin-nl)    ← RU стучится на NL
NL direct ──► интернет
```

> **Не путать с частью гайдов:** «RU-ссылка → outbound на NL» — это связь **нод панели** (NL → RU), не направление user-трафика. Для exit за рубежом нужно **RU outbound → NL inbound**.

### Схема

```
┌─────────┐  Reality+XHTTP   ┌─────────┐  Reality+XHTTP   ┌─────────┐
│ Телефон │ ───────────────► │   RU    │ ───────────────► │   NL    │ ──► 🌍
│         │  admin-ru :443   │         │  admin-nl :443   │ direct  │
└─────────┘                  └────┬────┘                  └─────────┘
                                  │
                         .ru / geoip:RU
                                  ▼
                               direct (RU IP, split)
```

### NL (`138.124.2.114`) — мост

| Что | Настройка |
|-----|-----------|
| **Inbound** | VLESS + Reality + XHTTP, порт **443** |
| **Клиент** | `admin-nl` — **только для RU**, не для телефона |
| **Reality dest** | `www.cloudflare.com:443` |
| **Outbound** | только **`direct`** — в интернет с NL IP |
| **Не нужно** | outbound на RU |

UFW (желательно ограничить мост):

```bash
ufw allow from 5.35.127.68 to any port 443 proto tcp
```

### RU (`5.35.127.68`) — вход + маршрутизация

| Что | Настройка |
|-----|-----------|
| **Inbound** | VLESS + Reality + XHTTP, порт **443** |
| **Клиент** | `admin-ru` → **ссылка в телефон / v2rayN** |
| **Reality dest** | свой dest (у нас `digitaltechnologies.top:443`) |
| **Outbound** | вставить **ссылку клиента `admin-nl` с NL** |
| **Routing** | `.ru` / `geoip:RU` → `direct`; остальной tcp,udp → outbound на NL |

Критично для моста RU → NL:

| Параметр | Должно совпадать |
|----------|------------------|
| UUID | RU outbound = клиент `admin-nl` на NL inbound |
| `serverName` | один из `serverNames` NL inbound (`www.cloudflare.com`) |
| `shortId` | один из `shortIds` NL inbound (`66eaa91a55`) |
| `publicKey` | пара к `privateKey` Reality на NL inbound |
| `path` / XHTTP | одинаково на обеих сторонах (`/`) |

### Routing на RU (split)

Российские сайты — напрямую (быстрее, RU IP). Остальное — через NL.

```json
{ "domain": ["regexp:.*\\.ru$", "..."], "outboundTag": "direct" }
{ "ip": ["ext:geoip_RU.dat:ru"], "outboundTag": "direct" }
{ "network": "tcp,udp", "outboundTag": "inbound-nl1-admin-nl" }
```

Порядок правил важен: **сначала** исключения (`.ru` → direct), **потом** общее правило на NL.

### Пошагово в панели 3x-ui

1. **NL:** Inbound → VLESS Reality XHTTP :443 → клиент `admin-nl` → скопировать **его** ссылку.
2. **RU:** Outbounds → Add → вставить ссылку `admin-nl`.
3. **RU:** Routing → `.ru` / geoip RU → direct; всё остальное → outbound на NL.
4. **RU:** Inbound → клиент `admin-ru` → ссылка **только в телефон**.
5. `x-ui restart` на обоих серверах.

### Проверка

| Тест | Ожидание |
|------|----------|
| `ifconfig.me` через прокси (google.com) | IP **NL** `138.124.2.114` |
| `ifconfig.me` на `.ru` сайте | IP **RU** `5.35.127.68` |
| Подключение клиента | только RU-ссылка (`admin-ru`) |

### Типичные ошибки

| Симптом | Причина |
|---------|---------|
| Везде RU IP | нет routing на NL outbound |
| Не коннектится мост | UUID / shortId / publicKey не совпадают |
| NL outbound на RU | обратное направление — для exit не нужно |
| RU-ссылка в NL outbound | связь нод, не user exit |

### Экспорт конфига для отладки

```bash
jq '{
  inbounds: [.inbounds[] | select(.protocol != "tunnel") | {tag, port, protocol, clients: [.settings.clients[]?.email]}],
  outbounds: [.outbounds[] | {tag, protocol, address: .settings.address, port: .settings.port}],
  routing: .routing.rules
}' /usr/local/x-ui/bin/config.json
```

Полный конфиг: `cat /usr/local/x-ui/bin/config.json` — **не коммитить** с ключами.

### Чеклист каскад

- [x] 3x-ui на RU и NL
- [x] NL inbound `admin-nl` :443
- [x] RU outbound → NL (ссылка `admin-nl`)
- [x] RU routing: .ru direct, остальное → NL
- [x] RU inbound `admin-ru` → телефон
- [x] `ifconfig.me` на google → NL IP
- [ ] SSH-ключи ([шаг 2](#шаг-2-вход-по-ssh-ключам-отложено))
- [ ] Доменный путь ([вариант B](#вариант-b-с-доменом--vless--tls--xhttp-отложено))

---

## Шаг 5. Клиент и проверка

1. Импорт **RU**-ссылки (`admin-ru`) в v2rayN / v2rayNG / Nekobox.
2. **Не** подключаться напрямую к NL `admin-nl` с телефона — это мост для RU.
3. Проверка split: зарубежный сайт → NL IP; `.ru` → RU IP.

---

## Следующие шаги

1. **SSH-ключи** — [шаг 2](#шаг-2-вход-по-ssh-ключам-отложено).
2. **Домен + VLESS TLS XHTTP** — [вариант B](#вариант-b-с-доменом--vless--tls--xhttp-отложено).
3. **FI-сервер** — когда будет доступен, третий hop.
4. **ICMP / ping** — опционально ([шаг 1](#опционально-отключить-ping-icmp-отложено)).
5. **host/path в XHTTP** — усилить маскировку ([шаг 4](#шаг-4-inbound--два-подхода)).

---

## Полезные команды

### UFW

```bash
ufw status numbered          # правила с номерами (для delete)
ufw allow 443/tcp            # HTTPS / прокси
ufw allow 50593/tcp           # порт панели 3x-ui (свой порт — x-ui settings)
ufw delete allow 443/tcp     # убрать правило
ufw disable                  # временно выключить (осторожно на prod)
```

### 3x-ui

```bash
x-ui              # меню управления
x-ui settings     # port, login, password, webBasePath
x-ui status       # systemd + последние логи
x-ui restart      # перезапуск (если database is locked)
x-ui log          # полный лог
x-ui banlog       # логи fail2ban / IP Limit
```

### RealiTLScanner

```bash
./RealiTLScanner-linux-amd64 --addr <IP-этого-сервера>   # скан подсети, Ctrl+C стоп
```

### Экспорт Xray config (без секретов)

```bash
jq '{
  inbounds: [.inbounds[] | select(.protocol != "tunnel") | {tag, port, protocol, clients: [.settings.clients[]?.email]}],
  outbounds: [.outbounds[] | {tag, protocol, address: .settings.address, port: .settings.port}],
  routing: .routing.rules
}' /usr/local/x-ui/bin/config.json
```

---

*Последнее обновление: шаг 6 — рабочий каскад RU→NL; примеры в `docs/examples/`.*
