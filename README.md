# Jenkins с агентом: Docker и Ansible для сборок

Контроллер Jenkins (`jenkins/jenkins:lts`). 
Сборки идут на агенте: в нём Docker CLI (подключается к Docker хоста через сокет). 
Ansible (с sshpass — для первого входа на сервер по паролю), git и другие утилиты, корневые сертификаты.

## 1. Файлы

```
jenkins/
├── docker-compose.yml          
├── .env                         # DOCKER_GID=966 (шаг 3)
├── custom-certs/                # корневые сертификаты
└── jenkins-agent/
    ├── Dockerfile               # из репозитория
    └── jenkins_agent_key.pub    # открытый ключ контроллера (шаг 2)
```

Если Jenkins уже запущен из другой папки со своим compose-файлом — удобнее
перенести в неё `docker-compose.yml` и `jenkins-agent/`, а не наоборот: 
тома `jenkins_home` и `jenkins-cacerts` привязаны к имени проекта compose, то есть к имени папки.
Сменится папка — compose заведёт новые пустые тома, и Jenkins начнёт с нуля.

## 2. Ключ для подключения контроллера к агенту

```bash
ssh-keygen -t ed25519 -N "" -C jenkins-agent -f jenkins_agent_key
```

- Закрытый ключ (`jenkins_agent_key`) — в Jenkins: *Manage Jenkins →
  Credentials → Add*: тип *SSH Username with private key*, ID
  `jenkins-agent-ssh`, Username `jenkins`, ключ — содержимое файла.
- Открытый (`jenkins_agent_key.pub`) положить в `jenkins-agent/` рядом с
  Dockerfile: тот копирует его в `/home/jenkins/.ssh/authorized_keys` агента.
- Закрытый файл после этого с диска убрать.
- Сменить ключ — заменить `jenkins_agent_key.pub` и пересобрать образ (шаг 4).

## 3. Номер группы docker на хосте

```bash
getent group docker
>>> docker:x:966:
```

Поле — в `.env` на сервере это `966`: `DOCKER_GID=966`.

## 4. Агент

Порт наружу агенту не нужен — контроллер ходит к нему внутри сети compose по имени `jenkins-agent`.

Собрать и поднять только агента, контроллер не перезапуская:

```bash
docker compose build jenkins-agent
docker compose up -d jenkins-agent
```
Проверка, что Docker хоста виден из агента:

```bash
docker compose exec -u jenkins jenkins-agent docker version
```

## 5. Узел в Jenkins

*Manage Jenkins → Nodes → New Node*:

| Поле | Значение |
|---|---|
| Name | `docker-agent` |
| Type | Permanent Agent |
| Number of executors | `1` |
| Remote root directory | `/home/jenkins/agent` |
| Labels | `docker` |
| Usage | Only build jobs with label expressions matching this node |
| Launch method | Launch agents via SSH |
| Host | `jenkins-agent` |
| Credentials | `jenkins-agent-ssh` |
| Host Key Verification Strategy | Manually trusted key Verification Strategy (при первом подключении — доверить ключ) |
| Advanced → JavaPath | `/opt/java/openjdk/bin/java` |

На встроенном узле контроллера (*Built-In Node*) поставить **Number of
executors = 0** — тогда сборки на контроллере не пойдут вовсе.
