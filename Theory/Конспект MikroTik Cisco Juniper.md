# Конспект: MikroTik, Cisco и Juniper

Краткое сравнение для курса. Практика в лабораториях — на **MikroTik** ([Настройка Firewall](Настройка%20Firewall.md)). Cisco и Juniper — тот же смысл политик, другой CLI.

Пример политики курса: **разрешить DNS** (UDP/TCP 53) из blue.net `172.16.10.0/24` на `172.16.20.53`; **запретить** инициацию из red.net в blue.net.

---

## 1. Общее

На всех трёх платформах есть:

* адреса на интерфейсах и маршрутизация;
* фильтрация по адресу, порту, протоколу;
* порядок правил (обычно побеждает первое совпадение);
* идея зон доверия (LAN / DMZ / WAN).

Отличается не «физика сети», а **язык конфигурации** и модель объектов (цепочки, ACL, zones).

---

## 2. Сравнение «на одном взгляде»

| | **MikroTik** | **Cisco** (IOS / ASA) | **Juniper** (SRX) |
| :--- | :--- | :--- | :--- |
| Стиль CLI | Пути `/ip ...`, сразу применяется | Режимы `enable` → `configure` | Иерархия `set ...`, нужен **`commit`** |
| Фильтрация | `/ip firewall filter`, цепочки `input`/`forward`/`output` | ACL + `access-group`; на ASA ещё security-level | Security zones + policies from-zone/to-zone |
| Stateful | `connection-state=established,related` | inspect / stateful на ASA и современном IOS | По умолчанию в security policies |
| Типичный «язык» | port 53 явно | `eq 53` в ACL | часто application `junos-dns` |
| Для новичка | Быстрый результат | Много режимов | Непривычен commit |

---

## 3. MikroTik (то, что в лаборатории)

* Трафик **через** роутер — цепочка **`forward`**.
* Трафик **к** роутеру (Winbox, SSH) — **`input`**.
* Правила: `accept` / `drop`, сверху вниз.

Идея ваших правил:

```text
/ip firewall filter add chain=forward action=accept connection-state=established,related
/ip firewall filter add chain=forward action=accept protocol=udp src-address=172.16.10.0/24 dst-address=172.16.20.53 dst-port=53
/ip firewall filter add chain=forward action=drop src-address=172.16.20.0/24 dst-address=172.16.10.0/24
```

Подробно: [Настройка Firewall](Настройка%20Firewall.md).

---

## 4. Cisco (IOS ACL — идея)

На маршрутизаторе IOS политика часто выглядит как numbered/named ACL и привязка к интерфейсу (`ip access-group ... in|out`).

Смысл «разрешить DNS blue → DNS-сервер» (упрощённо):

```text
access-list 100 permit udp 172.16.10.0 0.0.0.255 host 172.16.20.53 eq 53
access-list 100 permit tcp 172.16.10.0 0.0.0.255 host 172.16.20.53 eq 53
access-list 100 deny ip 172.16.20.0 0.0.0.255 172.16.10.0 0.0.0.255
```

На **ASA** добавляются зоны (`nameif`), **security-level** (чем выше — тем «довереннее») и ACL между зонами. Трафик из high в low по умолчанию проще, из low в high — только с явным разрешением. Это ближе к «ролям» DMZ, чем сырой IOS ACL.

Маска в ACL Cisco — **wildcard** (`0.0.0.255` ≈ «последний октет любой»), не CIDR `/24` как у MikroTik.

---

## 5. Juniper (SRX — идея)

Конфиг правят командами `set`, затем **`commit`**. Политика читается как «из зоны A в зону B».

Упрощённая схема смысла (не готовый полный конфиг стенда):

* интерфейс blue → zone `blue`, red → zone `red`;
* policy: from-zone blue to-zone red, match destination DNS, application DNS → permit;
* policy: from-zone red to-zone blue → deny (или не создавать permit).

Пример каркаса:

```text
set security policies from-zone blue to-zone red policy allow-dns match source-address any
set security policies from-zone blue to-zone red policy allow-dns match destination-address DNS-HOST
set security policies from-zone blue to-zone red policy allow-dns match application junos-dns
set security policies from-zone blue to-zone red policy allow-dns then permit
commit
```

Без `commit` изменения в рабочей конфигурации не появятся — главное отличие от RouterOS.

---

## 6. Как переносить знания с MikroTik

| Вы уже знаете | На Cisco думайте | На Juniper думайте |
| :--- | :--- | :--- |
| `forward` | ACL на интерфейсе / между зонами ASA | policy from-zone → to-zone |
| `accept` / `drop` | `permit` / `deny` | `then permit` / `deny` |
| `dst-port=53` | `eq 53` | application DNS |
| Порядок правил | Порядок строк ACL | Порядок политик в зоне |
| Сразу применилось | `write memory` после conf | обязательно `commit` |

**Итог:** выучив политику DMZ на MikroTik, вы понимаете **что** настраивать на Cisco и Juniper. Отдельно нужно выучить **как** это записать в их CLI — это следующий навык, не другая теория сетей.
