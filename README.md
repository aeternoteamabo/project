# Fedora 44 Monitoring

Монорепозиторий для подготовки Fedora 44 через Ansible и развёртывания стека мониторинга через Docker Compose. Учебный сценарий проверен на отдельной Fedora 44 в WSL 2 с Docker Desktop. GitHub Actions проверяет изменения в pull request и запускает ручной Deploy в лабораторию через Tailscale.

## Как устроен проект

| Каталог или файл | Назначение |
|---|---|
| `ansible/playbooks/bootstrap/main.yml` | Первичная подготовка Fedora. Первый play подключается под `root`, второй — под `adminops`. |
| `ansible/playbooks/deploy/main.yml` | Развёртывание Docker-стека и Nginx под `adminops` с проверкой Grafana. |
| `ansible/inventory/` | Адреса, SSH-параметры и настройки окружений. `fedora44-clean.ini` используется для лаборатории. |
| `ansible/roles/` | Роли для подготовки пакетов и пользователей, firewall, SSH, Fail2ban, Docker, мониторинга и Nginx. |
| `ansible/requirements.txt` | Версия Ansible Controller для локальной работы и CI. |
| `ansible/collections/requirements.yml` | Версии используемых коллекций Ansible. |
| `infra-monitoring/` | Compose-файлы, настройки сервисов, правила Prometheus, provisioning и дашборд Grafana. |
| `.github/workflows/ci.yml` | Проверка синтаксиса Ansible и конфигурации Compose. |
| `.github/workflows/check-lab-ssh.yml` | Ручная проверка маршрута, SSH-входа и `sudo` из GitHub Actions. |
| `.github/workflows/deploy-lab.yml` | Ручной Deploy в лабораторное окружение. |

Оставляйте каталоги `ansible/` и `infra-monitoring/` рядом. Роль `monitoring` читает исходные файлы на контроллере и копирует их на целевой хост в `/opt/infra-monitoring`. Nginx развёртывается отдельным Compose-проектом в `/opt/nginx-proxy`.

| Компонент | Что делает |
|---|---|
| Node Exporter | Экспортирует системные метрики. |
| Blackbox Exporter | Проверяет доступность HTTP-целей. |
| Prometheus | Собирает метрики и вычисляет правила оповещений. |
| Alertmanager | Принимает оповещения от Prometheus и маршрутизирует их по своей конфигурации. |
| Alloy | Собирает журналы и передаёт их в Loki. |
| Loki | Хранит журналы и обслуживает запросы к ним. |
| Grafana | Показывает метрики Prometheus и журналы Loki. Данные Grafana хранятся в Docker volume. |
| Nginx | Проксирует HTTP-запросы к Grafana через общую сеть Docker. Порт хоста по умолчанию — `8080`. |

## Предварительные условия

- **Контроллер** — Linux или WSL с Python 3.12, 3.13 или 3.14 и SSH-клиентом. Управляющая учётная запись может называться как угодно.
- **Чистая цель** — Fedora 44 с рабочим `systemd`, Python 3 и доступом к репозиториям пакетов. Для первого Bootstrap нужен уже работающий `sshd` и вход под `root` по начальному ключу на `2222/tcp`.
- **Docker на цели** — доступный движок и плагин Compose. В лаборатории их предоставляет Docker Desktop с включённой WSL Integration для целевого дистрибутива. Текущая роль `docker` настраивает доступ и проверяет Docker. Установку Docker Engine на отдельной Fedora нужно выполнить отдельно.
- **Связь с целью** — адрес, доступный с контроллера, и проверенный ключ SSH-сервера в `known_hosts`. `127.0.0.1` в лабораторном inventory подходит для нашей WSL-схемы. Для удалённой Fedora укажите её фактический адрес.
- **Firewalld** — проверьте, что зона в переменных ролей соответствует интерфейсу, через который приходит SSH. В лаборатории использовалась `public`.

Если контроллер и цель находятся в двух WSL-дистрибутивах на одном компьютере, проверьте владельца порта `2222`. В испытанной конфигурации дистрибутивы разделяли сетевое пространство, поэтому для лаборатории отдельно передавали управление SSH и firewalld. Docker Desktop тоже может быть общим. Лаборатория использует собственные имена Compose-проекта, тома и сети.

## Подготовка контроллера

Клонируйте репозиторий в файловую систему Linux или WSL и откройте его каталог `ansible/`:

