# Скрипт проверки доступности портала по HTTPS средствами MikroTik

**Оборудование:** MikroTik RouterOS v7.22.1 (stable), RB3011UiAS

## Заменить на свои значения

- `192.168.0.1` — адрес проверяемого портала
- `--- TOKEN ---` — токен VK
- `2000001111` — идентификатор чата VK (`peer_id`)
- `admin` — владелец скрипта

## Проблема

Бесконечный спам алертов.

## Локализация проблемы

```routeros
/tool traceroute 192.168.0.1
```

```text
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS         LOSS   SENT  LAST    AVG   BEST  WORST  STD-DEV
0  198.51.100.105  11.4%   166  72.8ms  29.9  0.6   125.2  27.3
1  10.200.255.253  7.3%    166  1.9ms   2.2   0.6   10.5   2.2
2  198.51.100.17   0.6%    165  1ms     1.7   0.6   13.7   2
3  203.0.113.137   1.8%    165  3.4ms   4.1   2.5   41.1   3.6
4  203.0.113.53    0.6%    165  3.5ms   4.7   2.9   28.6   3.2
5  203.0.113.250   5.5%    165  20.6ms  13.4  4.4   131.8  15.2
6  192.0.2.49      7.3%    165  5.4ms   11.5  5.1   91.3   11.8
7  192.168.0.1     0.6%    165  4.8ms   15.1  4.5   654.4  62.3
```

**Джиттер и дрейф RTT (WORST = 654.4 мс, STD-DEV = 62.3).** Несмотря на среднюю задержку AVG = 15.1 мс, на целевом хосте (хоп 7) зафиксирован максимальный всплеск RTT 654.4 мс.

**Потери пакетов на промежуточных узлах (Control Plane Rate Limiting).** Потери 5.5%–11.4% на узлах 0, 1, 5, 6 при почти нулевой абсолютной потере на конечном узле (0.6% на хопе 7) указывают на лимитирование ICMP-ответов (CoPP/rate-limit) самими маршрутизаторами на пути, а не на реальную потерю L3-трафика.

**Почему замена на `type=icmp` не убрала спам.** Netwatch без логики гистерезиса реагирует на любое одиночное превышение порога timeout или потерянный пакет. Если один ICMP-пакет теряется (0.6% потерь = 1 потерянный пакет из 165) или задерживается выше таймаута, Netwatch мгновенно запускает down-script. На следующем шаге пакет проходит, и мгновенно запускается up-script.

| Критерий | `type=https-get` (L7) | `type=icmp` (L3) |
|---|---|---|

## Скрипт

```routeros
/system script
add dont-require-permissions=no name=check_portal owner=admin policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive source="
:global portalFailCnt
:global portalStatus

:if ([:len $portalFailCnt] = 0) do={ :set portalFailCnt 0 }
:if ([:len $portalStatus] = 0) do={ :set portalStatus "UP" }

:local targetUrl "https://192.168.0.1/"
:local vkToken "--- TOKEN ---"
:local peerID "2000001111"
:local sysName [/system identity get name]

:local checkResult false
:do {
    /tool fetch mode=https url=$targetUrl keep-result=no
    :set checkResult true
} on-error={
    :set checkResult false
}

:if ($checkResult = false) do={
    :set portalFailCnt ($portalFailCnt + 1)

    :if ($portalFailCnt >= 3 && $portalStatus = "UP") do={
        :set portalStatus "DOWN"
        :log error "PORTAL is DOWN! Sending VK notification..."

        :local msg ($sysName . "%20PORTAL%20DOWN")
        :local vkUrl ("https://api.vk.com/method/messages.send?access_token=" . $vkToken . "&v=5.199&peer_id=" . $peerID . "&random_id=0&message=" . $msg)

        :do {
            /tool fetch mode=https url=$vkUrl keep-result=no
        } on-error={
            :log error "Failed to send VK DOWN alert"
        }
    }
} else={
    :if ($portalStatus = "DOWN") do={
        :set portalStatus "UP"
        :log info "PORTAL is UP! Sending VK notification..."

        :local msg ($sysName . "%20PORTAL%20UP")
        :local vkUrl ("https://api.vk.com/method/messages.send?access_token=" . $vkToken . "&v=5.199&peer_id=" . $peerID . "&random_id=0&message=" . $msg)

        :do {
            /tool fetch mode=https url=$vkUrl keep-result=no
        } on-error={
            :log error "Failed to send VK UP alert"
        }
    }
    :set portalFailCnt 0
}"
```

## Планировщик

```routeros
/system scheduler
add interval=10s name=sched_check_portal on-event="/system script run check_portal" policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive
```

Ручной запуск:

```routeros
/system script run check_portal
```

```text
      status: finished
        code: 200
  downloaded: 2KiB
       total: 2KiB
    duration: 1s
```

## Корректировка логов

```routeros
/system logging
set 0 topics=info,!fetch
```

## Логи падения соединения и отправки уведомления

```text
2026-09-07 13:01:05 fetch,info Download from 192.168.0.1 FAILED: Host is unreachable
2026-09-07 13:01:15 fetch,info Download from 192.168.0.1 FAILED: Host is unreachable
2026-09-07 13:01:25 fetch,info Download from 192.168.0.1 FAILED: Host is unreachable
2026-09-07 13:01:25 script,error PORTAL is DOWN! Sending VK notification...
2026-09-07 13:01:25 fetch,info Download from api.vk.com FINISHED
```

## Логи успешного соединения и отправки уведомления

```text
2026-09-07 13:02:12 fetch,info Download from 192.168.0.1 FINISHED
2026-09-07 13:02:13 script,info PORTAL is UP! Sending VK notification...
2026-09-07 13:02:13 fetch,info Download from api.vk.com FINISHED
2026-09-07 13:02:22 fetch,info Download from 192.168.0.1 FINISHED
```
