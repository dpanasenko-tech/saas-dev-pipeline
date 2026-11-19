# Измерения и следующий сбор

Hetzner, разовый снимок 19 ноября: 4 логических CPU, load average 11,19 / 11,89 / 11,61. Память 7 745 MiB, доступно 2 009 MiB, занято swap 5 039 из 8 191 MiB. Корневой диск 150 GiB, занято 123 GiB, доступно 22 GiB (86%). Один PostgreSQL потреблял около 688 MiB. Это смешанная машина. Load включает ожидание I/O, занятый swap сам по себе не измеряет текущий paging. Эти числа нельзя использовать для заказа production VPS.

На FirstVDS SSH с исходной машины завершился TCP timeout до аутентификации. Уточнить штатный маршрут в панели. На Timeweb сверить host key через консоль провайдера командой `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`. На Beelink сначала выполнить [первое подключение](beelink-ssh.md).

| Где | Сбор только для чтения | Для какого решения |
|---|---|---|
| FirstVDS | `nproc`, `free -m`, `df -hT`, `df -i`, `lsblk`, `uptime`, `vmstat 5 12`, `docker stats --no-stream`, `systemctl list-timers --all`, `ss -lnt` | Ёмкость и наличие offsite-расписания |
| FirstVDS и staging | За 7 дней существующих метрик извлечь интервалы sync, ночного backup и нагрузки UI. Если истории нет, согласовать сбор раз в минуту. CPU busy/steal, iowait, MemAvailable, swap-in/out, OOM, DB connections, рост WAL/томов, sync lag, HTTP p95 и ошибки | Пики отдельно и совместно, сравнение с базовым prod SLO |
| Hetzner | `docker compose ls`, `docker volume ls`, `docker system df`, списки timers/cron, Git worktrees, владельцы DNS, сервисов, незакоммиченных данных и других приложений | Частный реестр с решением сохранить/перенести/списать для каждой зависимости |
| Timeweb | После сверки ключа: `ip -br link`, `ip rule`, `ip route`, `ip -6 route`, `ss -lntu`, `resolvectl status`, локальная проверка firewall и контейнерных сетей | Сравнить маршруты до и после включения обоих VPN |
| Beelink | `lscpu`, `free -m`, `lsblk -o NAME,SIZE,MODEL`, модель из DMI, SMART накопителя, BIOS AC recovery, питание по ваттметру | Паспорт, пригодность к работе 24/7 и цена |
| GitHub | Проверить квоты/плательщика, защиту ветки и тегов, permissions, runner groups, registry retention, доступность billing. Видимость token в API может быть ограничена | Перед расходами и переносом release gate подтвердить возможности тарифа |

Вести замеры локально в нужном контуре. В публичный отчёт переносить агрегаты. История нагрузок, доступ владельца к восстановлению и working set для совместной нагрузки пока не измерены.

## Команды владельца для консолей VPS

Откройте веб-консоль нужной машины у её провайдера. На каждом VPS выполните следующий блок. В приватный ответ достаточно прислать listen-порт и SHA256 ED25519, для FirstVDS также свободную память, диск, CPU и статус таймера. Учётные данные, содержимое конфигураций и полные сетевые таблицы оставьте на сервере.

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
sudo sshd -T | awk '$1 == "port" || $1 == "listenaddress"'
sudo ss -lntp | grep -E 'sshd|systemd'
systemctl status ssh.socket ssh.service --no-pager --lines=0
```

Для FirstVDS в той же консоли:

```bash
cat /etc/os-release
nproc
free -m
df -hT /
df -i /
lsblk -o NAME,TYPE,SIZE,FSTYPE
uptime
vmstat 5 12
sudo docker stats --no-stream --format '{{.Name}} {{.MemUsage}} {{.CPUPerc}}'
systemctl is-enabled backup-offsite.timer
systemctl list-timers --all backup-offsite.timer
systemctl show backup-offsite.service -p Result -p ExecMainStatus -p ExecMainExitTimestamp
```

Если SSH слушает, а подключение зависает, посмотреть разрешённые источники для этого порта в firewall панели и в действующем firewall ОС (`sudo ufw status verbose` либо `sudo nft list ruleset`). Уточнить штатный jump host, VPN или разрешённую сеть. При использовании socket activation фактический listener проверяется через ss. Для Timeweb после сверки ключа выполнить команды сети из таблицы. С точкой доступа, уже разрешённой в firewall, проверить `ssh -p ПОРТ ПОЛЬЗОВАТЕЛЬ@АДРЕС` и сравнить предъявленный отпечаток с консольным.
