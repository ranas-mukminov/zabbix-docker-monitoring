# Модуль мониторинга Docker для Zabbix (форк run-as-daemon от monitoringartist/zabbix-docker-monitoring)

Загружаемый C-модуль + шаблоны для мониторинга Docker и контейнерных окружений с помощью Zabbix.

[Русский] | [English](README.md)

---

## Обзор

Этот проект предоставляет высокопроизводительный **модуль Zabbix для Docker** (загружаемый модуль на языке C для Zabbix agent) в комплекте с готовыми шаблонами Zabbix и интеграцией с Docker для комплексного мониторинга контейнеров. Он позволяет собирать метрики Docker-контейнеров — CPU, память, блоковый ввод-вывод, сеть и статусную информацию — напрямую в Zabbix без накладных расходов на внешние скрипты.

**Это форк проекта [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring)**, поддерживаемый и адаптированный **Ранасом Мукминовым** (run-as-daemon.ru) для использования в промышленных средах Zabbix.

Этот проект был изначально создан для более старых версий Zabbix, но теперь предназначен для использования с текущими поддерживаемыми ветками Zabbix (см. [Поддерживаемые версии Zabbix](#поддерживаемые-версии-zabbix--ос-обновлено-ноябрь-2025) ниже).

Ключевые преимущества:
- **Нативная производительность**: Реализован как скомпилированный C-модуль, обеспечивающий сбор метрик в ~10 раз быстрее, чем UserParameter-скрипты
- **Поддержка контейнерных платформ**: Работает с Docker, Kubernetes, Mesos/Marathon/Chronos, Docker Swarm, LXC/LXD и контейнерными окружениями на базе Systemd
- **Готовность к production**: Проверенные в боевых условиях шаблоны и предсобранные бинарные файлы для основных дистрибутивов Linux и версий Zabbix
- **Простое развёртывание**: Доступен в составе Docker-образа Dockbix agent XXL для быстрого старта

## Возможности

- **Комплексные метрики контейнеров**
  - Использование CPU (system, user, total, throttling)
  - Потребление памяти (RSS, cache, swap, лимиты)
  - Статистика блокового ввода-вывода (операции чтения/записи, байты, время обслуживания)
  - Сетевой ввод-вывод (байты, пакеты, ошибки, потери)
  - Время работы контейнера и отслеживание состояния

- **Low-Level Discovery (LLD)**
  - Автоматическое обнаружение запущенных контейнеров
  - Обнаружение опубликованных портов контейнеров
  - Динамическое обновление инвентаря

- **Инвентаризация и метаданные контейнеров**
  - Имена контейнеров, ID и человекочитаемые идентификаторы
  - IP-адреса и конфигурация сети
  - Переменные окружения и метки
  - Информация об образах

- **Агрегированные метрики статуса**
  - Количество контейнеров по статусам (running, exited, crashed, paused)
  - Количество образов по статусам (все, dangling)
  - Количество volumes по статусам (все, dangling)

- **Прямая интеграция с Docker API**
  - Доступ к системной информации `docker info`
  - `docker inspect` для получения детальных метаданных контейнера
  - `docker stats` для статистики использования ресурсов в реальном времени

- **Высокая производительность**
  - Нативная реализация на C устраняет накладные расходы на выполнение скриптов
  - Оптимизированное чтение файловой системы cgroup
  - Минимальное влияние на мониторируемые системы

- **Готовые шаблоны Zabbix**
  - Стандартный шаблон с пассивными проверками
  - Вариант с активными проверками для распределённого мониторинга
  - Специализированный шаблон для Mesos/Marathon/Chronos

- **Интеграция с экосистемой**
  - Совместимость с Docker-образом Dockbix agent XXL
  - Поддержка дашбордов Grafana через источник данных Zabbix
  - Включена политика SELinux

## Архитектура

### Модуль Zabbix Docker

Ядро проекта — это `zabbix_module_docker.so`, загружаемый модуль для Zabbix agent. Модуль:
- Подключается непосредственно к процессу Zabbix agent во время выполнения
- Предоставляет метрики Docker через пользовательские ключи элементов данных (с префиксом `docker.*`)
- Читает метрики контейнеров из файловой системы cgroup для максимальной производительности
- Взаимодействует с Docker через Unix-сокет API для получения метаданных и обнаружения
- Требует минимальной настройки — просто загрузите модуль и используйте шаблоны

### Шаблоны

Предварительно настроенные шаблоны Zabbix предоставляют:
- Прототипы элементов данных с использованием LLD для автоматического мониторинга контейнеров
- Графики для визуализации метрик CPU, памяти, ввода-вывода и сети
- Триггеры для оповещения об изменениях состояния контейнеров и превышении порогов ресурсов
- Макросы для простой настройки

### Интеграция с Docker

Модуль можно развернуть двумя способами:
1. **Нативная установка**: Установка модуля на хостах со стандартным Zabbix agent
2. **Контейнеризованная**: Использование Docker-образа Dockbix agent XXL с предустановленным модулем

```
┌─────────────┐
│ Docker Host │
│             │
│  Container  │ ──┐
│  Container  │   │ cgroups
│  Container  │   │ метрики
└─────────────┘   │
                  ▼
         ┌───────────────┐       ┌──────────────┐       ┌─────────┐
         │ Zabbix Agent  │──────▶│ Zabbix Server│──────▶│ Grafana │
         │  + docker.so  │       │              │       │         │
         └───────────────┘       └──────────────┘       └─────────┘
```

## Быстрый старт

### Использование с Dockbix Agent XXL

Самый быстрый способ начать мониторинг Docker-контейнеров — использовать Docker-образ Dockbix agent XXL:

```bash
docker run \
  --name=dockbix-agent-xxl \
  --net=host \
  --privileged \
  -v /:/rootfs \
  -v /var/run:/var/run \
  --restart unless-stopped \
  -e "ZA_Server=<IP_ZABBIX_СЕРВЕРА>" \
  -e "ZA_ServerActive=<IP_ZABBIX_СЕРВЕРА>" \
  -d monitoringartist/dockbix-agent-xxl-limited:latest
```

Замените `<IP_ZABBIX_СЕРВЕРА>` на адрес вашего Zabbix-сервера.

Затем в Zabbix:
1. Импортируйте шаблон [Zabbix-Template-App-Docker.xml](https://raw.githubusercontent.com/ranas-mukminov/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker.xml)
2. Создайте или настройте узел сети, представляющий ваш Docker-хост
3. Привяжите шаблон "Template App Docker" к узлу
4. Дождитесь начала обнаружения и сбора метрик

Подробнее см. [документацию Dockbix agent XXL](https://github.com/monitoringartist/dockbix-agent-xxl).

### Ручная установка (штатный Zabbix Agent)

Для существующих установок Zabbix agent:

1. **Скачайте модуль** для вашей ОС и версии Zabbix из [таблицы предсобранных бинарных файлов](#совместимость)
   
   Пример для CentOS 7 с Zabbix 6.0:
   ```bash
   wget https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/6.0/zabbix_module_docker.so -O /usr/lib/zabbix/modules/zabbix_module_docker.so
   ```

2. **Настройте Zabbix agent** на загрузку модуля:
   
   Отредактируйте `/etc/zabbix/zabbix_agentd.conf`:
   ```ini
   LoadModulePath=/usr/lib/zabbix/modules
   LoadModule=zabbix_module_docker.so
   ```

3. **Настройте права Docker** (см. [Дополнительные права Docker](#дополнительные-права-docker)):
   ```bash
   usermod -aG docker zabbix
   ```

4. **Перезапустите Zabbix agent**:
   ```bash
   systemctl restart zabbix-agent
   ```

5. **Импортируйте шаблон** в веб-интерфейсе Zabbix и привяжите его к вашему Docker-хосту

6. **Проверьте**, что модуль загружен:
   ```bash
   zabbix_agentd -p | grep docker.modver
   ```

Если вам нужно скомпилировать модуль для неподдерживаемой платформы, см. раздел [Компиляция](#компиляция).

## Шаблоны

В каталоге `template/` доступны три шаблона Zabbix:

### Zabbix-Template-App-Docker.xml (Стандартный)

**Рекомендуется** для большинства случаев использования. Возможности:
- Пассивные проверки (Zabbix-сервер опрашивает агента)
- Автоматическое обнаружение контейнеров каждые 60 секунд
- Метрики CPU, памяти, блокового ввода-вывода, сети
- Мониторинг состояния контейнеров и триггеры
- Предварительно настроенные графики

**Используйте, когда:** У вас стандартный Zabbix agent в пассивном режиме.

### Zabbix-Template-App-Docker-active.xml (Активные проверки)

Аналогичен стандартному шаблону, но использует активные проверки, при которых агент отправляет данные на сервер.

**Используйте, когда:** У вас распределённые среды или среды с NAT, где агенты должны инициировать соединения.

### Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml

Специализированный шаблон для мониторинга **кластера Mesos** с планировщиками Marathon и Chronos. Включает:
- Расширенное обнаружение контейнеров с использованием идентификаторов задач Mesos
- Макросы для конкретных приложений
- Соглашения об именовании Mesos/Marathon

**Используйте, когда:** Мониторинг Docker-контейнеров, оркестрируемых Mesos/Marathon/Chronos.

### Импорт шаблонов

Вы можете импортировать шаблоны вручную через веб-интерфейс Zabbix или использовать автоматизированный Docker-образ:

```bash
docker run --rm \
  -e XXL_apiurl=http://ваш-zabbix-сервер/zabbix \
  -e XXL_apiuser=Admin \
  -e XXL_apipass=zabbix \
  monitoringartist/zabbix-templates
```

## Доступные метрики

Модуль предоставляет всесторонние метрики Docker через различные семейства ключей элементов данных:

### Ключи обнаружения

| Ключ | Описание |
|------|----------|
| `docker.discovery[<par1>,<par2>,<par3>]` | **Обнаружение контейнеров** (LLD). Обнаруживает все запущенные контейнеры. Необязательные параметры позволяют фильтрацию и пользовательское именование с использованием данных docker.inspect. Возвращает макросы: `{#FCONTAINERID}` (полный 64-символьный ID), `{#SCONTAINERID}` (короткий 12-символьный ID), `{#HCONTAINERID}` (человекочитаемое имя), `{#SYSTEM.HOSTNAME}` (имя хоста системы). Пример: `docker.discovery[Config,Env,MESOS_TASK_ID=]` для контейнеров Mesos. |
| `docker.port.discovery[cid,<protocol>]` | **Обнаружение портов** (LLD). Обнаруживает опубликованные порты контейнера. Протокол: `tcp`, `udp` или `all` (по умолчанию). |

### Метрики ресурсов

| Ключ | Описание |
|------|----------|
| `docker.cpu[cid,cmetric]` | **Метрики CPU** из cpuacct.stat и cpu.stat. Метрики: `system`, `user`, `total`, `nr_throttled`, `throttled_time`. Пример: `docker.cpu[cid,user]`. Примечание: Используйте Delta (скорость в секунду) в предобработке Zabbix для процента использования. |
| `docker.mem[cid,mmetric]` | **Метрики памяти** из memory.stat. Метрики: `cache`, `rss`, `mapped_file`, `swap`, `pgfault`, `pgmajfault`, `inactive_anon`, `active_anon`, `total_rss`, `hierarchical_memory_limit` и другие. Пример: `docker.mem[cid,rss]`. Требуется включённая memory cgroup (параметр ядра `cgroup_enable=memory`). |
| `docker.dev[cid,bfile,bmetric]` | **Метрики блокового ввода-вывода** из файлов blkio cgroup. Примеры: `docker.dev[cid,blkio.io_service_bytes,Total]`, `docker.dev[cid,blkio.io_serviced,'8:0 Sync']`. Некоторые метрики требуют конфигурации ядра `CONFIG_DEBUG_BLK_CGROUP=y`. |
| `docker.xnet[cid,interface,nmetric]` | **Сетевые метрики** (экспериментальная функция, требует root). Интерфейс: имя сетевого интерфейса или `all` для суммы. Метрики: `MTU`, `RX-OK`, `RX-ERR`, `RX-DRP`, `TX-OK`, `TX-ERR` и т.д. Пример: `docker.xnet[cid,eth0,TX-OK]`. Требуется `AllowRoot=1` и установленный netstat. |

### Метаданные контейнера

| Ключ | Описание |
|------|----------|
| `docker.inspect[cid,par1,<par2>,<par3>]` | **Данные docker inspect**. Возвращает значения из JSON docker inspect (например, [API v1.21](https://docs.docker.com/engine/api/v1.21/#inspect-a-container)). Параметры: имена свойств JSON 1-го/2-го/3-го уровня или селекторы массива. Примеры: `docker.inspect[cid,Config,Image]`, `docker.inspect[cid,NetworkSettings,IPAddress]`, `docker.inspect[cid,Name]`, `docker.inspect[cid,Config,Env,MESOS_TASK_ID=]`. Возвращает только текстовые/числовые значения. Требуются дополнительные права Docker. |
| `docker.stats[cid,par1,<par2>,<par3>]` | **Docker stats** использования ресурсов в реальном времени. Требуется Docker 1.5+. Возвращает значения из JSON stats (например, [API v1.21](https://docs.docker.com/engine/api/v1.21/#get-container-stats-based-on-resource-usage)). Примеры: `docker.stats[cid,memory_stats,usage]`, `docker.stats[cid,network,rx_bytes]`, `docker.stats[cid,cpu_stats,cpu_usage,total_usage]`. Самый точный, но медленный метод (0.3-0.7 с на вызов). Требуются дополнительные права Docker. |
| `docker.info[info]` | **Системная информация Docker**. Возвращает значения из JSON docker info (например, [API v1.21](https://docs.docker.com/engine/api/v1.21/#display-system-wide-information)). Примеры: `docker.info[Containers]`, `docker.info[Images]`, `docker.info[NCPU]`. Требуются дополнительные права Docker. |

### Метрики статуса и подсчёта

| Ключ | Описание |
|------|----------|
| `docker.cstatus[status]` | **Количество контейнеров по статусу**. Значения статуса: `All` (все контейнеры), `Up` (работающие, включая приостановленные), `Exited` (остановленные), `Crashed` (код выхода ≠ 0), `Paused` (приостановленные). Требуются дополнительные права Docker. |
| `docker.istatus[status]` | **Количество образов по статусу**. Значения статуса: `All` (все образы), `Dangling` (образы без тегов). Требуются дополнительные права Docker. |
| `docker.vstatus[status]` | **Количество volumes по статусу**. Значения статуса: `All` (все volumes), `Dangling` (неиспользуемые volumes). Требуется Docker API v1.21+ и дополнительные права Docker. |
| `docker.up[cid]` | **Состояние работы контейнера**. Возвращает `1`, если контейнер работает, иначе `0`. Быстрая проверка с использованием файловой системы cgroup. |
| `docker.modver` | **Версия модуля**. Возвращает версию загруженного модуля zabbix_module_docker. |

### Идентификатор контейнера (cid)

Параметр `cid` принимает:
- **Полный ID контейнера** (64 символа): макрос `{#FCONTAINERID}` из обнаружения
- **Короткий ID контейнера** (12 символов): макрос `{#SCONTAINERID}` — должен начинаться с `/`, например `/2599a1d88f75`
- **Человекочитаемое имя**: макрос `{#HCONTAINERID}` — должно начинаться с `/`, например `/zabbix-server`

Примечание: Человекочитаемые имена и короткие ID требуют дополнительных прав Docker.

## Дашборд Grafana

При использовании этого модуля Zabbix доступен пользовательский **дашборд Grafana** для мониторинга Docker. Дашборд предоставляет:
- Использование ресурсов контейнерами в реальном времени
- Анализ исторических трендов
- Представления для сравнения нескольких контейнеров
- Интеграция с плагином источника данных Zabbix

![Дашборд Docker в Grafana](https://raw.githubusercontent.com/monitoringartist/grafana-zabbix-dashboards/master/overview-docker/overview-docker.png)

Получите доступ к дашборду в [репозитории Grafana Zabbix Dashboards](https://github.com/monitoringartist/grafana-zabbix-dashboards).

**Примечание:** Интеграция с Grafana опциональна. Модуль полностью работает внутри самого Zabbix; Grafana предоставляет расширенные возможности визуализации.

## Поддерживаемые версии Zabbix / ОС (Обновлено: ноябрь 2025)

### Рекомендуемые версии Zabbix

Этот модуль рекомендуется использовать с текущими поддерживаемыми ветками Zabbix:

- **Zabbix 7.4** (последняя стабильная версия)
- **Zabbix 7.0 LTS** (долгосрочная поддержка)
- **Zabbix 6.0 LTS** (долгосрочная поддержка)
- Zabbix 7.2 (стандартный релиз, опционально)

**Устаревшие версии** (4.0, 5.0, 5.4) больше не поддерживаются официально и не рекомендуются для использования в production. Для получения актуальной информации о жизненном цикле версий обратитесь к:
- https://www.zabbix.com/download
- https://endoflife.date/zabbix

Предсобранные бинарные файлы доступны для версий Zabbix agent: **4.0, 5.0, 5.4, 6.0** (см. таблицу ниже). Для Zabbix 7.0 и 7.4 вы можете скомпилировать модуль из исходного кода (см. раздел [Компиляция](#компиляция)).

### Рекомендуемые дистрибутивы Linux

Типовые цели Linux, протестированные с этим модулем:
- **Debian 12** (Bookworm)
- **Ubuntu 22.04 LTS** (Jammy) / **Ubuntu 24.04 LTS** (Noble)
- **Rocky Linux 9** / **AlmaLinux 9** / **RHEL 9**
- **Docker Engine 23.x / 24.x**
- **Docker Compose v2**

Для получения точной информации о поддерживаемых ОС и комбинациях баз данных проверьте официальную документацию Zabbix:
- https://www.zabbix.com/download
- https://www.zabbix.com/requirements

### Загрузка предсобранных бинарных файлов

| Дистрибутив | Zabbix 6.0 | Zabbix 5.4 | Zabbix 5.0 | Zabbix 4.0 |
|-------------|:----------:|:----------:|:----------:|:----------:|
| Amazon Linux 2 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/4.0/zabbix_module_docker.so) |
| Amazon Linux 1 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/4.0/zabbix_module_docker.so) |
| CentOS 7 / RHEL 7 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/4.0/zabbix_module_docker.so) |
| Debian 11 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/4.0/zabbix_module_docker.so) |
| Debian 10 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/4.0/zabbix_module_docker.so) |
| Debian 9 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/4.0/zabbix_module_docker.so) |
| Fedora 35 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/4.0/zabbix_module_docker.so) |
| Fedora 34 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/4.0/zabbix_module_docker.so) |
| Fedora 33 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/4.0/zabbix_module_docker.so) |
| openSUSE 15 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/4.0/zabbix_module_docker.so) |
| openSUSE 42 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/4.0/zabbix_module_docker.so) |
| Ubuntu 20.04 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/4.0/zabbix_module_docker.so) |
| Ubuntu 18.04 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/4.0/zabbix_module_docker.so) |
| Ubuntu 16.04 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/4.0/zabbix_module_docker.so) |
| Ubuntu 14.04 | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/6.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/5.4/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/5.0/zabbix_module_docker.so) | [Скачать](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/4.0/zabbix_module_docker.so) |

**Примечание:** Для более новых версий Zabbix или дистрибутивов, не указанных в списке, вы можете скомпилировать модуль из исходного кода. См. [Компиляция](#компиляция).

## Дополнительные права Docker

Модулю требуется доступ к Unix-сокету Docker для использования определённых функций (обнаружение, inspect, info, stats). Выберите один из этих вариантов:

### Вариант 1: Добавить пользователя Zabbix в группу Docker (Рекомендуется)

```bash
usermod -aG docker zabbix
# Перезапустите Zabbix agent, чтобы изменения группы вступили в силу
systemctl restart zabbix-agent
```

### Вариант 2: Запуск Zabbix Agent от имени root

Отредактируйте `/etc/zabbix/zabbix_agentd.conf`:
```ini
AllowRoot=1
```

**Примечание:** Это требуется при использовании Docker из репозиториев RHEL/CentOS.

### Настройка SELinux

Если SELinux находится в режиме enforcing (`getenforce`), установите предоставленный модуль политики SELinux:

```bash
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/selinux/zabbix-docker.te
checkmodule -M -m -o zabbix-docker.mod zabbix-docker.te
semodule_package -o zabbix-docker.pp -m zabbix-docker.mod
semodule -i zabbix-docker.pp
```

Эта политика сохраняется после перезагрузок и разрешает Zabbix agent доступ к Docker.

## Мониторинг логов контейнеров

Для мониторинга логов контейнеров можно использовать стандартный [мониторинг логов Zabbix](https://www.zabbix.com/documentation/current/ru/manual/config/items/itemtypes/log_items). Docker логирует stdout/stderr в:

```
/var/lib/docker/containers/<ПОЛНЫЙ_ID_КОНТЕЙНЕРА>/<ПОЛНЫЙ_ID_КОНТЕЙНЕРА>-json.log
```

Используйте этот ключ элемента данных лога Zabbix для парсинга логов Docker в формате JSON:

```
log[/var/lib/docker/containers/{#FCONTAINERID}/{#FCONTAINERID}-json.log,"\"log\":\"(.*)\",\"stream",,,skip,\1]
```

**Совет:** Чтобы приложения логировали в stdout/stderr, создайте символьные ссылки на файлы логов:
```bash
ln -sf /dev/stdout /var/log/nginx/access.log
ln -sf /dev/stderr /var/log/nginx/error.log
```

Используйте макрос LLD `{#FCONTAINERID}` для автоматического мониторинга логов на основе обнаружения.

## Отладка

### Модуль не загружается

**Симптом:** Лог Zabbix agent показывает ошибки загрузки модуля

**Решения:**
- Проверьте путь к модулю в `zabbix_agentd.conf` соответствует расположению файла
- Проверьте права доступа к файлу модуля: `chmod 644 /путь/к/zabbix_module_docker.so`
- Убедитесь в правильном бинарном файле для вашей ОС/версии Zabbix
- Проверьте лог Zabbix agent: `tail -f /var/log/zabbix/zabbix_agentd.log`
- Проверьте зависимости: `ldd /путь/к/zabbix_module_docker.so`

### Ключи Docker возвращают "Unsupported"

**Симптом:** Элементы данных с ключами `docker.*` показывают состояние "не поддерживается"

**Решения:**
- Проверьте, что модуль загружен: `zabbix_agentd -p | grep docker.modver`
- Проверьте права Docker (см. [Дополнительные права Docker](#дополнительные-права-docker))
- Проверьте доступ к сокету Docker: `sudo -u zabbix docker ps` (должно работать)
- Проверьте, что Docker запущен: `systemctl status docker`
- Проверьте монтирование cgroup: `mount | grep cgroup`

### Отсутствие Cgroup или прав Docker

**Симптом:** Некоторые метрики возвращают пустые значения или нули

**Решения:**
- Включите memory cgroup: Добавьте `cgroup_enable=memory` в параметры загрузки ядра (в `/etc/default/grub`), затем `update-grub` и перезагрузка
- Проверьте файловую систему cgroup: `ls -la /sys/fs/cgroup/`
- Добавьте пользователя zabbix в группу docker: `usermod -aG docker zabbix`
- Для систем с SELinux установите модуль политики SELinux (см. выше)

### Проблемы с производительностью docker.stats

**Симптом:** Тайм-аут агента или высокая загрузка CPU при использовании ключей `docker.stats`

**Объяснение:** `docker.stats` стримит данные в реальном времени из Docker API (0.3-0.7 с на вызов), что делает его медленнее метрик на основе cgroup.

**Решения:**
- По возможности используйте ключи на основе cgroup (`docker.cpu`, `docker.mem`, `docker.dev`) — они в ~10 раз быстрее
- Увеличьте настройку `Timeout` Zabbix agent в `zabbix_agentd.conf`: `Timeout=10`
- Уменьшите частоту опроса для элементов данных `docker.stats`
- Рассмотрите использование пассивных проверок с более длинными интервалами

### Отладка

Включите логирование отладки в `/etc/zabbix/zabbix_agentd.conf`:
```ini
DebugLevel=4
```

Перезапустите Zabbix agent и проверьте логи:
```bash
systemctl restart zabbix-agent
tail -f /var/log/zabbix/zabbix_agentd.log
```

Отладочные сообщения модуля будут включать префикс `[Docker]`.

## Связь с официальным шаблоном "Docker by Zabbix agent 2"

Современные релизы Zabbix (6.0+) поставляются с официальным шаблоном **"Docker by Zabbix agent 2"**, который обеспечивает мониторинг Docker используя:
- **Zabbix agent 2** (агент на Go) со встроенным плагином Docker
- Не требуются внешние скрипты или модули
- Прямая интеграция с Docker API

**Страница официальной интеграции:** https://www.zabbix.com/integrations/docker

### Как этот модуль сравнивается с официальным

Этот репозиторий предоставляет **альтернативный подход к мониторингу Docker** через внешний C-модуль для классического Zabbix agent (C agent). Рассмотрите использование этого модуля, когда:

1. **Вам нужны дополнительные или пользовательские метрики**, недоступные в официальном шаблоне
2. **Вы уже используете этот модуль** в существующих production-окружениях и хотите продолжить его использование
3. **Вы мигрируете с устаревших установок** и нужна совместимость с инфраструктурой старого Zabbix agent
4. **Вы предпочитаете метрики на основе cgroup**, которые обеспечивают сбор в ~10 раз быстрее по сравнению с методами на основе API
5. **Вы работаете со специализированными контейнерными платформами**, такими как Mesos/Marathon/Chronos, со специфическими шаблонами

Для новых развертываний Zabbix (особенно с Zabbix 7.0+) сначала оцените, подходит ли вам официальный шаблон "Docker by Zabbix agent 2". Оба подхода являются корректными и могут сосуществовать в разных частях вашей инфраструктуры.

## Рекомендации для продакшена

При развертывании этого модуля в production-окружениях следуйте этим лучшим практикам:

### Частота обнаружения и опроса элементов данных
- Установите соответствующие интервалы обнаружения (по умолчанию: 60 секунд) в зависимости от частоты изменения контейнеров
- Для окружений с высокой изменчивостью (частое создание/удаление контейнеров) увеличьте интервал для снижения нагрузки
- Используйте макросы для настройки интервалов опроса для конкретных узлов или шаблонов

### Оптимизация производительности
- Предпочитайте **метрики на основе cgroup** (`docker.cpu`, `docker.mem`, `docker.dev`) вместо `docker.stats` для лучшей производительности
- Вызовы `docker.stats` занимают 0.3-0.7с каждый; используйте их экономно или увеличивайте интервалы элементов данных
- Мониторьте использование CPU и памяти агентом Zabbix; корректируйте интервалы элементов, если агент потребляет избыточные ресурсы

### Безопасность
- **Ограничьте доступ к сокету Docker**: Предоставляйте пользователю `zabbix` доступ только к группе docker (по возможности избегайте запуска агента от root)
- **SELinux/AppArmor**: Используйте предоставленный модуль политики SELinux или создайте соответствующие профили AppArmor
- **Изоляция сети**: Держите Zabbix-сервер и Docker-хосты в отдельных сетях с правилами firewall
- **Аудит меток контейнеров**: При необходимости фильтруйте конфиденциальные контейнеры из обнаружения

### Архитектура
- **Раздельный мониторинг**: Запускайте Zabbix-сервер и базу данных на выделенной инфраструктуре, а не на мониторируемых Docker-хостах
- **Размещение агента**: Развертывайте Zabbix agent на хосте Docker (не внутри контейнеров) для точного доступа к cgroup
- **Ограничения ресурсов**: Установите лимиты памяти/CPU для контейнеризованных агентов Zabbix (Dockbix agent XXL) для предотвращения исчерпания ресурсов

### Область мониторинга
- **Назначение шаблонов**: Привязывайте шаблоны Docker только к узлам, на которых запущен Docker; избегайте привязки к узлам без Docker
- **Селективный мониторинг**: Используйте фильтры обнаружения для мониторинга только критичных контейнеров (избегайте мониторинга всех системных контейнеров)
- **Мониторинг логов**: Внедряйте мониторинг логов выборочно; JSON-логи Docker могут сильно расти

### Высокая доступность
- Развертывайте несколько Zabbix-прокси в распределенных окружениях для отказоустойчивости
- Используйте активные проверки для агентов за NAT или в динамических окружениях
- Мониторьте доступность агентов Zabbix отдельными элементами данных "ping"

### Обслуживание
- **Обновления модуля**: Тестируйте обновления модуля в staging перед развертыванием в production
- **Обновления Zabbix**: При обновлении Zabbix перекомпилируйте модуль для новой версии Zabbix
- **Обновления шаблонов**: Проверяйте изменения шаблонов перед импортом; делайте резервные копии существующих шаблонов

## Часто задаваемые вопросы (FAQ)

### Как проверить, что модуль загружен?

Выполните эту команду на хосте, где запущен Zabbix agent:

```bash
zabbix_agentd -p | grep docker.modver
```

Если модуль загружен, вы увидите вывод вроде:
```
docker.modver                                 [s|0.5]
```

Альтернативно, проверьте лог Zabbix agent (`/var/log/zabbix/zabbix_agentd.log`) на наличие сообщений о загрузке модуля.

### Почему не обнаруживаются контейнеры?

Распространенные причины и решения:

1. **Права Docker**: Пользователь `zabbix` должен иметь доступ к сокету Docker
   ```bash
   # Добавьте zabbix в группу docker
   sudo usermod -aG docker zabbix
   sudo systemctl restart zabbix-agent

   # Проверьте доступ
   sudo -u zabbix docker ps
   ```

2. **Модуль не загружен**: Проверьте, что модуль загружен (см. вопрос выше)

3. **Нет запущенных контейнеров**: Обнаружение находит только запущенные контейнеры; запустите хотя бы один контейнер

4. **Правило обнаружения отключено**: Проверьте, что правило обнаружения "Docker containers" включено в привязанном шаблоне

5. **Интервал обнаружения**: Подождите следующий цикл обнаружения (по умолчанию: 60 секунд)

### Можно ли использовать это вместе с официальным шаблоном "Docker by Zabbix agent 2"?

Да, но не на одном хосте с одним агентом. У вас есть два варианта:

1. **Разные хосты**: Используйте этот модуль (с Zabbix agent C) на одних Docker-хостах и официальный шаблон (с Zabbix agent 2) на других хостах
2. **Разные метрики**: Если вы запускаете оба агента на одном хосте (возможно, но необычно), настройте разные порты и используйте разные шаблоны

В большинстве случаев выбирайте один подход на хост, чтобы избежать дублирования метрик и путаницы.

### Как мониторить контейнеры в Kubernetes?

Этот модуль может мониторить Docker-контейнеры на узлах Kubernetes:

1. **Развертывание на уровне узла**: Запускайте Zabbix agent (с этим модулем) непосредственно на каждом рабочем узле Kubernetes (не как pod)
2. **Подход DaemonSet** (продвинутый): Разверните Zabbix agent как Kubernetes DaemonSet с:
   - `hostNetwork: true`
   - `privileged: true`
   - Монтированием тома для `/var/run/docker.sock` (если используется Docker runtime)
   - Монтированием тома для `/sys/fs/cgroup`

**Примечание**: Если ваш кластер Kubernetes использует containerd или CRI-O (вместо Docker), этот модуль не будет работать напрямую. Рассмотрите использование инструментов мониторинга, нативных для Kubernetes, или шаблонов Kubernetes от Zabbix.

### Какие метрики собираются быстрее всего?

Рейтинг производительности (от быстрых к медленным):

1. **`docker.up[cid]`** - Самая быстрая (простая проверка cgroup)
2. **`docker.cpu[cid,metric]`** - Быстро (чтение cgroup)
3. **`docker.mem[cid,metric]`** - Быстро (чтение cgroup)
4. **`docker.dev[cid,file,metric]`** - Быстро (чтение cgroup)
5. **`docker.discovery`** - Средне (вызов Docker API)
6. **`docker.inspect[cid,...]`** - Средне (вызов Docker API)
7. **`docker.stats[cid,...]`** - Медленно (0.3-0.7с на контейнер, потоковая передача в реальном времени)

Для лучшей производительности отдавайте предпочтение метрикам на основе cgroup перед `docker.stats`.

### Как обновиться на новую версию Zabbix?

При обновлении Zabbix:

1. **Обновите Zabbix-сервер и базу данных** первыми (следуйте официальному руководству по обновлению Zabbix)
2. **Обновите Zabbix agent** на Docker-хостах
3. **Перекомпилируйте модуль** для новой версии Zabbix:
   - Скачайте новый исходный код Zabbix (соответствующий вашей новой версии)
   - Скомпилируйте модуль, следуя разделу [Компиляция](#компиляция)
4. **Замените старый модуль** новым файлом `.so`
5. **Перезапустите Zabbix agent**:
   ```bash
   sudo systemctl restart zabbix-agent
   ```
6. **Проверьте**, что модуль загружается корректно

**Предсобранные бинарные файлы** могут быть недоступны для последних версий Zabbix сразу; будьте готовы к компиляции из исходного кода.

### Что делать при ошибках "Unsupported item key"?

Причины и решения:

1. **Модуль не загружен**: Проверьте с помощью `zabbix_agentd -p | grep docker`
2. **Неправильная версия Zabbix**: Убедитесь, что модуль скомпилирован для вашей точной версии Zabbix
3. **Опечатка в ключе элемента данных**: Проверьте синтаксис ключа (например, `docker.cpu[cid,user]`, а не `docker.cpu[cid user]`)
4. **Формат ID контейнера**: Для коротких ID и имен используйте префикс `/` (например, `/zabbix-server`)

### Работает ли это с Podman?

Частично. Podman предоставляет совместимый с Docker API сокета, поэтому некоторые функции могут работать:

1. **Включите сокет Podman**:
   ```bash
   systemctl enable --now podman.socket
   ```
2. **Создайте символическую ссылку на сокет Docker**:
   ```bash
   ln -s /run/podman/podman.sock /var/run/docker.sock
   ```

Однако **пути cgroup отличаются** между Docker и Podman, поэтому метрики на основе cgroup могут работать некорректно. Этот модуль в первую очередь предназначен для Docker; поддержка Podman является экспериментальной.

## Компиляция

Если предсобранные бинарные файлы не работают в вашей системе, скомпилируйте модуль из исходного кода.

### Предварительные требования

Установите необходимые пакеты:

**CentOS/RHEL:**
```bash
yum install -y wget autoconf automake gcc git pcre-devel jansson-devel
```

**Debian/Ubuntu:**
```bash
apt-get install -y wget autoconf automake gcc git make pkg-config libpcre3-dev libjansson-dev
```

**Fedora:**
```bash
dnf install -y wget autoconf automake gcc git make pcre-devel jansson-devel
```

**openSUSE:**
```bash
zypper install -y wget autoconf automake gcc git make pkg-config pcre-devel libjansson-devel
```

### Шаги сборки

```bash
# Клонируйте исходный код Zabbix (укажите вашу версию Zabbix)
git clone -b 6.0.0 --depth 1 https://github.com/zabbix/zabbix.git /usr/src/zabbix
cd /usr/src/zabbix

# Настройте Zabbix
./bootstrap.sh
./configure --enable-agent

# Подготовьте каталог модуля
mkdir -p src/modules/zabbix_module_docker
cd src/modules/zabbix_module_docker

# Скачайте исходный код модуля
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/zabbix_module_docker.c
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/Makefile

# Скомпилируйте
make
```

Результат: `zabbix_module_docker.so` (динамически связанная библиотека shared object)

### Компиляция на основе Docker

Альтернативно, используйте Docker для компиляции. Примеры Dockerfile для различных дистрибутивов доступны в каталоге [`dockerfiles/`](https://github.com/monitoringartist/zabbix-docker-monitoring/tree/master/dockerfiles).

Пример:
```bash
cd dockerfiles/centos
docker build -t zabbix-docker-module-builder .
docker run --rm -v $(pwd):/output zabbix-docker-module-builder
```

## Разработка

### Структура репозитория

```
zabbix-docker-monitoring/
├── src/modules/zabbix_module_docker/  # Исходный код C-модуля
│   ├── zabbix_module_docker.c         # Основная реализация модуля
│   └── Makefile                        # Конфигурация сборки
├── template/                           # Шаблоны Zabbix (XML)
│   ├── Zabbix-Template-App-Docker.xml
│   ├── Zabbix-Template-App-Docker-active.xml
│   └── Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml
├── dockerfiles/                        # Dockerfile'ы для сборки бинарных файлов
│   ├── amazonlinux/
│   ├── centos/
│   ├── debian/
│   ├── fedora/
│   ├── opensuse/
│   └── ubuntu/
├── doc/                                # Документация и изображения
├── selinux/                            # Модуль политики SELinux
└── LICENSE                             # Лицензия GPL-2.0
```

### Сборка модуля

См. [Компиляция](#компиляция) для подробных инструкций по сборке.

### Зависимости

- **Заголовочные файлы Zabbix agent** (из исходного кода Zabbix)
- **Библиотека PCRE** (libpcre) для регулярных выражений
- **Библиотека Jansson** (libjansson) для парсинга JSON
- **Компилятор C** (gcc, clang)
- **Make**

### Вклад в проект

Приветствуются вклады! Пожалуйста:
1. Сделайте форк этого репозитория
2. Создайте ветку для новой функции
3. Тщательно протестируйте свои изменения
4. Отправьте pull request с чётким описанием

Для сообщения об ошибках и запросов функций используйте [трекер GitHub issues](https://github.com/ranas-mukminov/zabbix-docker-monitoring/issues).

## Поддержка и услуги от run-as-daemon.ru

Этот форк поддерживается **Ранасом Мукминовым** (run-as-daemon.ru) и активно используется в промышленных средах Zabbix для мониторинга Docker и Kubernetes-инфраструктуры.

Если вам нужна помощь с мониторингом Zabbix/Docker/Grafana в production, проектированием высокодоступных систем, стратегией резервного копирования или реагированием на инциденты, вы можете обратиться через: **[run-as-daemon.ru](https://run-as-daemon.ru)**

Предоставляемые услуги:
- Консалтинг по развертыванию и настройке Zabbix
- Решения для мониторинга Docker/Kubernetes
- Разработка дашбордов Grafana
- Оптимизация производительности и устранение неполадок
- Разработка пользовательских модулей и интеграция

## Лицензия

Этот проект распространяется под лицензией **GNU General Public License v2.0 (GPL-2.0)**.

Оригинальный проект: [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring) от Monitoring Artist

Форк поддерживается: Ранас Мукминов ([run-as-daemon.ru](https://run-as-daemon.ru))

См. файл [LICENSE](LICENSE) для полного текста лицензии.

---

## Полезные ресурсы

- [Официальная документация Zabbix](https://www.zabbix.com/documentation)
- [Справочник Docker API](https://docs.docker.com/engine/api/)
- [Документация Cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/)
- [Dockbix Agent XXL](https://github.com/monitoringartist/dockbix-agent-xxl)
- [Плагин Grafana Zabbix](https://github.com/alexanderzobnin/grafana-zabbix)
- [Встроенный мониторинг Docker в Zabbix (Agent2)](https://www.zabbix.com/integrations/docker)

---

**Примечание:** Этот модуль дополняет Zabbix agent (C agent). Для Zabbix agent2 (агент на Go) рассмотрите использование [встроенного плагина Docker](https://www.zabbix.com/integrations/docker).
