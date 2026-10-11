# Руководство по развертыванию MikroTik CHR в Docker на ZimaOS

Подробная пошаговая инструкция по запуску и настройке виртуального маршрутизатора MikroTik RouterOS (Cloud Hosted Router) в Docker-окружении на сервере ZimaOS с одним физическим сетевым портом.

---

## 1. Архитектура и принцип работы

### Исходные условия:
- Физический сервер ZimaOS с одним портом Ethernet (`eth0`), подключенный к основному GPON-роутеру провайдера.
- Основной GPON-роутер (`192.168.1.1`) раздает адреса домашней/офисной сети (`192.168.1.0/24`).

### Задача:
- Развернуть MikroTik CHR в Docker с аппаратным ускорением процессора (`/dev/kvm`).
- Выделить контейнеру MikroTik собственный независимый IP-адрес **`192.168.1.200`** в физической локальной сети через драйвер `macvlan`.
- Организовать сетевой мост `macvlan-shim`, чтобы сам сервер ZimaOS имел прямой доступ к MikroTik.
- Настроить MikroTik как локальный DNS-сервер для перенаправления доменов на сервисы хоста.
- Подключить гостевой роутер в топологию без поддержки VLAN на GPON-терминале.

```text
[GPON-роутер провайдера] (Шлюз: 192.168.1.1)
           |
           | Физический кабель в LAN1
           v
[Сетевая карта сервера: eth0]
           |
           +---> [Хост ZimaOS]
           |          |
           |          +--- [macvlan-shim: 192.168.1.199] (Служебный интерфейс связи)
           |
           +---> [macvlan сеть Docker]
                      |
                      v
             [Контейнер MikroTik CHR] (IP: 192.168.1.200)
             - Порт 8291: WinBox
             - Порт 80: WebFig (веб-интерфейс)
             - Порт 53: DNS-сервер
```

---

## 2. Этап 1. Настройка хоста ZimaOS (Служебный мост macvlan-shim)

По соображениям изоляции ядра Linux, хост-система не может напрямую связываться со своими контейнерами в сети `macvlan` через один и тот же родительский интерфейс. Для решения этой задачи на сервере создается интерфейс `macvlan-shim`.

### 2.1. Создание сервиса через текстовый редактор nano
1. Откройте файл в редакторе nano:
```bash
nano /etc/systemd/system/macvlan-shim.service
```

2. Вставьте в открывшееся окно следующий текст:
```ini
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
ExecStart=/sbin/ip route add 192.168.1.208/28 dev macvlan-shim
ExecStart=/sbin/ip route add 192.168.1.224/27 dev macvlan-shim
ExecStop=/sbin/ip link del macvlan-shim
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

3. Сохраните и закройте файл:
- Нажмите `Ctrl + O` и затем `Enter` (для сохранения файла).
- Нажмите `Ctrl + X` (для выхода из редактора nano).

### 2.2. Активация и запуск сервиса
```bash
systemctl daemon-reload
systemctl enable --now macvlan-shim
```

### 2.3. Проверка статуса
```bash
systemctl status macvlan-shim
```
Статус должен быть: `Active: active (exited)` со всеми шагами в статусе `0/SUCCESS`.

---

## 3. Этап 2. Развертывание Docker Compose

### 3.1. Подготовка каталогов
```bash
mkdir -p /media/ZimaOS-HD/AppData/routeros
mkdir -p ~/Docker/mikrotik
cd ~/Docker/mikrotik
```

### 3.2. Создание файла `docker-compose.yml` через nano
1. Откройте файл в редакторе nano:
```bash
nano ~/Docker/mikrotik/docker-compose.yml
```

2. Вставьте следующее содержимое:
```yaml
networks:
  lan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1

services:
  mikrotik:
    container_name: mikrotik-chr
    image: ghcr.io/zmseld/chr:7.24.5
    restart: unless-stopped
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
        ipv4_address: 192.168.1.200
    volumes:
      - source: /media/ZimaOS-HD/AppData/routeros
        target: /routeros
        type: bind
