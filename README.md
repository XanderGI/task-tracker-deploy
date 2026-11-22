# Task Tracker Deploy

Docker Compose для Task Tracker. Полный стек поднимает веб-интерфейс, REST API, планировщик отчётов, генерацию резюме и отправку писем. Для PostgreSQL, Kafka и Mailpit используются именованные тома.

## Полный стек

Подходит для локального прогона и развёртывания на сервере. Понадобятся Docker и Docker Compose. Образы приложений пулятся из Docker Hub.

В корне этого репозитория скопируйте `.env.example` в `.env` (`cp .env.example .env`). Задайте пароль PostgreSQL (`DB_PASSWORD`), два разных секрета JWT (`JWT_SECRET_PHRASE`, `INTERNAL_JWT_SECRET_PHRASE`, **каждый не короче 32 байт**) и заполните `GEMINI_MODEL` и `GEMINI_API_KEY`.

В `.env.example` задано `REPORT_CRON=0 * * * * *` для быстрой проверки отчёта раз в минуту. Для постоянной работы уберите `REPORT_CRON` из `.env` или задайте `REPORT_CRON=0 0 0 * * *`. В обоих случаях отчёт будет запускаться ежедневно в полночь по Москве.

```bash
docker compose -f docker-compose.prod.yaml config --quiet
docker compose -f docker-compose.prod.yaml pull
docker compose -f docker-compose.prod.yaml up -d
```

Откройте `http://localhost` на машине с Docker. Почта в этой конфигурации идёт в Mailpit (`mailpit:1025`). Его веб-интерфейс доступен на той же машине по `http://127.0.0.1:8025`.

Для проверки зарегистрируйте тестового пользователя и найдите приветственное письмо в Mailpit. Создайте задачу и дождитесь ближайшего запуска планировщика. Текст письма с отчётом должен совпасть с текстом в диалоге «Последний отчёт».

## Локальная инфраструктура

Чтобы запустить приложения из исходников, сначала поднимите PostgreSQL, Kafka, Kafka UI и Mailpit. Если из этого каталога уже запущен полный стек, остановите его командой `docker compose -f docker-compose.prod.yaml down`. Оба Compose-файла по умолчанию используют один проект.

```bash
docker compose -f docker-compose.dev.yaml up -d
```

| Сервис | Адрес с машины разработчика |
| --- | --- |
| PostgreSQL | `jdbc:postgresql://localhost:5430/postgres_db` |
| Kafka | `localhost:9094` |
| Kafka UI | `http://localhost:8088` |
| Mailpit | SMTP `localhost:1025`, веб-интерфейс `http://localhost:8025` |

Когда PostgreSQL и Kafka перейдут в состояние `healthy`, можно запускать приложения. Для Java-сервисов нужен JDK 17. Запускайте каждый сервис из корня его репозитория командой `./gradlew bootRun` (`.\gradlew.bat bootRun` в PowerShell) или через IDE. Файл `.env` в deploy читает Docker Compose. При запуске из исходников задайте нужные сервису переменные в терминале или в конфигурации запуска IDE.

1. Запустите backend. Для работы только с REST API этого достаточно. [Настройки backend](https://github.com/XanderGI/task-tracker-backend#запуск).
2. Для проверки писем запустите email-sender. Для проверки ежедневного отчёта запустите также summarization, затем scheduler. У backend и scheduler должен быть одинаковый `INTERNAL_JWT_SECRET_PHRASE`. Для быстрого прогона задайте `REPORT_CRON=0 * * * * *` в окружении scheduler. [Email Sender](https://github.com/XanderGI/task-tracker-email-sender#запуск), [Summarization](https://github.com/XanderGI/task-tracker-summarization#запуск), [Scheduler](https://github.com/XanderGI/task-tracker-scheduler#запуск).
3. Для работы через браузер запустите локальный Nginx из репозитория frontend. [Локальный запуск frontend](https://github.com/XanderGI/task-tracker-frontend#локальная-разработка).

## Почта через SMTP.BZ

На VPS добавьте в `.env` логин `SMTP_BZ_USERNAME` и пароль `SMTP_BZ_PASSWORD` из вкладки SMTP в кабинете провайдера. `docker-compose.smtp.yaml` переключает `email-sender` на `connect.smtp.bz:465` с TLS и задаёт адрес отправителя `noreply@tracker-task.xyz`.

Проверьте конфигурацию и пересоздайте `email-sender` в уже запущенном стеке:

```bash
docker compose -f docker-compose.prod.yaml -f docker-compose.smtp.yaml config --quiet
docker compose -f docker-compose.prod.yaml -f docker-compose.smtp.yaml up -d
```