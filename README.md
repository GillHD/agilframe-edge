# AgilFrame Edge

**Русский** · [English](README.en.md)

AgilFrame Edge публикует подготовленный MP4-файл со встраиваемого Linux-устройства в SFU-комнату AgilFrame. Агент на Node.js передаёт видео и звук через FFmpeg и серверный LiveKit Ingress по RTMP или RTMPS без перекодирования. Источник воспроизводится по кругу.

[Проект AgilFrame](https://github.com/GillHD/agilframe) · [Core](https://github.com/GillHD/agilframe-core)

## Подготовка

- Установите Node.js 20 или новее, FFmpeg и FFprobe на устройстве.
- Подготовьте MP4 с одним видеотреком H.264 (`yuv420p`) и одним аудиотреком AAC-LC. Поддерживаются один или два аудиоканала с частотой 44,1 или 48 кГц.
- Включите Redis для Flows в Core, настройте управляемый вход RTMP в LiveKit Ingress и создайте SFU-комнату в организации, которой принадлежит ключ API. [Настройка Core и LiveKit Ingress](https://github.com/GillHD/agilframe-core/blob/main/agil/core/docs/flows.md#managed-rtmp-push)

## Публикация

Из корня репозитория запустите агент от выделенной учётной записи на доверенном устройстве. Для `--core` и `--ingress` указывайте адреса сервисов без пути и ключа. Для `--input` нужен абсолютный путь к файлу. HTTP и RTMP разрешены только при локальной проверке с числовым loopback-адресом. При RTMPS агент на Node.js проверяет сертификат и имя сервера перед передачей данных. FFmpeg подключается через временный локальный порт.

```bash
read -rs -p 'Tenant API key: ' AGIL_EDGE_CORE_API_KEY
export AGIL_EDGE_CORE_API_KEY
node agent/run-rtmp-publisher.mjs \
  --core https://core.example.org \
  --ingress rtmps://publish.example.org:443 \
  --room studio \
  --input /srv/edge/media/studio.mp4
unset AGIL_EDGE_CORE_API_KEY
```

Агент создаёт отдельный Flow для каждого запуска и записывает его идентификатор и состояние в журнал. Состояние `ready` означает, что Ingress подтвердил приём потока. Если Ingress временно перестаёт подтверждать приём, агент переходит в `degraded`. При ожидании подтверждения дольше 45 секунд агент завершает публикацию и удаляет созданный Flow.

При `SIGINT` или `SIGTERM` агент останавливает FFmpeg и запрашивает удаление своего Flow. Core при удалении Flow отзывает вход. Если ответ Core потерян, найдите Flow по идентификатору из журнала и при необходимости удалите его через панель или API.

Ключ API Core используется только агентом. Локальные процессы с доступом к аргументам FFmpeg могут увидеть ключ публикации.

## Проверка

С установленными FFmpeg и FFprobe выполните `npm test` из корня репозитория. Устанавливать npm-пакеты для этих тестов не требуется.

Чтобы включить тесты передачи медиапотока, укажите путь к MediaMTX на тестовом стенде. В тестах MediaMTX играет роль серверного приёмника. Устройство при штатной публикации отправляет поток в LiveKit Ingress.

```bash
AGIL_QA_MEDIAMTX_BIN=/absolute/path/to/mediamtx npm test
```

Тесты проверяют приём H.264 и AAC, прекращение приёма после остановки и проверку имени в сертификате RTMPS. Отдельный сквозной сценарий Core проверяет кадры и звук в браузере, а также отзыв входа после остановки. [Инструкция по сквозной проверке](https://github.com/GillHD/agilframe-core/tree/main/agil/edge)

[Лицензия](LICENSE) · [Уведомления](NOTICE) · [Как внести вклад](CONTRIBUTING.md) · [Безопасность](SECURITY.md)
