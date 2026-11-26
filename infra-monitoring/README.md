# Мониторинг Fedora 44 в WSL

Один хост запускает контейнеры и поставляет данные. Node Exporter читает метрики ядра и файловой системы Fedora в WSL. Blackbox Exporter проверяет HTTP-сервисы. Prometheus хранит метрики и отправляет тревоги в Alertmanager. Alloy читает journal этой же Fedora и отправляет записи в Loki. Grafana показывает оба источника.

## 1. Подготовь WSL

Используй Docker Engine, запущенный **внутри этой Fedora**, и плагин Docker Compose. В примере `network_mode: host` означает сетевое пространство этой Linux-среды. Установка Docker Engine описана в [документации Docker для Fedora](https://docs.docker.com/engine/install/fedora/).

Если `ps -p 1 -o comm=` показывает `systemd`, переходи к командам ниже. Иначе добавь `systemd=true` в секцию `[boot]` файла `/etc/wsl.conf`, сохрани остальные настройки и выполни `wsl --shutdown` из PowerShell. Затем снова открой Fedora. [Документация Microsoft](https://learn.microsoft.com/en-us/windows/wsl/systemd)

~~~bash
sudo systemctl enable --now docker
sudo docker compose version
sudo docker context show
sudo journalctl -n 5 --no-pager
~~~

Стенд собирает Linux-журнал и Linux-метрики WSL. Корневой диск в дашборде соответствует файловой системе дистрибутива. Интерфейсы и потребление CPU/RAM относятся к Linux-среде. Закрытие всей WSL-среды останавливает сбор.

## 2. Размести файлы

Скопируй каталог `infra-monitoring` из проекта Obsidian в домашний каталог Fedora. При копировании через Windows путь начинается с `/mnt/c/Users/Asik/Documents/IT_Knowledge_OS`. Запускай Compose из Linux-каталога `~/infra-monitoring`.

| Путь внутри `~/infra-monitoring` | Назначение |
| --- | --- |
| `compose.yaml` | Семь контейнеров, монтирование конфигов, перезапуск и volumes |
| `.env.example` и `.gitignore` | Шаблон пароля Grafana и исключение локального `.env` из Git |
| `prometheus/prometheus.yml` | Куда ходить за метриками и куда отправлять алерты |
| `prometheus/rules/host.yml` | Тревоги CPU, RAM, диска и доступности |
| `blackbox/blackbox.yml` | Модуль проверки HTTP |
| `alertmanager/alertmanager.yml` | Группировка и маршрутизация оповещений |
| `alloy/config.alloy` | Чтение journald и выделение метки `unit` |
| `loki/loki.yml` | Хранение логов Loki до семи дней |
| `grafana/provisioning/datasources/local.yml` | Автоподключение Prometheus и Loki |
| `grafana/provisioning/dashboards/local.yml` | Автозагрузка дашборда |
| `grafana/dashboards/wsl.json` | Панели CPU, RAM, диска, сети, HTTP и journald |
| `README.md` | Эта инструкция |

Разделение по сервисам упрощает поиск настроек. Версии образов закреплены, чтобы одинаковый Compose использовал одинаковые версии. Данные живут в Docker volumes, а конфигурация — в Git. Пароль остаётся в локальном `.env`.

~~~bash
cp -r /mnt/c/Users/Asik/Documents/IT_Knowledge_OS/Практика/infra-monitoring ~/
cd ~/infra-monitoring
git init
umask 077
cp .env.example .env
vi .env
~~~

Впиши уникальный пароль. Alloy читает журнал от root внутри контейнера, а каталоги journal подключены только для чтения. Node Exporter использует PID и сеть хоста и монтирует корень только для чтения. Все HTTP-интерфейсы слушают loopback. [Node Exporter](https://github.com/prometheus/node_exporter) · [Alloy и journald](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.journal/)

## 3. Запусти и проверь

~~~bash
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
sudo docker compose up -d
sudo docker compose ps
curl -fsS http://127.0.0.1:3100/ready
~~~

Loki может стать ready через несколько секунд. При ошибке смотри `sudo docker compose logs --tail=80 loki alloy prometheus`. Если порты заняты другим стендом, посмотри `sudo ss -ltnp`. Если дистрибутив использует SELinux Enforcing, проверь отказ в журнале перед изменением монтирований. Корневую ФС и каталоги journald не перемечай для контейнеров.

Открой Grafana на `http://localhost:3000` в Windows и войди как `admin`. Сначала из Fedora проверь `curl -fsS http://127.0.0.1:3000/api/health`, если браузер не подключается. Prometheus показывает сбор на `http://localhost:9090/targets`, а Alertmanager — события на `http://localhost:9093`. На странице Targets проверь обе цели. Сетевым графикам нужен минимум второй опрос. Графики используют среднюю скорость за пять минут.

Для проверки журнала выполни `logger -t monitoring-practice "Проверка доставки из Fedora WSL"`. В Grafana открой **Explore → Loki**, выбери последние 15 минут и запусти запрос:

~~~logql
{job="fedora-journal"} |= "Проверка доставки из Fedora WSL"
~~~

Журнал SSH доступен по запросу `{job="fedora-journal", unit="sshd.service"}`. Для сверки сравни его с `sudo journalctl -u sshd.service -n 10`. Если метки unit нет, найди запись по её сообщению в общем потоке. Alloy при первом чтении забирает до 24 часов доступных записей. Позиция чтения сохраняется в volume.

## 4. Проверь оповещения

Временно добавь в блок `rules:` файла `prometheus/rules/host.yml` правило с тем же отступом, что и остальные:

~~~yaml
      - alert: MonitoringPracticeTest
        expr: vector(1)
        labels:
          severity: info
~~~

Выполни проверку и перезапуск:

~~~bash
sudo docker compose exec prometheus promtool check rules /etc/prometheus/rules/host.yml
sudo docker compose restart prometheus
~~~

Дождись появления тревоги в Prometheus на `/alerts` и затем в Alertmanager. Приёмник `local` оставляет оповещения в интерфейсе. Чтобы доставлять их наружу, добавь в этот приёмник `email_configs` и SMTP-параметры в `global` либо подключи свой webhook. После изменения проверь `sudo docker compose exec alertmanager amtool check-config /config/alertmanager.yml` и перезапусти Alertmanager. [Настройки получателей](https://prometheus.io/docs/alerting/latest/configuration/)

Удали тестовое правило, снова проверь конфигурацию и перезапусти Prometheus. Для проверки отдельного веб-сервиса добавь его URL в список Blackbox в `prometheus/prometheus.yml` и перезапусти Prometheus. В учебном варианте уже проверяется Grafana.

Для остановки выполни `sudo docker compose down`. Volume сохраняет историю метрик, журналы и состояние Grafana. Флаг `-v` удаляет эти данные.
