---
tags:
 - kubernetes
 - devops
 - security
 - config
aliases:
 - ConfigMap
 - K8s Secrets
 - Управление конфигурацией
---

# ☸️ Kubernetes:  ConfigMap & Secret

Докер-образы должны быть "глупыми" и универсальными. Они не должны содержать внутри себя пароли, токены или адреса баз данных. Всю конфигурацию Кубернетес "впрыскивает" в контейнер снаружи в момент запуска.

Для этого используются два ресурса:

## 📜 1. ConfigMap (Для открытых данных)
Хранит несекретную конфигурацию в виде словаря (Key-Value). 
* **Что храним:** URL-адреса API, порты, уровень логирования (`DEBUG=True`), конфигурационные файлы целиком (например, текстовый файл `nginx.conf`).

**Пример манифеста:**

 apiVersion: v1
 kind: ConfigMap
 metadata:
 name: bot-config
 data:
 LOG_LEVEL: "INFO"
 DB_HOST: "postgres-service"

## 🔐 2. Secret (Для чувствительных данных)
Хранит секретные данные. На первый взгляд выглядит так же, как ConfigMap, но есть нюанс.
* **Что храним:** Пароли от БД, API-токены (Telegram Token), SSH-ключи, TLS-сертификаты.
* ⚠️ **Главный подвох (ИБ):** По умолчанию Секреты в K8s **НЕ зашифрованы**! Они просто закодированы в формат `base64`. Любой человек, имеющий доступ к кластеру, может раскодировать их командой `echo "MTIz..." | base64 -d`. 
 *(На реальном проде DevOps-инженеры используют сторонние утилиты, например, HashiCorp Vault, чтобы реально шифровать данные).*

**Пример манифеста:**

 apiVersion: v1
 kind: Secret
 metadata:
 name: bot-secret
 type: Opaque
 data:
 # Токен обязательно должен быть закодирован в base64!
 BOT_TOKEN: "MTIzNDU2OkhHZmRzYXF3ZXJ0eQ==" 

---

## 💉 Как передать это в Pod? (Два способа)

Когда ты описываешь Deployment, ты указываешь, как именно Под должен прочитать эти конфиги. Есть два пути:

### Способ А: Как Переменные Окружения (ENV)
Кубер берет ключи из ConfigMap/Secret и делает их системными переменными (которые твой Python-код прочитает через `os.getenv('BOT_TOKEN')`).

 containers:
 - name: my-bot
 image: grishanya/bot:v1.0
 envFrom:
 - configMapRef:
  name: bot-config  # Подгружаем все переменные из ConfigMap
 - secretRef:
  name: bot-secret  # Подгружаем все переменные из Secret

### Способ Б: Как файлы (через Volumes)
Используется, когда нужно прокинуть целый файл (например, конфиг Nginx или SSL-сертификат). Кубер создает виртуальный жесткий диск, кладет туда данные из ConfigMap/Secret и монтирует как папку внутрь Пода.

 volumes:
 - name: nginx-config-volume
 configMap:
  name: nginx-config # Имя ConfigMap, где лежит текст конфига
 containers:
 - name: nginx
 image: nginx
 volumeMounts:
 - name: nginx-config-volume
  mountPath: /etc/nginx/conf.d/ # Куда положить файл внутри контейнера