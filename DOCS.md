# Полное руководство по развертыванию MikroTik CHR, Traefik и Wildcard SSL в Docker

Настоящий документ представляет собой исчерпывающее руководство по архитектуре, развертыванию, конфигурации и аварийному восстановлению локальной сетевой инфраструктуры офиса на базе MikroTik CHR (Cloud Hosted Router) в Docker, обратного прокси-сервера Traefik (Coolify Proxy) и доверенных SSL-сертификатов Let's Encrypt Wildcard через Cloudflare DNS-01 Challenge.

Документ описывает весь путь создания системы, каждый конфигурационный файл, все выполненные терминальные команды, разбор возникших ошибок и регламенты эксплуатации.

---

## Содержание

1. Архитектура системы и исходные условия
2. Концепция Zero-IP Management и сквозная схема движения трафика
3. Подготовка операционной системы хоста и сетей Docker
4. Развертывание MikroTik CHR в Docker и фиксация MAC-адреса
5. Конфигурация MikroTik RouterOS (Адресация, NAT, DNS)
6. Выпуск Wildcard SSL Let's Encrypt через Cloudflare DNS API
7. Интеграция сертификатов в Traefik (Coolify Proxy) и разбор ошибок парсинга
8. Полная автоматизация автопродления сертификатов (Certbot Deploy-Hook)
9. Регламент добавления новых проектов
10. Книга устранения неисправностей и аварийного восстановления (Runbook)

---

## 1. Архитектура системы и исходные условия

### 1.1. Исходные условия и аппаратные ограничения
* **Главный GPON-роутер провайдера (`192.168.1.1`):** Оптический терминал офиса, раздающий адреса подсети `192.168.1.0/24`. Терминал закрыт провайдером: добавление пользовательских статических маршрутов (Static Routes) в прошивке невозможно.
* **Физический сервер Linux / ZimaOS (`192.168.1.199` / `192.168.1.200`):** Сервер с одним физическим сетевым интерфейсом (`eth0`), на котором запущен Docker Engine под управлением платформы Coolify.
* **Клиентские устройства (`192.168.1.0/24`):** Ноутбуки, рабочие станции и смартфоны сотрудников, подключенные по кабелю и Wi-Fi к офисной сети.
* **Доменная зона:** Принадлежащий организации домен `*.start-team.uz`, управляемый через DNS-панель Cloudflare.

### 1.2. Поставленные технические задачи
1. **Изолированный запуск RouterOS:** Развернуть MikroTik CHR внутри контейнера Docker с аппаратной виртуализацией KVM, выделив ему независимый сетевой адрес `192.168.1.201` в физической локальной сети.
2. **Обход ограничений GPON-роутера:** Обеспечить доступ со всех клиентских устройств ко всем внутренним проектам Docker без изменения настроек маршрутизации на провайдерском роутере.
3. **Принцип Zero-IP Management:** Исключить привязку внутренних проектов к статическим IP-адресам контейнеров. Маршрутизация должна осуществляться исключительно по именам/тегам сервисов на уровне L7 через обратный прокси-сервер Traefik.
4. **Валидный доверенный SSL:** Обеспечить открытие внутренних веб-интерфейсов по протоколу HTTPS с валидным доверенным сертификатом без предупреждений браузера ("Не защищено") и без необходимости установки локальных корневых сертификатов на устройства пользователей.
5. **Изоляция от публичного интернета:** Сервисы офиса не должны публиковаться наружу через проброс портов или Cloudflare Tunnels; трафик должен замыкаться строго внутри офиса.
6. **Аппаратная стабильность MAC-адресов:** Исключить создание фантомных сетевых интерфейсов RouterOS (`ether3`, `ether4`, `ether5`...) со сбросом конфигурации в `0.0.0.0` при обновлениях контейнера.

---

## 2. Концепция Zero-IP Management и сквозная схема движения трафика

