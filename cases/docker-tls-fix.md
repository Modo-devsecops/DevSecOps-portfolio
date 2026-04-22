# Кейс: Ошибка TLS handshake timeout в Docker

## 📌 Проблема

При попытке выполнить `docker run hello-world` возникала ошибка:
docker: Error response from daemon: failed to resolve reference "docker.io/library/hello-world:latest": failed to authorize: failed to fetch oauth token: Post "https://auth.docker.io/token": net/http: TLS handshake timeout


**Симптомы:**
- Docker CLI не мог скачать образ
- Иконка Docker Desktop в трее была "бледной" (не активно)
- При этом `auth.docker.io/token` открывался в браузере
- Docker daemon работал локально (`docker ps` работал)

## 🔍 Диагностика

### Шаг 1. Проверка сети
- Интернет работал
- Другие сайты открывались

### Шаг 2. Проверка WSL
- Docker Desktop работал на WSL2 бэкенде
- `wsl --list --verbose` показывал работающий `docker-desktop`

### Шаг 3. Проверка DNS
- `/etc/resolv.conf` внутри WSL показывал DNS провайдера (10.255.255.254)
- Это не было критично, но указывало на возможные проблемы

### Шаг 4. Проверка авторизации
- Ошибка происходила именно на этапе `failed to fetch oauth token`
- Сервер авторизации Docker Hub (`auth.docker.io`) был недоступен из Docker, хотя браузер его открывал

## 🧠 Анализ

Проблема была **не в Docker**, а в **сетевой маршрутизации**:
- Провайдер (Ростелеком) блокировал или искажал трафик к `auth.docker.io`
- VPN на уровне ПК работал нестабильно — соединение "проседало"
- Docker Desktop терял авторизацию, но сам движок продолжал работать

## ✅ Решение

### Основное решение: VPN на уровне роутера
1. Настроил VPN-клиент на роутере Cudy WR3000
2. Весь трафик ПК теперь идёт через стабильный туннель
3. Docker Desktop стабильно авторизуется

### Дополнительные шаги (для WSL2)
- Настроил MTU для WSL-интерфейса: `netsh interface ipv4 set subinterface "vEthernet (WSL)" mtu=1400`
- Настроил DNS в WSL на `8.8.8.8`

### Альтернативное решение (для проверки)
- Использовал Podman вместо Docker (работает без демона, обходит некоторые сетевые проблемы)

## 📊 Результат

- `docker run hello-world` успешно выполняется
- Иконка Docker Desktop зелёная
- Авторизация проходит стабильно

## 🎓 Выводы

1. **Ошибка TLS handshake timeout** — это не всегда проблема Docker. Часто это проблема сети, маршрутизации или блокировки.
2. **VPN на уровне ПК нестабилен** — лучше использовать роутерный VPN.
3. **Диагностика должна быть системной**: проверять DNS, MTU, маршруты, а не только логи Docker.
4. **Важно различать**:
   - Docker daemon (работает локально) → отвечает за запуск контейнеров
   - Docker авторизация (требует интернет) → отвечает за иконку в трее и скачивание образов

## 📁 Связанные материалы

- [Решение проблемы MTU в WSL2](https://github.com/microsoft/WSL/issues/4697)
- [GitHub: SSH через порт 443](https://docs.github.com/ru/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)

