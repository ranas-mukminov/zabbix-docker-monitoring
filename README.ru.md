# Zabbix Docker Monitoring (форк от run-as-daemon.ru)

Модуль Zabbix, шаблоны и Docker-образ для мониторинга Docker, Kubernetes, Mesos, Swarm и других контейнерных платформ.

[Русский] | [English](README.md)

---

## Обзор

Этот проект предоставляет высокопроизводительный **модуль Zabbix для Docker** (загружаемый модуль на языке C для Zabbix agent) в комплекте с готовыми шаблонами Zabbix и интеграцией с Docker для комплексного мониторинга контейнеров. Он позволяет собирать метрики Docker-контейнеров — CPU, память, блоковый ввод-вывод, сеть и статусную информацию — напрямую в Zabbix без накладных расходов на внешние скрипты.

**Это форк проекта [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring)**, поддерживаемый и адаптированный **Ранасом Мукминовым** для использования в промышленных средах Zabbix.

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

## Совместимость

### Поддерживаемые версии Zabbix

Предсобранные бинарные файлы доступны для версий Zabbix agent: **4.0, 5.0, 5.4, 6.0**

### Поддерживаемые дистрибутивы Linux

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

## О форке и поддержке

Этот форк поддерживается **Ранасом Мукминовым** и активно используется в промышленных средах Zabbix для мониторинга Docker и Kubernetes-инфраструктуры.

**Ранас Мукминов** специализируется на корпоративных решениях мониторинга, консалтинге Zabbix и автоматизации DevOps. Этот форк включает дополнительные улучшения, обновления и адаптации на основе реального опыта эксплуатации в production.

Для профессионального консалтинга по мониторингу, внедрения Zabbix или индивидуальной разработки: [run-as-daemon.ru](https://run-as-daemon.ru)

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
