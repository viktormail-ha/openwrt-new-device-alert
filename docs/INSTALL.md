# Installation manual

🇬🇧 English | 🇷🇺 [Русский](INSTALL.ru.md) · [Back to README](../README.md)

All commands are run over SSH on the router unless stated otherwise.

## 1. Check the router and install packages

```sh
cat /etc/openwrt_release
df -h /overlay
```

You need a few MB of free flash space.

OpenWrt 24.10:

```sh
opkg update
opkg install curl ca-bundle # libcurl4 may also be required; see note below
```

The curl and libcurl4 packages on the router must have compatible versions. In my case, I also needed to install: `opkg install libcurl4`

OpenWrt 25.x:

```sh
apk update
apk add curl ca-bundle # libcurl4 may also be required; see note below
```

## 2. Find the interfaces of the main and guest networks

```sh
uci show dhcp | grep -E "dhcp\.(lan|guest)\.interface"
uci -q get network.lan.device
uci -q get network.guest.device
ip -br -4 addr
```

You need the names of the devices that serve DHCP, for example `br-lan` and `br-guest`. They may differ on your router (for example `br-lan.20` for a VLAN): match them by subnet in the output of `ip -br -4 addr`.

## 3. Create a Telegram bot and get your chat id

1. In Telegram open **@BotFather**, send `/newbot`, follow the prompts and copy the **token**.
2. Open a chat with your new bot and press **Start** (or add the bot to a group and write any message there).
3. In a browser open `https://api.telegram.org/bot<TOKEN>/getUpdates` and find `"chat":{"id":...}`. That number is your `chat_id` (group ids are usually negative).

Each person who should receive alerts must press **Start** in the bot once, otherwise Telegram will reject messages to them.

## 4. Install the files

Create a directory:

`mkdir -p /etc/lanwatch && chmod 700 /etc/lanwatch`

To install the script run the command `cat > /usr/bin/lanwatch-leases <<'EOF'`
Paste the **entire** content of `src/lanwatch-leases`, press Enter, type `EOF` on its own line and press Enter again. 

