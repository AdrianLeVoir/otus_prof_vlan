# Домашнее задание: Строим бонды и вланы

## Текст задания

1. В Office1 в тестовой подсети появляются сервера с доп интерфейсами и адресами во
   внутренней сети testLAN:
   - testClient1 — 10.10.10.254
   - testClient2 — 10.10.10.254
   - testServer1 — 10.10.10.1
   - testServer2 — 10.10.10.1

   Развести вланами:
   - testClient1 <-> testServer1
   - testClient2 <-> testServer2

2. Между centralRouter и inetRouter «пробросить» 2 линка (общая internal сеть) и
   объединить их в бонд, проверить работу с отключением интерфейсов.

Формат — ansible; стенд развёрнут на Proxmox.

## Схема сети

```
                    mgmt 192.168.1.0/24 (vmbr0, все ВМ)
   testClient1 (192.168.1.160) ──┐
   testServer1 (192.168.1.161) ──┤ vmbr-testlan (общий L2, без VLAN-фильтрации)
   testClient2 (192.168.1.162) ──┤      каждая пара тегирует 802.1Q сама:
   testServer2 (192.168.1.163) ──┘      vlan1: testClient1<->testServer1 (10.10.10.0/24)
                                        vlan2: testClient2<->testServer2 (10.10.10.0/24)

   inetRouter (192.168.1.110)                        centralRouter (192.168.1.111)
   enp6s23 ─── vmbr-bond ─── enp7s2 ┐                bond0: 192.168.250.1/30 <-> 192.168.250.2/30
   enp7s1  ─── vmbr-bond ─── enp7s3 ┘                mode=active-backup, miimon=100
```

ВМ: testClient1 (540), testServer1 (541), testClient2 (542), testServer2 (543) —
клоны базовой Ubuntu 24.04; бонд добавлен на существующие inetRouter (501) и
centralRouter (502).

## Реализация

### 1. VLAN

Каждый тестовый хост сам создаёт 802.1Q сабинтерфейс на физическом порту testLAN
(`vlan1@enp6s19` с id=1 или `vlan2@enp6s19` с id=2, netplan). Мост пропускает
тегированные кадры прозрачно — изоляция обеспечивается тегом, свитч не нужен.

Адреса — по заданию (в обеих парах одинаковые: .254 и .1 — это допустимо, т.к.
пары изолированы на L2).

Проверка связности:

```
testClient1 -> testServer1 (vlan1): 2/2, 0% loss
testClient2 -> testServer2 (vlan2): 2/2, 0% loss
```

Проверка изоляции (правильный тест — по тегам, а не пингом: адреса в парах
совпадают, кросс-пинг «самопинговый»). tcpdump на testServer2 во время
одновременных пингов из обеих пар:

```
=== VLAN1 frames: 0      <- кадры пары 1 НЕ дошли (изоляция работает)
=== VLAN2 frames: 8      <- ICMP echo request/reply пары 2
=== SAMPLES===
... vlan 2, p 0, ethertype IPv4, 10.10.10.254 > 10.10.10.1: ICMP echo request
... vlan 2, p 0, ethertype IPv4, 10.10.10.1 > 10.10.10.254: ICMP echo reply
```

### 2. Bond (active-backup) между inetRouter и centralRouter

По 2 линка от каждого роутера в общую сеть vmbr-bond; bond0 через netplan:

```yaml
bonds:
  bond0:
    interfaces: [enp6s23, enp7s1]      # centralRouter: [enp7s2, enp7s3]
    macaddress: bc:24:11:51:75:14      
    addresses: [192.168.250.1/30]      # centralRouter: 192.168.250.2/30
    parameters:
      mode: active-backup
      mii-monitor-interval: 100
```

Состояние:

```
Bonding Mode: fault-tolerance (active-backup)
Currently Active Slave: enp7s1
Slave Interface: enp7s1  MII Status: up
Slave Interface: enp6s23 MII Status: up
```

Проверка: ping 192.168.250.2 -> .1 — 2/2.

### Тест отказоустойчивости

Пинг непрерывно (30 пакетов, i=0.5), в этот момент на centralRouter гасится
активный слейв: `ip link set enp7s2 down`:

```
Currently Active Slave: enp7s3   <- bond переключился на второй линк
29 replies received / 29 sent    <- 0 потерь, пинг не прервался
ip link set enp7s2 up            <- слейв возвращён в бонд
```

## Файлы

| Файл | Назначение |
|---|---|
| `ansible/hosts` | инвентарь: vlan_id/vlan_ip, bond_ip/bond_slaves/bond_mac |
| `ansible/provision.yml` | установка тулов, VLAN и bond через netplan |
| `ansible/templates/50-vlan.yaml.j2` | VLAN-сабинтерфейс на тестовых хостах |
| `ansible/templates/60-bond.yaml.j2` | bond0 active-backup с явным MAC |
| `README.md` | этот файл |
