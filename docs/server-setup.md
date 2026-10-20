# Конфигурация виртуальной машины devops-vm

Документ описывает состояние VM после практической работы № 5. Используется для восстановления со снимка `01-clean-install` (≈10 мин) и как основа для Ansible (ПР № 11).

## 1. Параметры машины

| Параметр | Значение |
|----------|----------|
| Имя в VirtualBox | devops-vm |
| ОС | Ubuntu Server 24.04 LTS |
| RAM | 2048 МБ |
| CPU | 2 ядра |
| Диск | 25 ГБ, динамический |

## 2. Сетевые интерфейсы

| Адаптер | Тип | Адрес (типичный) | Назначение |
|---------|-----|------------------|------------|
| 1 | NAT | 10.0.2.15/24 | Доступ в интернет (`apt`), проброс SSH с хоста |
| 2 | Host-only | 192.168.56.101/24 | Доступ с Windows: HTTP/HTTPS (`devops.local`), ping |

Проверка: `ip -brief address`. Адрес Host-only может отличаться — зафиксировать фактический и обновить `hosts` на Windows.

## 3. Правило проброса портов

Адаптер **NAT**, проброс TCP:

| Поле | Значение |
|------|----------|
| Имя правила | ssh |
| IP хоста | 127.0.0.1 |
| Порт хоста | 2222 |
| Порт гостя | **2222** (после hardening SSH; до hardening было 22) |

Подключение с Windows: `ssh -p 2222 devops@127.0.0.1` или `ssh devops` (см. `~/.ssh/config` на хосте).

## 4. Учётные записи

| Пользователь | Группы | Аутентификация |
|--------------|--------|----------------|
| student | sudo | Пароль (учётная запись установки) |
| devops | sudo | Только SSH-ключ (Ed25519), парольный вход отключён на уровне sshd |

Создание devops (если восстанавливаем с чистого снимка):

```bash
sudo adduser devops
sudo usermod -aG sudo devops
```

Открытый ключ с хоста — в `/home/devops/.ssh/authorized_keys`, права `700` на `~/.ssh`, `600` на `authorized_keys`.

## 5. Служба SSH

- **Порт:** 2222 (не 22).
- **Дополнительный конфиг:** `/etc/ssh/sshd_config.d/99-hardening.conf` (основной `sshd_config` не править).

Содержимое `99-hardening.conf`:

```text
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers devops
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Если есть `/etc/ssh/sshd_config.d/50-cloud-init.conf` с `PasswordAuthentication yes` — строку закомментировать.

**Служба и socket (Ubuntu 24.04):**

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo sshd -t && sudo systemctl restart ssh
```

Проверка: `sudo ss -tlnp | grep sshd` → `0.0.0.0:2222`.

Резервная копия: `/etc/ssh/sshd_config.backup`.

## 6. Правила межсетевого экрана

Порядок важен: правила **до** `ufw enable`.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw delete allow 2222/tcp
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw logging medium
```

Не использовать `ufw allow ssh` — это порт 22 из `/etc/services`, не актуальный порт SSH.

Проверка: `sudo ufw status verbose` — Status: active, Default: deny (incoming), правило 2222 с LIMIT.

## 7. Снимки состояния

| Имя | Когда создавать |
|-----|-----------------|
| 01-clean-install | После установки ОС, `apt upgrade`, работающего SSH, до hardening |
| 02-keys-configured | Пользователь devops, ключ, `ssh devops` с хоста без пароля |
| 03-ssh-hardened | После шага 4.2 ПР № 5 (hardening + проброс гостя 2222) |

VM выключать перед снимком: `sudo poweroff`.

## 8. Имя devops.local (хост)

**Windows:** `C:\Windows\System32\drivers\etc\hosts` (от администратора):

```text
192.168.56.101 devops.local
```

**VM:**

```bash
sudo hostnamectl set-hostname devops-vm
sudo sed -i 's/^127.0.1.1.*/127.0.1.1 devops-vm.devops.local devops-vm/' /etc/hosts
```

## 9. SSH на хосте (Windows)

`%USERPROFILE%\.ssh\config`:

```text
Host devops devops.local
    HostName 127.0.0.1
    Port 2222
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes
```

Для входа по имени через Host-only (без NAT) — отдельный блок с `HostName devops.local` и тем же ключом (см. ПР № 5, задание 4).

## 10. Контроль после восстановления

На VM из каталога репозитория:

```bash
bash scripts/audit.sh
echo $?   # ожидается 0
```

С хоста: `ssh devops`, `ping devops.local`.
