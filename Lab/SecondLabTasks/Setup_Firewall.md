# Настройка Firewall

Пошаговая настройка фильтрации на **MikroTik** между **blue.net** и **red.net** для [Второй лабораторной работы](../Second.md).

| Параметр | Значение |
| :--- | :--- |
| Устройство | MikroTik RouterOS |
| blue.net (клиенты) | `172.16.10.0/24`, шлюз `ether1` = `172.16.10.254` |
| red.net (DMZ / DNS) | `172.16.20.0/24`, шлюз `ether2` = `172.16.20.254` |
| DNS | Alpine `172.16.20.53`, порт UDP/TCP **53** |

Предусловие: IP на интерфейсах уже назначены, DNS в DMZ настроен ([Настройка DMZ в Alpine](./Setup_DNS.md)).

---

## 1. Зачем firewall между зонами

**Firewall (межсетевой экран)** решает, каким пакетам можно проходить между сетями.

Идея DMZ:

* из **blue.net** к DNS в **red.net** — только нужный сервис (DNS, порт 53);
* из **red.net** в **blue.net** — сервер **не** должен сам открывать соединения к клиентам;
* остальные «лишние» направления между зонами — по политике курса (ниже — практичный минимальный набор).

Это **не** защита от DNS Spoofing внутри blue.net (третья лабораторная): атака на L2 до маршрутизатора. Firewall здесь про политику **между** сегментами.

На MikroTik правила смотрят в `/ip firewall filter`. Для трафика *через* роутер важна цепочка **`forward`**.

---

## 2. Цепочки filter (кратко)

| Цепочка | Когда срабатывает |
| :--- | :--- |
| `input` | Пакет **к самому** роутеру (Winbox, SSH на MikroTik) |
| `output` | Пакет **от** роутера |
| `forward` | Пакет **через** роутер (blue ↔ red) |

В этой инструкции настраиваем в основном **`forward`**.

Порядок правил важен: срабатывает **первое подходящее**. Сначала `accept`, потом `drop`.

---

## 3. Базовые правила (минимум для лабораторной)

В консоли MikroTik:

```text
/ip firewall filter add chain=forward action=accept connection-state=established,related comment="Allow established/related"
/ip firewall filter add chain=forward action=accept protocol=udp src-address=172.16.10.0/24 dst-address=172.16.20.53 dst-port=53 comment="DNS UDP blue->red"
/ip firewall filter add chain=forward action=accept protocol=tcp src-address=172.16.10.0/24 dst-address=172.16.20.53 dst-port=53 comment="DNS TCP blue->red"
/ip firewall filter add chain=forward action=accept protocol=icmp src-address=172.16.10.0/24 dst-address=172.16.20.0/24 comment="ICMP blue->red for lab checks"
/ip firewall filter add chain=forward action=drop src-address=172.16.20.0/24 dst-address=172.16.10.0/24 comment="Block red->blue"
```

Смысл по строкам:

1. **established,related** — ответы на уже разрешённые сессии (в т.ч. ответы DNS) не режутся зря.
2–3. Явно разрешить DNS blue → Alpine.
4. ICMP blue → red — чтобы `ping 172.16.20.53` из Lab 2 продолжал работать после включения фильтра (учебное удобство).
5. Запрет инициации из DMZ в клиентскую сеть.

Просмотр:

```text
/ip firewall filter print
```

Удалить правило по номеру (если ошиблись):

```text
/ip firewall filter remove numbers=N
```

---

## 4. Ужесточение (по желанию преподавателя)

После проверки DNS можно убрать «широкий» ICMP и оставить только порт 53 + established:

```text
/ip firewall filter remove [find comment="ICMP blue->red for lab checks"]
```

Или заменить на ICMP только к хосту DNS:

```text
/ip firewall filter add chain=forward action=accept protocol=icmp src-address=172.16.10.0/24 dst-address=172.16.20.53 comment="ICMP to DNS only"
```

(вставьте **выше** правила `drop red->blue`).

Дополнительно можно дропать всё прочее blue→red, кроме DNS:

```text
/ip firewall filter add chain=forward action=drop src-address=172.16.10.0/24 dst-address=172.16.20.0/24 comment="Drop other blue->red" place-before=[find comment="Block red->blue"]
```

Внимание: такое правило должно стоять **после** accept для DNS (и ICMP, если нужен), иначе DNS перестанет работать.

---

## 5. Проверка

С клиента в **blue.net** (PC1):

1. `ping 172.16.20.53` — если ICMP разрешён (шаг 3).
2. `nslookup www.lab.local 172.16.20.53` или `dig www.lab.local @172.16.20.53` — должен ответить `172.16.20.53`.
3. С Alpine (если есть второй адрес/утилита) попытка инициировать доступ в blue.net должна **не** проходить при правиле `Block red->blue` (зависит от того, что именно пробуете).

Счётчики правил:

```text
/ip firewall filter print stats
```

Если пакеты DNS не растут на правилах accept — проверьте порядок правил и адреса.

---

## 6. Связь с курсом

| Работа | Роль firewall |
| :--- | :--- |
| [Вторая лабораторная](../Lab/Вторая%20лабораторная%20работа.md) | Выполнить эту инструкцию (шаг про firewall) |
| [Третья лабораторная](../Lab/Третья%20лабораторная%20работа.md) | Понять: FW не спасает от spoofing внутри blue.net |
| [Теоретический конспект](../Notes/Теоретический%20конспект%20ко%20второй%20лабораторной.md) | Раздел про ACL / DMZ |