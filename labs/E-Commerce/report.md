# K8s Configuration Management

Практика управления конфигурациями и секретами в Kubernetes. Проект разделен на две изолированные среды для демонстрации разных подходов к доставке настроек в поды.

## Реализовано
- **ConfigMaps & Secrets:** Хранение публичных конфигураций и безопасное хранение чувствительных данных (БД, JWT, TLS в Base64).
- **Dev-среда (`ecommerce-dev`):** Передача настроек строго через переменные окружения (Environment Variables).
- **Prod-среда (`ecommerce-prod`):** Проброс конфига (`nginx.conf`) и сертификатов в файловую систему через Volume Mounts для возможности обновления "на лету" без рестарта подов.


## Структура
```text
├── manifests/
│  ├── dev/  # Манифесты для dev (Env Vars инъекция)
│  └── prod/ # Манифесты для prod (Volume Mounts)
├── config-files/
│   ├── jwt-public.key
│   ├── tls.crt
│   └── tls.key
└── readme.md