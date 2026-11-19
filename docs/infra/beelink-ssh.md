# Первый вход на домашний Beelink

Понадобятся монитор с клавиатурой у Beelink на время настройки, Ethernet, MacBook и учётная запись Tailscale с MFA. Ubuntu уже установлена. Домашний роутер оставьте без проброса портов. Подготовьте рабочий tailnet и согласуйте стоимость коммерческого использования до подключения платного плана.

1. **На MacBook** создайте отдельный ключ с парольной фразой. Если такой файл уже существует, выберите другое имя. Скопируйте только файл с суффиксом .pub на флешку.

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/beelink_admin -C beelink-owner
   ssh-keygen -lf ~/.ssh/beelink_admin.pub
   ```

2. **На локальной консоли Beelink** войдите обычным пользователем Ubuntu с sudo. Запишите результат `whoami`. Скопируйте публичный файл с флешки в свою домашнюю папку, сравните отпечаток с Mac. Установите ключ от своего пользователя.

   ```bash
   whoami
   cat /etc/os-release
   ssh-keygen -lf ~/beelink_admin.pub
   install -d -m 700 ~/.ssh
   cat ~/beelink_admin.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

3. **На Beelink** установите клиент Tailscale по официальной инструкции для определённой на предыдущем шаге Ubuntu. Вариант с установщиком ниже сохраняет скрипт для просмотра. При неподдерживаемом релизе остановитесь и используйте резервный маршрут из архитектуры. Для этого пилота удобен вход по tailnet IPv4 без изменения DNS.

   ```bash
   sudo apt update
   sudo apt install curl ufw
   curl -fsSLo /tmp/beelink-tailscale-install.sh https://tailscale.com/install.sh
   less /tmp/beelink-tailscale-install.sh
   sudo sh /tmp/beelink-tailscale-install.sh
   sudo systemctl enable --now tailscaled
   sudo tailscale up --accept-dns=false --accept-routes=false
   tailscale ip -4
   ```

4. **На MacBook** установите Tailscale, войдите в тот же tailnet. В админке ограничьте доступ с устройства Mac на TCP/22 Beelink и удалите перекрывающий широкий allow-all в новом tailnet. Запишите tailnet IPv4 обоих устройств. На **локальной консоли Beelink**, пока она остаётся доступной, подставьте IPv4 Mac в команду ниже. Для свежей машины проверьте `sudo ufw status verbose` и `sudo ufw show added`: широкого разрешения для SSH быть не должно. Эта настройка включает локальный firewall.

   ```bash
   read -rp 'Tailnet IPv4 Mac: ' mac_tail_ip
   sudo ufw default deny incoming
   sudo ufw allow in on tailscale0 from "$mac_tail_ip" to any port 22 proto tcp
   sudo ufw enable
   sudo apt install openssh-server
   sudo systemctl enable --now ssh
   sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   ```

5. **На MacBook** замените поля в угловых скобках своими значениями. Сверьте отпечаток из запроса SSH с экраном Beelink перед подтверждением.

   ```text
   ssh -o IdentitiesOnly=yes -o PreferredAuthentications=publickey -i ~/.ssh/beelink_admin <пользователь-Ubuntu>@<tailnet-IPv4-Beelink>
   ```

   После успешного входа проверьте `hostname` и `sudo -v`. На локальной консоли отключите парольный SSH, проверьте эффективные параметры и откройте второе подключение с Mac, сохраняя первое открытым.

   ```bash
   sudo tee /etc/ssh/sshd_config.d/00-beelink-keys.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
   sudo sshd -t && sudo systemctl reload ssh
   sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
   ```

6. **Передача управления.** Переключите Mac на мобильный интернет, проверьте повторный SSH, затем после перезагрузки Beelink повторите проверку. На Mac откройте VS Code → Remote-SSH → тот же пользователь и адрес (можно сохранить Host в ~/.ssh/config с IdentityFile). Откройте рабочий каталог на Beelink, начните там сессию агента и поручите инвентаризацию из этого плана. Агент в такой сессии уже работает на Beelink. Для продолжения именно с Hetzner потребуется отдельно согласованный путь доступа: адрес из tailnet двух устройств с Hetzner недоступен. Приватные ключи и пароли остаются у владельца. Если SSH не проходит, проверьте Tailscale status, grant, адрес Mac в UFW, `systemctl status ssh` и `ss -lnt` на локальной консоли.

[Установка Tailscale](https://tailscale.com/docs/install/linux) · [OpenSSH в Ubuntu](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