```bash
git clone https://github.com/aeternoteamabo/project.git fedora-monitoring
cd fedora-monitoring/ansible

python3 -m venv "$HOME/.venvs/fedora-monitoring"
source "$HOME/.venvs/fedora-monitoring/bin/activate"
python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

Проект проверялся с `ansible-core 2.20.7`, `ansible.posix 2.2.2` и `community.docker 5.4.1`. Локально использовался Docker Compose `v5.6.1`. CI выводит версию Compose, установленную на GitHub-hosted runner.

### Inventory и SSH

Для своей цели подготовьте inventory в каталоге `ansible/inventory/`. Сохраните группу `monitoring`. Если меняете псевдоним хоста `mon-server`, используйте новый псевдоним и в `host_vars`, и в аргументе `--limit`.

```ini
[monitoring]
mon-server ansible_host=192.0.2.10 ansible_port=2222 ansible_user=adminops

[monitoring:vars]
monitoring_compose_project=infra-monitoring-lab
monitoring_docker_network=monitoring_lab_net
```

`192.0.2.10` — адрес для примера. Параметры `HostKeyAlias` и алгоритм ключа из лабораторного inventory относятся к проверенной лаборатории. Для другого сервера настройте проверку его собственного ключа SSH. Перед Bootstrap проверьте начальный вход под `root` по ключу, полученному при подготовке хоста.

| Ключ на контроллере | Назначение |
|---|---|
| `~/.ssh/fedora44_root_bootstrap` | Начальный SSH-вход под `root`. Путь можно переопределить переменной `bootstrap_private_key_file`. |
| `~/.ssh/fedora44_adminops.pub` | Публичный ключ, который Bootstrap добавляет для `adminops`. Путь можно переопределить через `adminops_public_key_file`. |
| `~/.ssh/fedora44_adminops` | Приватный ключ для локального Deploy. Настраивается в `inventory/host_vars/mon-server/connection.yml`. |
| Отдельный SSH-ключ для GitHub Actions | Публичная часть добавляется в `authorized_keys` пользователя `adminops`, приватная часть хранится в секрете окружения `lab`. |

### Локальный Ansible Vault

Рабочий `ansible/inventory/host_vars/mon-server/vault.yml` исключён из публичного репозитория. Создайте его локально и храните файл расшифровки вне репозитория. Для существующего Vault укажите путь к его файлу расшифровки и используйте `ansible-vault edit`.

```bash
export ANSIBLE_VAULT_PASSWORD_FILE="$HOME/.config/ansible/mon-server.vault-pass"
# Один раз, если файл пароля ещё не создан.
install -d -m 700 "$HOME/.config/ansible"
(
  umask 077
  set -o noclobber
  python3 -c 'import secrets; print(secrets.token_hex(32))' > "$ANSIBLE_VAULT_PASSWORD_FILE"
)
ansible-vault create inventory/host_vars/mon-server/vault.yml
```

| Переменная Vault | Для чего нужна |
|---|---|
| `ansible_become_password` | Пароль `sudo` пользователя `adminops`. |
| `adminops_password_hash` | Хеш того же пароля в формате Linux для создания нового пользователя. Пароль существующего `adminops` роль сохраняет. |
| `vault_grafana_admin_password` | Начальный пароль администратора Grafana для пустой базы. |

Перед коммитом проверьте, что рабочий `vault.yml` игнорируется Git и остаётся зашифрованным. Приватные SSH-ключи, пароль расшифровки Vault и временные файлы с открытым паролем Grafana хранятся вне истории Git. `ansible-vault edit` показывает расшифрованный YAML в редакторе и сохраняет его обратно зашифрованным.

## Первый запуск на чистой Fedora

Все команды Ansible в этом разделе выполняются из `ansible/` на контроллере. Пример использует `inventory/fedora44-clean.ini`.

### 1. Bootstrap

В проверенном сценарии Bootstrap запускался по этапам:

```bash
for stage in packages services accounts; do
  ansible-playbook -i inventory/fedora44-clean.ini \
  playbooks/bootstrap/main.yml --limit mon-server --tags "$stage" || break
done
```

- `packages` проверяет вход под `root`, обновляет RPM-пакеты и устанавливает OpenSSH, firewalld и Fail2ban.
- `services` сохраняет разрешение для начального SSH-порта перед запуском firewalld, включает автозапуск SSH и firewall.
- `accounts` создаёт `adminops`, добавляет его в `wheel`, устанавливает публичный ключ и задаёт `umask 007` для Bash.

Если этап завершился ошибкой, исправьте первую упавшую задачу перед продолжением. После успешного `accounts` проверьте новый вход под `adminops` и повышение прав:

```bash
ansible -i inventory/fedora44-clean.ini mon-server \
  --ssh-extra-args='-S none' -m ansible.builtin.command -a 'id -un'

