# Настройка DMZ в Alpine

Пошаговая настройка DNS-сервера (BIND9) на Alpine Linux в сегменте **red.net** (DMZ) для [Второй лабораторной работы](../Lab/Вторая%20лабораторная%20работа.md).

| Параметр | Значение |
| :--- | :--- |
| Сегмент | **red.net** (DMZ / серверная) |
| ОС | Alpine Linux (VirtualBox) |
| IP DNS | `172.16.20.53/24` |
| Шлюз (MikroTik `ether2`) | `172.16.20.254` |
| Зона | `lab.local` |
| Клиенты | **blue.net** — `172.16.10.0/24` |

### Шаг 1: Виртуальная машина

1. Скачайте минимальный ISO Alpine (Standard или Virtual, ~50–150 МБ).
2. Создайте VM в VirtualBox: 1 vCPU, 256–512 МБ RAM, диск 1–2 ГБ.
3. Установите систему (`setup-alpine`).
4. Добавьте VM в GNS3: `Edit` → `Preferences` → `VirtualBox VMs` → `New`.
5. Подключите узел к коммутатору **red.net** (DMZ) в топологии лабораторной.

### Шаг 2: Статический IP

Имя интерфейса: `ip link` (часто `eth0`). Пример `/etc/network/interfaces`:

```text
auto eth0
iface eth0 inet static
    address 172.16.20.53
    netmask 255.255.255.0
    gateway 172.16.20.254
```

Примените: `sudo ifup eth0` или `sudo rc-service networking restart`.

### Шаг 3: Установка BIND

```bash
sudo apk update
sudo apk add bind bind-tools
```

### Шаг 4: Конфигурация зон

В Alpine зоны подключают через `/etc/bind/named.conf`. Удобнее вынести их в отдельный файл.

```bash
sudo nano /etc/bind/named.conf.zones
```

```text
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
};

zone "20.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.20";
};
```

В конце `/etc/bind/named.conf`:

```text
include "/etc/bind/named.conf.zones";
```

В секции `options`:

```text
options {
    listen-on { any; };
    allow-query { 172.16.10.0/24; 172.16.20.0/24; 127.0.0.1; };
    allow-recursion { 172.16.10.0/24; 127.0.0.1; };
};
```

### Шаг 5: Файлы зон

Прямая зона:

```bash
sudo cp /etc/bind/db.empty /etc/bind/db.lab.local 2>/dev/null || sudo touch /etc/bind/db.lab.local
sudo nano /etc/bind/db.lab.local
```

```text
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                          2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.lab.local.
ns1     IN      A       172.16.20.53
www     IN      A       172.16.20.53
```

Обратная зона:

```bash
sudo nano /etc/bind/db.172.16.20
```

```text
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                          2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.lab.local.
53      IN      PTR     ns1.lab.local.
```

Права:

```bash
sudo chown named:named /etc/bind/db.lab.local /etc/bind/db.172.16.20
sudo chmod 644 /etc/bind/db.lab.local /etc/bind/db.172.16.20
```

### Шаг 6: Проверка и запуск (OpenRC)

```bash
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/db.lab.local
sudo named-checkzone 20.16.172.in-addr.arpa /etc/bind/db.172.16.20
sudo rc-service named restart
sudo rc-service named status
sudo rc-update add named default
```

На сервере:

```bash
dig ns1.lab.local @127.0.0.1
dig www.lab.local @127.0.0.1
dig -x 172.16.20.53 @127.0.0.1
```

С другой машины:

```bash
dig ns1.lab.local @172.16.20.53
```

### Замечания

* IP `172.16.20.53` должен совпадать с адресом Alpine в DMZ.
* После правок зоны увеличивайте **Serial**.
* Логи: `sudo tail -f /var/log/messages | grep named`.
* Если named не отвечает снаружи — проверьте `listen-on { any; };`.
