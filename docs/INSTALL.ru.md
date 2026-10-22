# Подробная инструкция по установке

🇬🇧 [English](INSTALL.md) | 🇷🇺 Русский · [К README](../README.ru.md)

Все команды выполняются по SSH на роутере, если не сказано иное.

## 1. Проверь роутер и установи пакеты

```sh
cat /etc/openwrt_release
df -h /overlay
```

Нужно несколько МБ свободного места во flash.

OpenWrt 24.10:

```sh
opkg update
opkg install curl ca-bundle # возможно еще и libcurl4, смотри примечание ниже
```

Пакеты **curl** и **libcurl4** на роутере должны быть совместимы по версии, поэтому в моём случае потребовалось еще установить: `opkg install libcurl4`

OpenWrt 25.x:

```sh
apk update
apk add curl ca-bundle # возможно еще и libcurl4
```

## 2. Узнай интерфейсы основной и гостевой сетей

```sh
uci show dhcp | grep -E "dhcp\.(lan|guest)\.interface"
uci -q get network.lan.device
uci -q get network.guest.device
ip -br -4 addr
```

Нужны имена устройств, на которых работает DHCP, например `br-lan` и `br-guest`. На твоём роутере они могут отличаться (например, `br-lan.20` для VLAN): сверься по подсетям в выводе `ip -br -4 addr`.

## 3. Создай Telegram-бота и узнай chat id

1. В Telegram открой **@BotFather**, отправь `/newbot`, следуй подсказкам и скопируй **токен**.
2. Открой чат с новым ботом и нажми **Start** (или добавь бота в группу и напиши там любое сообщение).
3. В браузере открой `https://api.telegram.org/bot<ТОКЕН>/getUpdates` и найди `"chat":{"id":...}`. Это число и есть `chat_id` (у групп он обычно отрицательный).

Каждый, кто должен получать уведомления, должен один раз нажать **Start** у бота, иначе Telegram отклонит сообщения для него.

## 4. Установи файлы

Создай директорию:

`mkdir -p /etc/lanwatch && chmod 700 /etc/lanwatch`

Для установки скрипта выполни команду `cat > /usr/bin/lanwatch-leases <<'EOF'`
Вставь **весь** текст из `src/lanwatch-leases.ru`, нажми Enter, на отдельной строке введи `EOF` и снова нажми Enter. 

Для установки конфига выполни команду `cat > /etc/lanwatch/config <<'EOF'`
Вставь **весь** текст из `src/config.example` (желательно перед вставкой сразу указать `BOT_TOKEN`, `CHAT_IDS`, `ROUTER_NAME`, `MAIN_IF`, `GUEST_IF`, но можно и потом, смотри ниже), нажми Enter, на отдельной строке введи `EOF` и снова нажми Enter.

Далее:

```sh
chmod 700 /usr/bin/lanwatch-leases
chmod 600 /etc/lanwatch/config
sh -n /usr/bin/lanwatch-leases && echo "синтаксис OK"
```

## 5. Отредактируй конфиг (если не сделал заранее)

```sh
vi /etc/lanwatch/config
```

Укажи `BOT_TOKEN`, `CHAT_IDS`, `ROUTER_NAME`, `MAIN_IF`, `GUEST_IF` (см. таблицу в README). В `vi`: `i` для редактирования, `Esc`, затем `:wq` и Enter для сохранения.

## 6. Проверь Telegram отдельно

```sh
. /etc/lanwatch/config

for chat in $CHAT_IDS; do
    curl -fsS --connect-timeout 5 --max-time 10 \
        -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${chat}" \
        --data-urlencode "text=Тест уведомлений OpenWrt"
    echo
done
```

Каждый получатель должен получить сообщение, а в ответе должно быть `"ok":true`. Пока тест не пройдёт, дальше не переходи. Частые причины ошибок: пользователь не нажал Start, бот не добавлен в группу, неверный `chat_id`.

## 7. Первый запуск в тихом режиме (режим обучения)

Пока существует файл `/etc/lanwatch/learning`, база пополняется, но уведомления не отправляются. Так все текущие устройства записываются без лавины сообщений.

```sh
touch /etc/lanwatch/learning
/usr/bin/lanwatch-leases
cat /etc/lanwatch/devices.db
```

В базе по одной строке на устройство: `MAC|IP|имя|время последнего появления (unix)`. Все подключённые сейчас устройства должны быть в списке.

## 8. Включи cron

```sh
grep -q lanwatch-leases /etc/crontabs/root 2>/dev/null || echo '* * * * * /usr/bin/lanwatch-leases' >> /etc/crontabs/root
/etc/init.d/cron enable
/etc/init.d/cron restart
```