ansible -i inventory/fedora44-clean.ini mon-server \
  --ssh-extra-args='-S none' -b -m ansible.builtin.command -a 'id -un'
```

Ожидаемые результаты — `adminops` и `root`. Затем примените финализацию:

```bash
ansible-playbook -i inventory/fedora44-clean.ini \
  playbooks/bootstrap/main.yml --limit mon-server --tags bootstrap_finalize
```

Второй play снова проверяет вход и `sudo`, затем настраивает firewalld, применяет и проверяет конфигурацию `sshd` и запускает SSH-jail Fail2ban. Откройте ещё одно SSH-соединение под `adminops` и убедитесь, что начальный ключ `root` теперь отклоняется. Для дальнейших обновлений запускайте Deploy.

### 2. Deploy

Убедитесь, что Docker доступен на целевой Fedora. В WSL включите интеграцию Docker Desktop именно для целевого дистрибутива. Затем запустите:

```bash
ansible-playbook -i inventory/fedora44-clean.ini \
  playbooks/deploy/main.yml --limit mon-server
```

Playbook проверяет `adminops` и `sudo`, затем выполняет роли `docker`, `monitoring` и `nginx`. При необходимости Docker-проверку можно запустить отдельно с `--tags docker`. В конце Deploy обращается к `http://127.0.0.1:8080/api/health` **на целевом хосте** через Nginx и ожидает HTTP `200` и `database: ok`.

При первом старте с пустым томом роль `monitoring` создаёт временный файл из Vault и добавляет `compose.bootstrap.yaml`. Grafana читает пароль из файла через `GF_SECURITY_ADMIN_PASSWORD__FILE`. Роль проверяет вход `admin`, пересоздаёт Grafana с обычным Compose-файлом и повторно проверяет вход. Временный файл удаляется, а база сохраняется в томе. При повторном Deploy с готовой базой начальный пароль Grafana в контейнер не передаётся.

Для лаборатории используются Compose-проект `infra-monitoring-lab`, том `infra-monitoring-lab_grafana` и сеть `monitoring_lab_net`. Перед тестом на общем Docker Desktop проверьте, что выбран отдельный том. После прерванной инициализации роль использует маркер `.grafana-bootstrap-in-progress` для повторной попытки. Если база уже создана с другим паролем, выясните её принадлежность и состояние прежде, чем повторять тест.

### 3. Проверка результата

Из Ansible можно проверить доступность Grafana и состояние контейнеров:

```bash
ansible -i inventory/fedora44-clean.ini mon-server -m ansible.builtin.uri   -a 'url=http://127.0.0.1:8080/api/health status_code=200'

ansible -i inventory/fedora44-clean.ini mon-server -m ansible.builtin.command   -a 'docker compose -f /opt/infra-monitoring/compose.yaml -p infra-monitoring-lab ps -a'
```