### 2.1. Почему прямой роутинг по IP контейнеров непригоден
В стандартных Docker-сетях контейнеры динамически получают IP-адреса (например, `172.0.0.5`, `172.0.0.8`). При обновлении релиза, пересборке или перезапуске контейнера Docker не гарантирует сохранение прежнего IP-адреса. Прописывание маршрутов или правил NAT к конкретным адресам контейнеров приводит к падению всей инфраструктуры при первом же деплое.

Кроме того, L4 NAT (TCP/IP) не способен различать доменные имена: при обращении к одному IP-адресу на порт 443 роутер не знает, какой именно проект запрашивает браузер (`office...`, `crm...` или `api...`).

### 2.2. Принцип работы Zero-IP через связку MikroTik + Traefik
1. В локальной сети офиса единственной точкой входа объявляется адрес MikroTik CHR — **`192.168.1.201`**.
2. Встроенный DNS-сервер MikroTik перехватывает все запросы к доменной зоне `.*\.start-team\.uz` и сообщает клиентам: «Этот адрес находится на `192.168.1.201`».
3. Любое клиентское устройство отправляет HTTPS-запрос (порт 443) напрямую на `192.168.1.201`.
4. Межсетевой экран MikroTik (Dst-NAT) пересылает входящие TCP-пакеты с порта 443 на единый IP обратного прокси-сервера Traefik в сети Docker (`172.0.0.3`).
5. Traefik (L7 Reverse Proxy) считывает SNI (Server Name Indication) и заголовок `Host` из HTTP-запроса, применяет универсальный Wildcard SSL-сертификат `*.start-team.uz` и передает запрос в целевой контейнер по его **внутреннему Docker-имени (тегу)**, полностью игнорируя динамические IP-адреса.

### 2.3. Сквозная схема движения сетевых пакетов

```text
[Клиент: Ноутбук / Смартфон (192.168.1.76)]
       |
       | 1. DNS-запрос (UDP 53): "Каков IP у office.start-team.uz?"
       v
[MikroTik DNS (192.168.1.201:53)]
       |
       | 2. DNS-ответ: "192.168.1.201" (правило: regexp=".*\.start-team\.uz")
       v
[Клиент отправляет HTTPS-пакет (TCP 443) на 192.168.1.201]
       |
       | 3. Вход на интерфейс ether1 (macvlan) MikroTik
       |    Срабатывает Dst-NAT: подмена IP назначения на 172.0.0.3:443
       |    Срабатывает Src-NAT (masquerade): подмена IP источника на 172.0.0.2
       v
[Интерфейс ether2 MikroTik (172.0.0.2) в сети start-network]
       |
       | 4. Доставка пакета по внутренней сети Docker
       v
[Coolify Proxy / Traefik (172.0.0.3:443)]
       |
       | 5. TLS Handshake: отдача доверенного сертификата *.start-team.uz
       | 6. Разбор заголовка: Host: office.start-team.uz
       | 7. Резолв целевого сервиса через Docker DNS: "ws-server"
       v
[Контейнер приложения: ws-server / st-frontend]
```

---

## 3. Подготовка операционной системы хоста и сетей Docker

### 3.1. Создание сетевого моста macvlan-shim
По соображениям безопасности ядро Linux изолирует родительский физический интерфейс (`eth0`) от дочерних виртуальных интерфейсов `macvlan`. Чтобы сам сервер имел прямой двусторонний доступ к MikroTik `192.168.1.201`, на сервере создается интерфейс `macvlan-shim`.

1. Создайте юнит-файл службы systemd:
```bash
cat << 'EOF' > /etc/systemd/system/macvlan-shim.service
[Unit]
Description=Macvlan Shim Network Interface
After=network.target network-online.target

[Service]
Type=oneshot
ExecStartPre=-/sbin/ip link del macvlan-shim
ExecStart=/sbin/ip link add macvlan-shim link eth0 type macvlan mode bridge
ExecStart=/sbin/ip addr add 192.168.1.199/32 dev macvlan-shim
ExecStart=/sbin/ip link set macvlan-shim up
ExecStart=/sbin/ip route add 192.168.1.200/29 dev macvlan-shim
ExecStop=/sbin/ip link del macvlan-shim
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

2. Примените и запустите службу:
```bash
systemctl daemon-reload
systemctl enable --now macvlan-shim
systemctl status macvlan-shim
```
Статус должен иметь значение `Active: active (exited)` с кодом `0/SUCCESS`.

### 3.2. Создание сетей Docker
В системе разворачиваются две изолированные сети:
1. `lan` — сеть драйвера `macvlan` для прямого вывода интерфейса MikroTik в физический проводной сегмент офиса.
2. `start-network` — мостовая сеть (`bridge`), объединяющая контейнеры проектов и обратный прокси Traefik.

Команды создания сетей:
```bash
# Сеть macvlan для физической связи с офисом
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 lan