Если раз в минуту слишком часто, поставь `*/2` или `*/5` вместо `*` в первом поле. Интервал влияет только на задержку уведомления, а не на то, заметим ли мы устройство: аренда остаётся в файле, пока не истечёт.

По желанию: crond пишет в системный журнал каждый запуск задания (с меткой `cron.err`, хотя это не ошибка). Чтобы скрыть эти строки:

```sh
uci set system.@system[0].cronloglevel='9'
uci commit system
/etc/init.d/cron restart
```

## 9. Выключи режим обучения

Все текущие аренды уже в базе, поэтому уведомления можно включать сразу:

```sh
rm /etc/lanwatch/learning
```

Если хочешь избежать уведомлений «новое устройство» от устройств, которые сейчас офлайн (и потому не попали в файл аренд), оставь режим обучения включённым на сутки.

## 10. Проверь работу

При выключенном режиме обучения имитируем устройство с помощью поддельного файла аренд:

```sh
printf '0 02:aa:bb:cc:dd:01 192.168.50.250 test-host *\n' > /tmp/test.leases
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

Должно прийти «Новое устройство». Имитация смены IP:

```sh
printf '0 02:aa:bb:cc:dd:01 192.168.50.251 test-host *\n' > /tmp/test.leases
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

Имитация устройства, отсутствовавшего 45 дней:

```sh
OLD=$(( $(date +%s) - 45*86400 ))
sed -i "s/^\(02:aa:bb:cc:dd:01|[^|]*|[^|]*\)|.*/\1|$OLD/" /etc/lanwatch/devices.db
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

Имитация недоступного Telegram (сообщения должны попасть в очередь и уйти при следующем запуске):

```sh
sed -i '/^02:aa:bb:cc:dd:01|/d' /etc/lanwatch/devices.db
TG_API=http://127.0.0.1:9 LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
ls /etc/lanwatch/queue
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
ls /etc/lanwatch/queue
```

Уборка:

```sh
sed -i '/^02:aa:bb:cc:dd:01|/d' /etc/lanwatch/devices.db
rm -f /tmp/test.leases
```

Подключи реальное новое устройство к основной и к гостевой сети: в течение минуты должно прийти уведомление с правильным названием сети. Ошибки пишутся в системный журнал: `logread -e lanwatch`.

## Сохранение настроек при обновлении прошивки роутера (по желанию)

**sysupgrade с сохранением настроек.** Твои собственные файлы и установленные пакеты по умолчанию не сохраняются. Перед обновлением:

```sh
cat >> /etc/sysupgrade.conf <<'EOF'
/etc/lanwatch/
/usr/bin/lanwatch-leases
/etc/crontabs/root
EOF
```

После обновления установи пакеты заново (шаг 1), выполни `/etc/init.d/cron enable && /etc/init.d/cron restart` и проверь:

```sh
ls -l /usr/bin/lanwatch-leases /etc/lanwatch
ip -br -4 addr
logread -e lanwatch
```

Скрипт при смене версии OpenWrt не меняется, отличается только команда менеджера пакетов.

**Обновление самого скрипта.** Замени `/usr/bin/lanwatch-leases` новой версией. Конфиг и база остаются совместимыми; устройствам из старой базы время последнего появления проставится при первом запуске.

## Диагностика

| Симптом | Что проверить |
|---|---|
| Нет уведомлений, ошибок нет | Включён режим обучения: `ls /etc/lanwatch/learning` (файла быть не должно) |
| Скрипт перестал работать после принудительного завершения | Сними блокировку: `rmdir /tmp/lanwatch.lock` |
| Гостевые устройства не появляются | Посмотри `cat /tmp/dhcp.leases`. Если гостевой DHCP работает в отдельном экземпляре dnsmasq со своим файлом аренд, запусти скрипт второй раз с `LEASES=/путь/к/файлу` |
| В сообщении нет производителя | Проверь `curl https://api.macvendors.com/84:38:38`. У приватных (случайных) MAC производителя нет, это нормально |
| В журнале «Telegram отклонил сообщение» | Неверный токен или chat id, либо пользователь не нажимал Start |
| Все устройства пришли как «вернувшиеся» | Роутер (или cron) был выключен дольше `RETURN_DAYS`. При повторном запуске включи режим обучения |
| Неверное время сразу после загрузки | Учтено: пока часы не синхронизированы, логика с датами пропускается |

## Удаление

```sh
sed -i '/lanwatch-leases/d' /etc/crontabs/root
/etc/init.d/cron restart
rm -rf /usr/bin/lanwatch-leases /etc/lanwatch
```