В лаборатории Grafana открывается через [http://localhost:8080](http://localhost:8080), если порт доступен через WSL. HTTP-проверка подтверждает работу Nginx, Grafana и подключение к базе. Для проверки сбора данных откройте в Grafana источники Prometheus и Loki и убедитесь, что видны свежие метрики и журналы. После развёртывания повторите тот же Deploy. На подготовленном стенде ожидается `changed=0`, `failed=0` и `unreachable=0`.

## CI/CD в GitHub Actions

### CI

`CI` запускается на pull request и при push в `main`. Runner устанавливает зависимости и проверяет синтаксис обоих playbook с временным inventory. Затем запускает `docker compose config --quiet` для обычной конфигурации и для конфигурации с bootstrap-файлом. Проверка **Validate Ansible and Compose** обязательна для слияния PR в `main`.

### Доступ runner к лаборатории

В GitHub используется окружение `lab` с разрешённой веткой `main`. Лабораторная Fedora состоит в tailnet и помечена как `tag:fedora-lab`. Временный GitHub-hosted runner входит в tailnet через OIDC с тегом `tag:github-deploy`. Правило Tailscale разрешает этому тегу `2222/tcp` на `tag:fedora-lab`. Для фактического ограничения проверьте, что другое правило tailnet не выдаёт runner более широкий доступ.

В Tailscale создаётся federated identity для GitHub Actions с точным OIDC `Subject` репозитория и окружения `lab`. Для тега `tag:github-deploy` нужен scope **Auth Keys → Write** (`auth_keys`). SSH-аутентификацию выполняет OpenSSH в Fedora. Пока идёт Deploy, целевая WSL, Docker Desktop и Tailscale должны работать.

| Имя в окружении `lab` | Тип | Значение |
|---|---|---|
| `LAB_TAILSCALE_IP` | Variable | IPv4 лабораторной Fedora в tailnet. |
| `LAB_SSH_KNOWN_HOSTS` | Variable | Строка `known_hosts` с заранее проверенным публичным ключом сервера и псевдонимом `fedora44-ansible-lab`. |
| `TS_OAUTH_CLIENT_ID` | Secret | Client ID федеративной записи Tailscale. |
| `TS_AUDIENCE` | Secret | Audience этой же записи. |
| `LAB_SSH_PRIVATE_KEY` | Secret | Отдельный приватный ключ CI для `adminops`. |
| `LAB_BECOME_PASSWORD` | Secret | Пароль `sudo` пользователя `adminops` в лаборатории. |
| `LAB_GRAFANA_ADMIN_PASSWORD` | Secret | Начальный пароль Grafana. Нужен при первом запуске с пустой базой. |

Рабочий локальный Vault и пароль его расшифровки остаются на контроллере. Workflow собирает временные параметры Ansible из секретов окружения `lab` и использует строгую проверку SSH-ключа сервера. Начальный ключ `root` предназначен для локального Bootstrap.

### Запуск

1. Создайте ветку, отправьте pull request и дождитесь зелёного CI.
2. Слейте PR в `main`.
3. При первом подключении или после изменения сети откройте **Actions → Check lab SSH → Run workflow** и выберите `main`. Ожидайте SSH-баннер, успешный вход как `adminops` и `root` от проверки `sudo`.
4. Откройте **Actions → Deploy lab → Run workflow** и выберите `main`. Workflow ограничивает одновременные Deploy в лабораторию.
5. В логе Ansible проверьте `failed=0`, `unreachable=0` и успешную задачу проверки Grafana через Nginx.

## Диагностика

| Симптом | Что проверить |
|---|---|
| `Could not match supplied host pattern` | Текущий каталог `ansible/`, путь после `-i`, группу `monitoring` и псевдоним хоста. Команда `ansible-inventory -i inventory/fedora44-clean.ini --graph` покажет состав групп. |
| `Attempting to decrypt but no vault secrets found` | Для локального запуска проверьте `ANSIBLE_VAULT_PASSWORD_FILE` или передайте `--vault-password-file`. |
| `Connection timed out` в Actions | Лаборатория запущена, Tailscale подключён, `sshd` слушает нужный адрес на `2222`, в правилах Tailscale разрешён TCP-доступ и firewall пропускает соединение. |
| `403` при создании Tailscale auth key | Проверьте scope **Auth Keys → Write** и тег федеративной записи. |
| `Host key verification failed` | Сверьте отпечаток ключа через доверенную консоль Fedora и обновите `LAB_SSH_KNOWN_HOSTS` после подтверждения. |
| `Permission denied` у `docker.sock` | Проверьте Docker Desktop Integration и членство `adminops` в группе `docker`. После смены групп нужно новое SSH-соединение. |
| `401` при первом входе Grafana | Проверьте, что контейнер создан с обоими Compose-файлами, видит секрет и использует нужный том. Уже инициализированная база сохраняет пароль пользователя. |

## Что проверено

- На отдельной Fedora 44 в WSL прошли этапы Bootstrap. Проверены создание `adminops`, его SSH-вход и `sudo`, а также отказ прямого входа под `root` после настройки.
- Grafana стартовала с пустым лабораторным томом. Вход с паролем из Vault проверен до и после пересоздания контейнера.
- Повторный локальный Deploy на лабораторной Fedora завершился с `changed=0`. Проверка Grafana через Nginx прошла.
- В GitHub Actions проверены CI, обязательная проверка PR, SSH и `sudo` через Tailscale и ручной Deploy. Повторный запуск ручного Deploy подтверждён.
- Bootstrap проверялся по этапам. Сценарий на отдельной Fedora с Docker Engine и запуск Bootstrap одной командой на чистой цели отдельно не проверялись.