# Внутренняя сеть bridge для микросервисов
docker network create -d bridge \
  --subnet=172.0.0.0/24 \
  start-network
```

---

## 4. Развертывание MikroTik CHR в Docker и фиксация MAC-адреса

### 4.1. Анализ проблемы сброса MAC-адресов и интерфейсов
При стандартном перезапуске или обновлении контейнера Docker назначает виртуальным интерфейсам новые случайные MAC-адреса. Когда RouterOS видит сетевую карту с новым MAC-адресом, она считает ее новой физической платой и добавляет в систему интерфейсы `ether3`, `ether4`, `ether5`... оставляя их без настроек с IP `0.0.0.0`, а настроенные ранее интерфейсы помечает как отключенные.

Кроме того, если в постоянном хранилище (`/routeros`) уже был создан диск виртуальной машины с прежним MAC-адресом, RouterOS считывает старую базу интерфейсов, игнорируя новые параметры.

### 4.2. Эталонный файл `docker-compose.yml`
Для устранения этой проблемы MAC-адрес жестко фиксируется на уровне сервиса, а том монтируется в изолированный каталог данных.

Файл `/data/coolify/services/mikrotik/docker-compose.yml`:
```yaml
name: mikrotik
networks:
  lan:
    external: true
  start-network:
    external: true

services:
  mikrotik:
    container_name: mikrotik-chr
    image: 'ghcr.io/zmseld/chr:7.24.5'
    restart: unless-stopped
    mac_address: be:c0:19:d9:3d:34
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/kvm:/dev/kvm
      - /dev/net/tun:/dev/net/tun
    environment:
      - RAM_SIZE=1024M
      - CPU_COUNT=2
    networks:
      lan:
        ipv4_address: 192.168.1.201
      start-network:
    volumes:
      - /data/coolify/services/mikrotik/data:/routeros
