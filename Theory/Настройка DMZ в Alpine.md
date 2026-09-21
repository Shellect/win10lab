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

Отредактируйте конфигурацию сетевых интерфейсов:
`nano /etc/network/interfaces`:

```text
auto eth0
iface eth0 inet static
    address 172.16.20.53
    netmask 255.255.255.0
    gateway 172.16.20.254
```
Для редактирования используйте текстовый редактор nano (рекомендуется)или vim (для опытных пользователей).Оба требуют установки `apk add nano` или `apk add vim`

Перезагрузите интерфейс: `ifdown eth0 && ifup eth0`.

### Шаг 3: Установка BIND

```bash
apk update
apk add bind bind-tools
```

### Шаг 4: Конфигурация зон

Создайте файл основной конфигурации `/etc/bind/named.conf`
```bash
cp /etc/bind/named.conf.authoritative named.conf
```
Отредактируйте `/etc/bind/named.conf`:
```
options {
    directory "/var/bind";

    // Слушаем на всех интерфейсах (важно для GNS3)
    listen-on { any; };
    listen-on-v6 { none; };

    // Разрешаем запросы от клиентов blue.net и localhost
    allow-query { localhost; 172.16.10.0/24; 172.16.20.0/24; };
    
    // Разрешаем рекурсию (для кэширования внешних запросов)
    recursion yes;

    // Отключаем DNSSEC-валидацию для простоты в лабораторной среде
    dnssec-validation no;

    // Форвардеры (если нужно разрешать внешние имена через шлюз MikroTik)
    // Если MikroTik не настроен как DNS, укажите публичные (например, 8.8.8.8)
    forwarders {
        172.16.20.254;
    };
};

// Подключаем локальные зоны
include "/etc/bind/named.conf.zones";
```

Отредактируйте файл с настройками зон:
Здесь мы объявляем зону прямого просмотра (lab.local) и обратную зону (20.16.172.in-addr.arpa)

```bash
nano /etc/bind/named.conf.zones
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

### Шаг 5: Файлы зон

Прямая зона:

```bash
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
chown named:named /etc/bind/db.lab.local /etc/bind/db.172.16.20
chmod 644 /etc/bind/db.lab.local /etc/bind/db.172.16.20
```

### Шаг 6: Проверка и запуск (OpenRC)

```bash
named-checkconf
named-checkzone lab.local /etc/bind/db.lab.local
named-checkzone 20.16.172.in-addr.arpa /etc/bind/db.172.16.20
rc-service named restart
rc-service named status
rc-update add named default
```

На сервере:

```bash
dig ns1.lab.local @127.0.0.1
dig www.lab.local @127.0.0.1
dig -x 172.16.20.53 @127.0.0.1
```
### Замечания

* IP `172.16.20.53` должен совпадать с адресом Alpine в DMZ.
* После правок зоны увеличивайте **Serial**.
* Логи: `sudo tail -f /var/log/messages | grep named`.
* Если named не отвечает снаружи — проверьте `listen-on { any; };`.