```

3. Сохраните и закройте файл:
- Нажмите `Ctrl + O` и затем `Enter` (для сохранения файла).
- Нажмите `Ctrl + X` (для выхода из редактора nano).

### 3.3. Запуск контейнера
```bash
docker compose pull
docker compose up -d
```

### 3.4. Проверка работы
Просмотр журнала запуска:
```bash
docker logs -f mikrotik-chr
```
Проверка доступности по сети с сервера ZimaOS:
```bash
ping -c 3 192.168.1.200
```

---

## 4. Этап 3. Подключение и первичное управление

### Способ 1: WinBox (Рекомендуется)
1. Загрузите утилиту WinBox с сайта производителя: [mikrotik.com/download](https://mikrotik.com/download).
2. Запустите WinBox на компьютере в локальной сети.
3. В строке **Connect To** укажите: `192.168.1.200`.
4. В строке **Login** введите: `admin`.
5. Поле **Password** оставьте пустым.
6. Нажмите кнопку **Connect**.

### Способ 2: Веб-интерфейс (WebFig)
1. Откройте веб-браузер на любом устройстве в сети.
2. Перейдите по адресу: `http://192.168.1.200/`.
3. Авторизуйтесь с логином `admin` без пароля.

### Установка пароля администратора (Обязательно):
В терминале RouterOS или в меню WinBox (**System** -> **Users**):
```routeros
/user set admin password="ВашНадежныйПароль"
```

---

## 5. Этап 4. Настройка локального DNS-сервера

Для перенаправления доменных имен (например, `aDom11.com` или внутренних поддоменов) на IP-адрес сервера без выхода в интернет:

### 5.1. Включение DNS-сервера
В терминале MikroTik выполните:
```routeros
/ip dns set allow-remote-requests=yes servers=1.1.1.1,8.8.8.8
```

### 5.2. Добавление статических DNS-записей
Перенаправление конкретного домена на ZimaOS:
```routeros
/ip dns static add name="aDom11.com" address=192.168.1.199 ttl=1d
```
Перенаправление группы поддоменов через регулярное выражение:
```routeros
/ip dns static add regexp=".*\\.aDom11\\.com" address=192.168.1.199 ttl=1d
```

---

## 6. Этап 5. Подключение гостевого роутера без поддержки VLAN

Если GPON-роутер провайдера (например, BT-G710AX) не поддерживает разделение портов по VLAN, используется схема со статическим шлюзом через MikroTik:

1. Подключите кабель из порта **LAN4** GPON-роутера в порт **WAN** гостевого роутера.
2. В интерфейсе гостевого роутера настройте изолированную сеть для клиентов (например, `10.10.20.1/24`) с собственным DHCP-сервером и NAT.
3. В параметрах WAN-подключения гостевого роутера укажите статические настройки:
   - **IP-адрес WAN:** `192.168.1.15` (свободный адрес из домашней сети).
   - **Маска:** `255.255.255.0`
   - **Шлюз по умолчанию (Default Gateway):** `192.168.1.200` (MikroTik CHR).
   - **DNS-сервер:** `192.168.1.200`.

**Результат:**
- Гостевые устройства получают адреса `10.10.20.x` от своего роутера.
- Трафик гостей уходит в интернет строго через MikroTik `192.168.1.200`, где фильтруется и ограничивается по скорости.
- Офисные устройства в портах LAN1-LAN3 защищены от нежелательного DHCP-трафика.

---

## 7. Этап 6. Регламент аварийного восстановления

### 7.1. Сброс настроек MikroTik CHR до заводских
Если в процессе настройки RouterOS допущена критическая ошибка:
```bash
cd ~/Docker/mikrotik
docker compose down
rm -f /media/ZimaOS-HD/AppData/routeros/chr.img
docker compose up -d
```
При запуске контейнер автоматически развернет чистый заводской образ диска с сохранением исходного MAC-адреса.

### 7.2. Аппаратный сброс GPON-терминала BT-G710AX
Если утерян доступ к веб-панели терминала:
1. Не отключая питание, найдите углубление с кнопкой **Reset** на задней панели.
2. Зажмите кнопку тонким предметом на 15–20 секунд до одновременного мигания светодиодов.
3. Дождитесь полной загрузки (2 минуты). Терминал вернется к IP `192.168.1.1` и заводским учетным данным с наклейки на корпусе.