To install the config run the command `cat > /etc/lanwatch/config <<'EOF'`
Paste the **entire** content of `src/config.example` (before paste it's recommended to specify `BOT_TOKEN`, `CHAT_IDS`, `ROUTER_NAME`, `MAIN_IF`, `GUEST_IF` right away, but you can also set it later; see below), press Enter, type `EOF` on its own line and press Enter again.

Next:

```sh
chmod 700 /usr/bin/lanwatch-leases
chmod 600 /etc/lanwatch/config
sh -n /usr/bin/lanwatch-leases && echo "syntax OK"
```

## 5. Edit the config (if you haven't done so already)

```sh
vi /etc/lanwatch/config
```

Set `BOT_TOKEN`, `CHAT_IDS`, `ROUTER_NAME`, `MAIN_IF`, `GUEST_IF` (see the table in the README). In `vi`: press `i` to edit, `Esc` then `:wq` and Enter to save.

## 6. Test Telegram separately

```sh
. /etc/lanwatch/config

for chat in $CHAT_IDS; do
    curl -fsS --connect-timeout 5 --max-time 10 \
        -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${chat}" \
        --data-urlencode "text=OpenWrt notification test"
    echo
done
```

Every recipient must get the message and the reply must contain `"ok":true`. Do not continue until this works. Typical problems: the user did not press Start, the bot is not in the group, a wrong `chat_id`.

## 7. First run in silent (learning) mode

While `/etc/lanwatch/learning` exists, the database is filled but no alerts are sent. This records all current devices without a flood of messages.

```sh
touch /etc/lanwatch/learning
/usr/bin/lanwatch-leases
cat /etc/lanwatch/devices.db
```

The database contains one line per device: `MAC|IP|name|last seen (unix time)`. Every device that is connected now should be listed.

## 8. Enable cron

```sh
grep -q lanwatch-leases /etc/crontabs/root 2>/dev/null || echo '* * * * * /usr/bin/lanwatch-leases' >> /etc/crontabs/root
/etc/init.d/cron enable
/etc/init.d/cron restart
```

If every minute is too often, use `*/2` or `*/5` instead of `*` in the first field. The interval only affects the delay of an alert, not whether a device is noticed: a lease stays in the lease file until it expires.

Optional: crond logs every job start to the system log (as `cron.err`, although it is not an error). To hide these lines:

```sh
uci set system.@system[0].cronloglevel='9'
uci commit system
/etc/init.d/cron restart
```

## 9. Turn learning mode off

All current leases are already in the database, so you can switch alerts on right away:

```sh
rm /etc/lanwatch/learning
```

If you want to avoid "new device" alerts from devices that are offline now (and therefore not in the lease file), keep learning mode on for a day first.

## 10. Test

With learning mode off, simulate a device with a fake lease file:

```sh
printf '0 02:aa:bb:cc:dd:01 192.168.50.250 test-host *\n' > /tmp/test.leases
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

You should get "New device". Simulate an IP change:

```sh
printf '0 02:aa:bb:cc:dd:01 192.168.50.251 test-host *\n' > /tmp/test.leases
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

Simulate a device that was away for 45 days:

```sh
OLD=$(( $(date +%s) - 45*86400 ))
sed -i "s/^\(02:aa:bb:cc:dd:01|[^|]*|[^|]*\)|.*/\1|$OLD/" /etc/lanwatch/devices.db
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
```

Simulate an unreachable Telegram (messages must go to the queue and be delivered by the next run):

```sh
sed -i '/^02:aa:bb:cc:dd:01|/d' /etc/lanwatch/devices.db
TG_API=http://127.0.0.1:9 LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
ls /etc/lanwatch/queue
LEASES=/tmp/test.leases /usr/bin/lanwatch-leases
ls /etc/lanwatch/queue
```

Clean up:

```sh
sed -i '/^02:aa:bb:cc:dd:01|/d' /etc/lanwatch/devices.db
rm -f /tmp/test.leases
```

Finally connect a real new device to the main and to the guest network: an alert with the right network name should arrive within a minute. Errors go to the system log: `logread -e lanwatch`.

## Preserving Settings When Updating the Router Firmware (Optional)

**sysupgrade with "keep settings".** Your own files and installed packages are not kept by default. Before upgrading:

```sh
cat >> /etc/sysupgrade.conf <<'EOF'
/etc/lanwatch/
/usr/bin/lanwatch-leases
/etc/crontabs/root
EOF
```

After upgrading reinstall the packages (step 1), run `/etc/init.d/cron enable && /etc/init.d/cron restart` and check:

```sh
ls -l /usr/bin/lanwatch-leases /etc/lanwatch
ip -br -4 addr
logread -e lanwatch
```

The script does not change between OpenWrt versions; only the package manager command differs.

**Updating the script itself.** Replace `/usr/bin/lanwatch-leases` with the new version. The config and the database stay compatible; devices from an older database get their "last seen" time on the first run.

## Troubleshooting

| Symptom | What to check |
|---|---|
| No alerts, no errors | Learning mode is on: `ls /etc/lanwatch/learning` (the file must not exist) |
| The script stopped after being killed | Remove the lock: `rmdir /tmp/lanwatch.lock` |
| Guest devices never show up | Look at `cat /tmp/dhcp.leases`. If the guest DHCP runs in a separate dnsmasq instance with its own lease file, run the script a second time with `LEASES=/path/to/that/file` |
| No vendor in the message | Check `curl https://api.macvendors.com/84:38:38`. Private (random) MAC addresses have no vendor, that is normal |
| `Telegram rejected the message` in the log | Wrong token or chat id, or the user never pressed Start |
| All devices reported as "returned" | The router (or cron) was off for longer than `RETURN_DAYS`. Use learning mode while starting it again |
| Wrong time right after boot | Handled: until the clock is synchronized, time-based logic is skipped |

## Uninstall

```sh
sed -i '/lanwatch-leases/d' /etc/crontabs/root
/etc/init.d/cron restart
rm -rf /usr/bin/lanwatch-leases /etc/lanwatch
```
