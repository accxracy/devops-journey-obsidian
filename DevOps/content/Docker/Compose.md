---
tags:
 - docker
 - compose
 - devops
 - iac
aliases:
 - docker-compose.yml
 - Оркестрация
---

# 🐙 Docker Compose: Локальная оркестрация

Запускать 5 контейнеров (Бот, Redis, PostgreSQL, Nginx, Prometheus) ручными командами `docker run` с кучей флагов — это путь эникея. Девопсы используют **Docker Compose**.

Compose — это инструмент для описания многоконтейнерных приложений в формате YAML. Это базовая реализация **Инфраструктуры как Код (IaC)**.

## 📄 Анатомия docker-compose.yml

 services:
 bot:
  build: .          # Собрать образ из текущей папки
  restart: always      # Авто-рестарт при падении (OOM Killer)
  depends_on:
  - db           # Запустить только после старта БД
  env_file: .env       # Прокинуть токены из файла (12-Factor App)

 db:
  image: postgres:15-alpine # Взять готовый легкий образ
  ports:
  - "127.0.0.1:5432:5432"  # Проброс порта (только на localhost для безопасности!)
  volumes:
  - pg_data:/var/lib/postgresql/data

 volumes:
 pg_data:

## 🛠 Базовые команды
Выполняются в папке, где лежит `docker-compose.yml`:
* `docker compose up -d` : Собрать и запустить все сервисы в фоне (Daemon).
* `docker compose down` : Остановить и удалить контейнеры и сети.
* `docker compose ps` : Посмотреть статус сервисов.
* `docker compose logs -f bot` : Читать логи конкретного сервиса в реальном времени.
* `docker compose exec db sh` : Провалиться в консоль работающей базы данных.