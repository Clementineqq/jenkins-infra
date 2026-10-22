# jenkins-infra

Docker-обёртка для Jenkins controller: образ на базе `jenkins/jenkins:lts`
плюс `docker-cli`, и `docker-compose.yml`, который поднимает Jenkins.

Нужен, чтобы контроллер разворачивался на чистой машине предсказуемо, а не
собирался руками по инструкции из интернета. Сам Jenkins здесь — только
контроллер: сборки идут на отдельном агенте.

## Состав

```
Dockerfile           jenkins/jenkins:lts + docker-cli
docker-compose.yml   controller: порты 8081/50000, внешний том jenkins_home
```

## Требования

- Docker Engine и Docker Compose v2
- отдельная VM или хост для агента сборки

## Запуск

```bash
# 1. том объявлен external — создаём заранее
docker volume create jenkins_home

# 2. собрать образ и поднять контроллер
docker compose up -d --build

# 3. дождаться старта и забрать initial admin password
docker compose logs -f jenkins
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# 4. открыть веб-интерфейс
#    http://localhost:8081
```

После входа: **Manage Jenkins → Plugins** — поставить минимум

| Плагин | Зачем |
|---|---|
| Docker Pipeline | шаг `withDockerContainer` в общей библиотеке |
| Git / GitHub | работа с репозиториями и вебхуками |
| Credentials Binding | `withCredentials` для секретов |
| Slack Notification | уведомления о сборках |

Затем **Manage Jenkins → System → Jenkins URL** — указать адрес, по которому
GitHub будет присылать вебхуки.

## Порты

| Порт | Назначение |
|---|---|
| `8081` → `8080` | веб-интерфейс. На хосте специально не 8080: этот порт занят приложением |
| `50000` | JNLP-подключение агентов |



**Docker-сокет контроллеру не пробрасывается.** Раньше сборки шли прямо на
контроллере через смонтированный `/var/run/docker.sock` — это фактически root
на хосте для любого, кто может запустить пайплайн. Теперь контроллер только
раздаёт задачи, а сборка идёт на отдельном агенте с меткой `docker`
(`docker-build` для деплой-задач). В образе остался `docker-cli` — он нужен
агенту, а не контроллеру, и позволяет отлаживать пайплайны вручную.



**Образ собирается локально, а не тянется из registry.** Пока контроллер один,
проще: `docker compose up -d --build` всегда даёт актуальную версию. При
нескольких инстансах логичнее собирать и публиковать образ в GHCR.

## Агент сборки

Агент `docker-build` запущен на той же виртуалке, что хостит стенды для простоты (планирую разделить), связан с контроллером по SSH и имеет метки `docker`
(сборка) и `docker-build` (деплой). На машине установлены Docker и SSH-ключ
для подключения к стендам, поэтому пайплайн может и собрать образ, и раскатить
его.

Настройка выполняется вручную (Manage Jenkins → Nodes → New Node), поэтому в
репозитории её нет. Автоматическое создание через Ansible в планах.

## Webhook из GitHub в локальный Jenkins

Jenkins работает в VirtualBox без публичного IP, поэтому GitHub физически не
может доставить webhook на `http://localhost:8081`. Для этого поднимал
временный туннель через сервис CloudPub (аналог ngrok):


Дальше в GitHub: **Settings → Webhooks → Add webhook**

- Payload URL: `https://<поддомен>.cloudpub.ru/github-webhook/`
- Content type: `application/json`
- Secret: значение из Jenkins Credentials (проверка подписи payload)
- Events: *Just the push event*


### Ограничения этого решения

- адрес туннеля меняется при перезапуске, после чего вебхук приходится
  обновлять; стабильный URL даёт именованный туннель (cloudflared) или
  собственный внешний адрес;
- пока туннель поднят, Jenkins доступен из интернета;


Это лабораторное решение. В production Jenkins ставится за reverse proxy с
TLS, вебхук приходит на постоянный адрес и подписан секретом.

## Безопасность

- анонимный доступ выключен, права — `Logged-in users can do anything`;
- регистрация пользователей запрещена;
- секреты хранятся в Jenkins Credentials, а не в пайплайнах и не в репозитории;
- контроллер не имеет доступа к Docker-сокету хоста;
- туннель поднимается только на время работы и закрывается после сессии.


## Полезные команды

```bash
docker compose ps                  # состояние контроллера
docker compose logs -fjenkins      # логи
docker compose restart             # перезапуск
docker exec jenkins jenkins --version
docker volume inspect jenkins_home # где лежат данные
```

## Ограничения

- [ ] агент `docker-build` настраивается вручную, автоматического создания
      (Ansible/cloud-init) нет
- [ ] нет TLS и reverse proxy: Jenkins доступен по HTTP в локальной сети,
      наружу — только через временный туннель
- [ ] джобы и креденшелы создаются руками
- [ ] бэкап `jenkins_home` выполняется вручную, по расписанию не настроен
- [ ] контроллер один, резервирования нет