```

### 4.3. Очистка старого хранилища и первый запуск
Если ранее MikroTik запускался со случайным MAC-адресом, перед стартом необходимо очистить старый образ диска, чтобы RouterOS пересоздала базу с эталонным MAC:

```bash
cd /data/coolify/services/mikrotik
docker compose down
rm -rf /data/coolify/services/mikrotik/data/*
docker compose up -d
```

Проверка журнала запуска:
```bash
docker logs -f mikrotik-chr
```
В выводе должно присутствовать успешное создание tap-интерфейсов, запуск процесса QEMU KVM и поднятие `ether1` со статическим MAC-адресом `be:c0:19:d9:3d:34`.

---

## 5. Конфигурация MikroTik RouterOS (Адресация, NAT, DNS)

Все команды выполняются в терминале RouterOS через утилиту WinBox (доступна на mikrotik.com).

### 5.1. Сетевая адресация и разбор ловушки маски `/32`
При ручном добавлении адресов без указания маски (например, `add address=192.168.1.201 interface=ether1`) RouterOS автоматически выставляет маску `/32` (`192.168.1.201/32`). Маска `/32` означает изолированный хост без соседей по сети. Из-за этого роутер не формирует маршруты к локальной подсети и при попытке пинга соседей немедленно выдает ошибку `no route to host`.

Также категорически запрещено указывать нулевой адрес подсети (`192.168.1.0/24`) вместо адреса хоста, так как `.0` является идентификатором самой сети.

Выполните в New Terminal WinBox:
```routeros
/ip address
set [find interface=ether1] address=192.168.1.201/24
set [find interface=ether2] address=172.0.0.2/24
```

Проверьте корректность адресации:
```routeros
/ip address print
```
Вывод должен строго соответствовать:
```text
Columns: ADDRESS, NETWORK, INTERFACE, VRF
# ADDRESS           NETWORK      INTERFACE  VRF 
0 192.168.1.201/24  192.168.1.0  ether1     main
1 172.0.0.2/24      172.0.0.0    ether2     main
```

### 5.2. Проверка связности с Traefik
Проверьте доступность обратного прокси Traefik (`172.0.0.3`) из терминала MikroTik:
```routeros
/ping 172.0.0.3 count=3
```
Результат должен показывать `0% packet loss` со временем отклика менее 1 мс.

### 5.3. Настройка межсетевого экрана и трансляции адресов (NAT)
Для передачи трафика в Docker настраиваются два взаимосвязанных правила:
1. `dst-nat` — перенаправляет клиентский трафик, пришедший на IP MikroTik `192.168.1.201` по портам 80 и 443, на внутренний IP Traefik `172.0.0.3`.
2. `src-nat` (masquerade) — подменяет IP-адрес клиента на IP роутера в сети докера (`172.0.0.2`). Без этого правила Traefik будет отправлять обратные пакеты через шлюз Docker-моста (`172.0.0.1`), что вызовет асимметричный разрыв TCP-сессии (`ERR_CONNECTION_TIMED_OUT`).

Команды настройки NAT:
```routeros
/ip firewall nat
add chain=dstnat dst-address=192.168.1.201 protocol=tcp dst-port=80,443 action=dst-nat to-addresses=172.0.0.3
add chain=srcnat out-interface=ether2 action=masquerade
```

### 5.4. Настройка локального DNS-сервера (Устранение NXDOMAIN)
Если DNS-сервер пуст или на нем отключен параметр `allow-remote-requests`, устройства офиса не смогут разрешить доменное имя и браузер выдаст ошибку `DNS_PROBE_FINISHED_NXDOMAIN`.

Команды настройки DNS:
```routeros
# Разрешение ответов на клиентские DNS-запросы и указание апстримов
/ip dns set allow-remote-requests=yes servers=1.1.1.1,8.8.8.8

# Статическая запись для основного сервиса
/ip dns static add name=office.start-team.uz address=192.168.1.201 ttl=1d

# Wildcard-запись для автоматического перенаправления всех существующих и будущих поддоменов
/ip dns static add regexp=".*\\.start-team\\.uz" address=192.168.1.201 ttl=1d
```

Проверка записей DNS:
```routeros
/ip dns static print
```

---

## 6. Выпуск Wildcard SSL Let's Encrypt через Cloudflare DNS API

### 6.1. Почему используется DNS-01 Challenge
Стандартная проверка доменов через HTTP-01 Challenge требует открытия порта 80 из интернета напрямую к серверу. Так как офисный сервис является строго конфиденциальным, порты наружу закрыты, а GPON-терминал не имеет белого IP или статического проброса, единственным надежным решением является выпуск сертификата через DNS-01 Challenge:
* Certbot создает временную TXT-запись `_acme-challenge.start-team.uz` в панели Cloudflare через официальный REST API.
* Центр сертификации Let's Encrypt проверяет DNS-запись через глобальные серверы имен.
* Сертификат выпускается без единого открытого порта на сервере.

### 6.2. Создание токена Cloudflare и файла конфигурации
1. В личном кабинете Cloudflare перейдите в **My Profile** -> **API Tokens** -> **Create Token**.
2. Используйте шаблон **Edit zone DNS**:
   * Permissions: `Zone` -> `DNS` -> `Edit`.
   * Zone Resources: `Include` -> `Specific zone` -> `start-team.uz`.
3. Сохраните токен на сервере в закрытом конфигурационном файле:
```bash
cat << 'EOF' > /etc/letsencrypt/cloudflare.ini
dns_cloudflare_api_token = ВАШ_ТОКЕН_CLOUDFLARE_ЗДЕСЬ
EOF

chmod 600 /etc/letsencrypt/cloudflare.ini
```

### 6.3. Выпуск сертификата
Выполните выпуск Wildcard-сертификата (покрывающего как сам корень домена, так и любые поддомены любого уровня вложенности):
```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.start-team.uz" \
  -d "start-team.uz" \
  --register-unsafely-without-email \
  --agree-tos
```

Готовые файлы сохраняются в:
* `/etc/letsencrypt/live/start-team.uz/fullchain.pem` (публичный сертификат и цепочка промежуточных центров).
* `/etc/letsencrypt/live/start-team.uz/privkey.pem` (приватный ключ RSA).

---

## 7. Интеграция сертификатов в Traefik (Coolify Proxy) и разбор ошибок парсинга

### 7.1. Разбор архитектуры монтирования Traefik в Coolify
Контейнер `coolify-proxy` имеет точку монтирования:
* На хосте: `/data/coolify/proxy/`
* Внутри контейнера: `/traefik/`

Параметры запуска Traefik содержат флаги:
* `--providers.file.directory=/traefik/dynamic/`
* `--providers.file.watch=true`

Это означает, что любой YAML-файл, помещенный в директорию хоста `/data/coolify/proxy/dynamic/`, мгновенно считывается Traefik на лету без перезапуска контейнера.

### 7.2. Разбор и устранение ошибок парсинга

#### Ошибка 1: `field not found, node: keyFile`
* **Причина:** В Traefik v3 схема YAML строго типизирована. Если при оформлении списка `tls.certificates` допущена ошибка отступов или `keyFile` сдвинут на один пробел влево/вправо, парсер пытается декодировать параметр `keyFile` в родительский узел `TLSConfig`, где такого поля не существует.
* **Решение:** Соблюдение эталонной иерархии с двумя пробелами на уровень (приведено ниже).

#### Ошибка 2: `failed to find any PEM data in certificate input`
* **Причина:** Файл `certs.yaml` ссылается на `/traefik/dynamic/fullchain.pem`, но физический файл в папке отсутствует или имеет нулевой размер.
* **Решение:** Проверка наличия и размера скопированных файлов в `/data/coolify/proxy/dynamic/`.

#### Ошибка 3: `failed to find any PEM data in key input` и ловушка симлинков Let's Encrypt
* **Причина:** Файлы в каталоге `/etc/letsencrypt/live/start-team.uz/` являются относительными символическими ссылками (symlinks), указывающими на каталог `../../archive/start-team.uz/`.
  При выполнении стандартной команды `cp` копируются сами ссылки. Внутри контейнера Traefik каталог `/etc/letsencrypt/archive` не смонтирован, ссылка оказывается битой, Traefik не может открыть файл и пытается распарсить путь к файлу как открытый текст ключа.
* **Решение:** Обязательное использование флага `cp -L` (разыменование ссылок, копирование реального содержимого байт в файл назначения).

### 7.3. Эталонные команды копирования и конфигурирования
Выполните на сервере:

```bash
# 1. Создание каталога и копирование реальных файлов сертификатов
mkdir -p /data/coolify/proxy/dynamic
cp -L /etc/letsencrypt/live/start-team.uz/fullchain.pem /data/coolify/proxy/dynamic/fullchain.pem
cp -L /etc/letsencrypt/live/start-team.uz/privkey.pem /data/coolify/proxy/dynamic/privkey.pem
chmod 644 /data/coolify/proxy/dynamic/*.pem

# 2. Создание файла динамической конфигурации Traefik
cat << 'EOF' > /data/coolify/proxy/dynamic/certs.yaml
tls:
  certificates:
    - certFile: /traefik/dynamic/fullchain.pem
      keyFile: /traefik/dynamic/privkey.pem
  stores:
    default:
      defaultCertificate:
        certFile: /traefik/dynamic/fullchain.pem
        keyFile: /traefik/dynamic/privkey.pem
EOF

# 3. Перезапуск прокси для применения чистого состояния
docker restart coolify-proxy
```

### 7.4. Проверка отдачи сертификата сервером
Проверьте работу TLS-сокета на сервере локально через утилиту `openssl`:
```bash
echo | openssl s_client -servername office.start-team.uz -connect 127.0.0.1:443 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```
Вывод должен подтвердить:
* `subject=CN = *.start-team.uz`
* `issuer=C = US, O = Let's Encrypt`
* Срок действия: 90 дней вперед от текущей даты.

---

## 8. Полная автоматизация автопродления сертификатов (Certbot Deploy-Hook)

### 8.1. Принцип работы автоматического продления
Сертификаты Let's Encrypt имеют срок жизни 90 дней. Certbot содержит встроенную системную службу `certbot.timer`, которая дважды в сутки проверяет состояние сертификатов и автоматически запускает их продление, если до окончания срока остается менее 30 дней.

Однако простого продления файлов в `/etc/letsencrypt/` недостаточно: обновленные файлы должны автоматически попасть в каталог Traefik `/data/coolify/proxy/dynamic/`.

### 8.2. Создание скрипта автоматического деплоя (Deploy-Hook)
Certbot поддерживает механизм хуков развертывания: любой исполняемый скрипт в каталоге `/etc/letsencrypt/renewal-hooks/deploy/` автоматически вызывается ядром Certbot только после успешного получения нового сертификата.

1. Создайте исполняемый скрипт обновления:
```bash
cat << 'EOF' > /etc/letsencrypt/renewal-hooks/deploy/update-coolify-traefik.sh
#!/bin/bash
cp -L /etc/letsencrypt/live/start-team.uz/fullchain.pem /data/coolify/proxy/dynamic/fullchain.pem
cp -L /etc/letsencrypt/live/start-team.uz/privkey.pem /data/coolify/proxy/dynamic/privkey.pem
chmod 644 /data/coolify/proxy/dynamic/*.pem
EOF

chmod +x /etc/letsencrypt/renewal-hooks/deploy/update-coolify-traefik.sh
```

### 8.3. Тестирование автоматического продления
Выполните симуляцию автопродления:
```bash
certbot renew --dry-run
```
*Важное примечание:* При передаче флага `--dry-run` Certbot связывается со тестовым окружением ACME Staging, не сохраняет боевые файлы на диск и не вызывает хуки (чтобы избежать затирания рабочих ключей). Вывод `Congratulations, all simulated renewals succeeded` подтверждает полную готовность и автономность автоматического продления.

---

## 9. Регламент добавления новых проектов

Благодаря настроенному регулярному выражению в MikroTik (`.*\.start-team\.uz`) и Wildcard-сертификату в Traefik добавление любого нового проекта осуществляется без вмешательства в сетевую конфигурацию.

### Пошаговый алгоритм:
1. Запустите в Docker / Coolify любой контейнер приложения (например, CRM, ERP, Telegram-бот, API-бэкенд).
2. Подключите контейнер приложения к сети `start-network`.
3. В параметрах домена Coolify (или в метках Docker labels) укажите желаемый домен с протоколом HTTPS, например:
   * `https://crm.start-team.uz`
   * `https://billing.start-team.uz`
   * `https://api.start-team.uz`
4. **Результат:** Проект мгновенно начинает открываться со всех офисных компьютеров и смартфонов:
   * DNS MikroTik автоматически перенаправит запрос на `192.168.1.201`.
   * MikroTik перешлет пакеты в Traefik.
   * Traefik применит готовый Wildcard SSL-сертификат и направит запрос в контейнер по его Docker-тегу.

---

## 10. Книга устранения неисправностей и аварийного восстановления (Runbook)

### 10.1. Таблица типовых сбоев и пошаговых решений

| Симптом ошибки | Вероятная причина | Точные команды решения |
| :--- | :--- | :--- |
| **`DNS_PROBE_FINISHED_NXDOMAIN`** | Очистились статические DNS-записи в MikroTik после сброса диска. | В WinBox New Terminal:<br>`/ip dns set allow-remote-requests=yes`<br>`/ip dns static add name=office.start-team.uz address=192.168.1.201`<br>`/ip dns static add regexp=".*\\.start-team\\.uz" address=192.168.1.201` |
| **`no route to host`** при пинге из MikroTik | Сетевые интерфейсы получили маску `/32` вместо `/24`. | В WinBox New Terminal:<br>`/ip address set [find interface=ether1] address=192.168.1.201/24`<br>`/ip address set [find interface=ether2] address=172.0.0.2/24` |
| **`ERR_CONNECTION_TIMED_OUT`** | Отсутствует правило Masquerade (Src-NAT) на `ether2`, пакеты ответа уходят в шлюз моста Docker. | В WinBox New Terminal:<br>`/ip firewall nat add chain=srcnat out-interface=ether2 action=masquerade` |
| **Красная надпись «Не защищено»** в браузере при валидном сертификате на сервере | Chrome закешировал старую TLS-сессию с временным сертификатом Traefik. | 1. Открыть вкладку в режиме Инкогнито (`Ctrl + Shift + N`).<br>2. Очистить сокеты в Chrome: открыть `chrome://net-internals/#sockets` и нажать **Flush socket pools**.<br>3. Перезапустить браузер. |
| **Лог Traefik: `node: keyFile not found`** | Ошибка отступов/табуляции в файле `certs.yaml`. | Перезаписать файл эталонным содержимым через `cat << 'EOF' > /data/coolify/proxy/dynamic/certs.yaml` строго с двумя пробелами на уровень вложенности. |
| **Лог Traefik: `failed to find any PEM data in key input`** | Символические ссылки скопированы без разыменования (`cp` без флага `-L`). | Выполнить на сервере:<br>`cp -L /etc/letsencrypt/live/start-team.uz/*.pem /data/coolify/proxy/dynamic/`<br>`chmod 644 /data/coolify/proxy/dynamic/*.pem`<br>`docker restart coolify-proxy` |
| **Интерфейсы `ether3`, `ether4` с IP `0.0.0.0` в WinBox** | Пересоздание контейнера с новым случайным MAC-адресом. | 1. Убедиться в наличии строки `mac_address: be:c0:19:d9:3d:34` в `docker-compose.yml`.<br>2. Очистить persistent storage: `rm -rf /data/coolify/services/mikrotik/data/*`<br>3. Перезапустить контейнер: `docker compose up -d`. |

### 10.2. Холодный запуск всей системы с нуля (Полный Disaster Recovery за 5 минут)
Если сервер был полностью переустановлен, последовательность полного поднятия связки:

```bash
# 1. Активация macvlan-shim
systemctl enable --now macvlan-shim

# 2. Выпуск сертификата (или восстановление из резервной копии /etc/letsencrypt)
certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.start-team.uz" -d "start-team.uz" --register-unsafely-without-email --agree-tos

# 3. Подключение Traefik к сети
docker network connect start-network coolify-proxy

# 4. Копирование сертификатов и конфига
mkdir -p /data/coolify/proxy/dynamic
cp -L /etc/letsencrypt/live/start-team.uz/*.pem /data/coolify/proxy/dynamic/
chmod 644 /data/coolify/proxy/dynamic/*.pem
# (Создать certs.yaml согласно разделу 7.3)
docker restart coolify-proxy

# 5. Запуск MikroTik CHR
cd /data/coolify/services/mikrotik && docker compose up -d

# 6. В WinBox применить конфигурационный скрипт:
# /ip address set [find interface=ether1] address=192.168.1.201/24
# /ip address set [find interface=ether2] address=172.0.0.2/24
# /ip firewall nat add chain=dstnat dst-address=192.168.1.201 protocol=tcp dst-port=80,443 action=dst-nat to-addresses=172.0.0.3
# /ip firewall nat add chain=srcnat out-interface=ether2 action=masquerade
# /ip dns set allow-remote-requests=yes
# /ip dns static add regexp=".*\\.start-team\\.uz" address=192.168.1.201
```
