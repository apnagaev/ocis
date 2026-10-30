# ownCloud Infinite Scale cluster (Ansible + Docker Compose)

Файловое облако [oCIS](https://owncloud.dev/ocis/) на двух узлах с офисом, полнотекстовым поиском и антивирусом. Разворачивается одним плейбуком.

- oCIS с внешним NATS, общими данными на NFS, своими копиями конфигов и поискового индекса на каждом узле
- Вход через Keycloak (OIDC), пользователи и группы читаются из Active Directory по LDAPS
- Документы открываются в Collabora Online через сервис collaboration (WOPI)
- Полнотекстовый поиск через Apache Tika, проверка загрузок через ClamAV
- Загрузка из внешних источников через Uppy Companion
- Расширения веб-интерфейса: importer, unzip, progress-bars, photo-addon, json-viewer, external-sites, advanced-search
- Reverse proxy с vIP на тех же хостах, TLS терминируется на прокси
- Firewall на nftables: сервисные порты доступны только прокси и мониторингу, gRPC только узлам кластера

```
                     клиенты
                        │ HTTPS
                  ┌─────▼─────┐  vIP
                  │  reverse  │  cloud / wopi / collabora / companion
                  │   proxy   │  (на обоих хостах)
                  └──┬─────┬──┘
   :9200 :9300 :9980 │     │ :9200 :9300 :9980
     :3020           │     │ :3020
       ┌─────────────▼┐   ┌▼─────────────┐
       │ node1        │◄─►│ node2        │  gRPC :9142 :9301 :9233 :9115
       │ ocis, collab │   │ ocis, collab │
       │ collabora    │   │ collabora    │
       │ tika, clamav │   │ tika, clamav │
       └───┬───────┬──┘   └──┬───────┬───┘
           │       └────┬────┘       │
       ┌───▼───┐   ┌────▼────┐   ┌───▼────┐
       │ NFS   │   │  NATS   │   │ AD     │
       │/ocisdata  │ :4222   │   │ LDAPS  │
       └───────┘   └─────────┘   └────────┘
```

## Структура

```
install_ocis.yaml              # плейбук: firewall + конфиги + rolling restart + очистка
inventory.example.yml          # пример inventory (скопировать в inventory.yml)
tasks/healthcheck.yaml         # проверки узла после перезапуска
files/ocis/
  ocis.yaml.j2                 # шаблон docker-compose.yaml
  ocis.nft.j2                  # шаблон правил nftables
  owncloud.env.example         # шаблон секретов для переменных окружения
  owncloud.env                 # НЕ в репозитории: пароли LDAP, SMTP, админки Collabora
  conf/                        # раскладывается в /docker-compose/ocis/conf на каждом узле
    ocis.yaml                  # НЕ в репозитории: ключи кластера и ID (из ocis init)
    proxy.yaml.j2              # квота для роли User
    csp.yaml                   # Content Security Policy
    app-registry.yaml          # какие форматы открывает Collabora
    ldap-ca.crt                # НЕ в репозитории: CA контроллера домена (по желанию)
```

## Требования

- Хосты Debian/Ubuntu с Docker Engine и плагином `docker compose`
- Ansible на управляющей машине (только встроенные модули)
- Каталог `/ocisdata` на NFS, смонтированный на обоих узлах
- NATS (JetStream), доступный обоим узлам
- Keycloak с клиентом `ocis` и маппингом ролей в claim `roles`
- Active Directory: учётная запись для чтения и группы `owncloud_client`, `owncloud_home`
- DNS-имена `cloud`, `wopi`, `collabora`, `companion`, указывающие на прокси

## Настройка

1. Inventory:
   ```bash
   cp inventory.example.yml inventory.yml
   ```
   Для каждого хоста: `ansible_host` и `node` (1 или 2, порядок перезапуска). Группа `docker`.

2. Секреты. Их два вида:
   - **`files/ocis/conf/ocis.yaml`**: ключи кластера (JWT, machine auth, system user, transfer, service account, WOPI) и ID (system user, admin, storage mount). Создаётся один раз командой `ocis init` и дальше только раскладывается плейбуком, одинаковый на всех узлах. На новой инсталляции получите его так:
     ```bash
     docker run --rm -v "$PWD/files/ocis/conf:/etc/ocis" owncloud/ocis:<версия> init --insecure no
     ```
     На существующей скопируйте текущий `ocis.yaml` с узла и проверьте, нет ли в нём ключей, которые новая версия называет иначе:
     ```bash
     docker run --rm -v /docker-compose/ocis/conf:/etc/ocis owncloud/ocis:<версия> init --diff --insecure no
     ```
     Команда кладёт рядом `ocis.config.patch` с разницей и ничего не меняет. Перенесите нужное в `files/ocis/conf/ocis.yaml`, а сам патч удалите: в нём секреты. Пример из практики: при переходе на 7.x+ ключ `collaboration.wopiapp.secret` стал `collaboration.wopi.secret`. Со старым ключом collaboration не стартует («The WOPI secret has not been set properly»), и Collabora пропадает из интерфейса.
      ID в нём не меняйте: к ним привязаны пространства и учётные записи. Ключи менять можно, после смены `jwt_secret` все пользователи один раз войдут заново.
   - **`files/ocis/owncloud.env`**: пароли, которые oCIS получает через переменные окружения.
     ```bash
     cp files/ocis/owncloud.env.example files/ocis/owncloud.env
     ```

   Оба файла лучше зашифровать: `ansible-vault encrypt files/ocis/owncloud.env files/ocis/conf/ocis.yaml`, плейбук тогда запускайте с `--ask-vault-pass`.

3. CA контроллера домена (по желанию) положите в `files/ocis/conf/ldap-ca.crt` и задайте `ldap_ca_cert: ldap-ca.crt`: oCIS будет проверять сертификат LDAPS. Имя в сертификате должно совпадать с хостом из `ldap_uri`. По умолчанию `ldap_ca_cert: ""`, и проверка выключена (`OCIS_LDAP_INSECURE=true`).

4. Переменные плейбука (`install_ocis.yaml`, блок `vars`):

   | Переменная | Назначение |
   |---|---|
   | `ocis_version`, `collabora_version`, ... | версии образов, меняются только здесь |
   | `ocis_url`, `oidc_issuer`, `*_domain` | адреса облака, Keycloak, Collabora, WOPI, Companion |
   | `nats_address` | NATS, общий для кластера |
   | `smtp_*`, `ldap_*` | почта и Active Directory (пароли в `owncloud.env`) |
   | `default_quota_bytes` | квота нового пользователя, по умолчанию 1 ГБ |
   | `proxies`, `monitoring` | кому ещё открыты сервисные порты; узлы кластера и Docker-сети разрешены всегда |
   | `proxy_ports`, `cluster_ports` | какие порты закрывает firewall |

## Развертывание

```bash
ansible-playbook -i inventory.yml install_ocis.yaml
```

Плейбук:
1. Ставит nftables и применяет `/etc/nftables.d/ocis.nft` (с проверкой синтаксиса).
2. Создаёт каталоги с владельцем `1000:1000`, генерирует `docker-compose.yaml`, копирует `owncloud.env` (`0600`) и раскладывает в `/docker-compose/ocis/conf` конфиги: `ocis.yaml` (`0600`), `csp.yaml`, `app-registry.yaml`, `proxy.yaml` и CA.
3. Скачивает ровно те образы, что указаны в compose (`docker compose pull`).
4. Перезапускает **узел 1** и ждёт ответа oCIS (`/status.php`), Collabora (`/hosting/discovery`), WOPI и Companion. Затем то же для **узла 2**.
5. Удаляет неиспользуемые образы. Тома не трогает: в `clamav-db` лежат антивирусные базы.

Все команды compose на сервере выполняйте с `--env-file owncloud.env`, иначе Docker не подставит пароль админки Collabora:
```bash
cd /docker-compose/ocis && docker compose --env-file owncloud.env ps
```

## Сеть и firewall

| Порт | Сервис | Кому открыт |
|---|---|---|
| 9200 | oCIS (HTTP) | `proxies`, `monitoring`, узлы, Docker-сети, localhost |
| 9300 | collaboration, WOPI | то же |
| 9980 | Collabora | то же |
| 3020 | Companion | то же |
| 9142 | gRPC-шлюз REVA | узлы кластера, Docker-сети, localhost |
| 9301 | collaboration, gRPC | то же |
| 9233, 9115 | oCIS | то же |
| 9998 | Tika | не публикуется, доступна только внутри сети `ocis` |

- Docker публикует порты через DNAT раньше цепочки `input`, поэтому правила стоят в `prerouting` с приоритетом `-150`, пока виден настоящий порт и адрес клиента.
- **Не делайте `systemctl reload/restart nftables`** на работающем хосте: стандартный `/etc/nftables.conf` начинается с `flush ruleset` и снесёт правила Docker. Применять правила только так: `nft -f /etc/nftables.d/ocis.nft`.
- **Закройте на прокси админку Collabora:** пути `/browser/dist/admin` и `/cool/adminws` для `collabora.*`.

## Хранилище и отказоустойчивость

- `/ocisdata/data`, `storage`, `apps` — общие для обоих узлов (NFS).
- Конфиги лежат локально в `/docker-compose/ocis/conf` на каждом узле и монтируются в `ocis` и `collaboration` как `/etc/ocis`. Источник правды — репозиторий: правки на сервере перезапишет следующий прогон. Узел не зависит от NFS при старте, а сломанный конфиг не попадает сразу на все узлы.
- `ocis init` при старте контейнера не выполняется: если бы `ocis.yaml` не оказалось на месте, он сгенерировал бы на каждом узле свои ключи.
- `/ocisdata/search/docker<N>` — у каждого узла свой: индекс bleve нельзя делить между процессами.
- NATS один на кластер: если он недоступен, не работают оба узла.

### Фиксированные адреса в Docker-сети

Сервисы oCIS регистрируются в общем NATS со своим адресом внутри Docker-сети, а не с IP хоста. Сеть `ocis` на всех узлах одинаковая (`ocis_subnet`), и контейнеры `ocis` и `collaboration` получают на каждом узле одни и те же адреса: `ocis_ip` (.102) и `collaboration_ip` (.103). Поэтому любой адрес из реестра ведёт в локальный контейнер. Остальные контейнеры получают адреса из `ocis_dynamic_range` и не могут занять закреплённые.

Без закрепления порядок создания контейнеров определяет адреса. Если на узлах он разный, сервисы пытаются ходить на чужой адрес, и в логе появляется `dial tcp 172.28.2.x:9142: connect: connection refused`.

Меняя сеть или адреса, меняйте их сразу на всех узлах. Для настоящего gRPC между узлами нужен `network_mode: host` для `ocis` и `collaboration`.

## Обновление oCIS

Поменяйте `ocis_version` и прогоните плейбук. Перед мажорным обновлением (например, 7 → 8):
- сделайте снимок `/ocisdata`, при старте oCIS может мигрировать хранилище;
- прочитайте release notes новой версии oCIS: там перечислены изменённые и удалённые переменные окружения;
- проверьте, что `collaboration` использует тот же образ: в шаблоне это гарантировано одной переменной.

## Проверка

```bash
docker ps                                           # контейнеры Up, collabora healthy
curl -s http://127.0.0.1:9200/status.php            # "installed":true
curl -s http://127.0.0.1:9980/hosting/discovery | head -c 200
nft list table inet ocis
```

## Диагностика

**Не открываются документы в Collabora.** Проверьте, что `wopi.*` и `collabora.*` отдают валидный сертификат: проверка TLS включена (`COLLABORATION_APP_INSECURE=false`, `ssl.ssl_verification=true`). Если внутри сети прокси отдаёт сертификат внутреннего CA, смонтируйте этот CA в контейнеры или временно верните `true`.

**Никто не может войти после обновления конфига.** Посмотрите логи: `docker logs ocis 2>&1 | grep -i ldap`. Частая причина: сертификат LDAPS не проходит проверку по `ldap-ca.crt`.

**`dial tcp 172.28.2.x:9142: connect: connection refused`.** Адреса `ocis` или `collaboration` на узлах разошлись. Проверьте: `docker inspect -f '{{.Name}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ocis ocis_collab` должно показывать одинаковые адреса на всех узлах.

**`Error initializing search service … tika:9998 … connection refused` в первые секунды.** Tika стартует медленнее oCIS, поиск переподключится сам.

**Страницы грузятся очень долго.** Почти всегда это firewall режет прокси. Проверка: `nft delete table inet ocis` (вернуть: `nft -f /etc/nftables.d/ocis.nft`). Посмотреть, кого режут:
```bash
nft -a list chain inet ocis prerouting
nft insert rule inet ocis prerouting position <HANDLE> tcp dport { 9200, 9300, 9980, 3020 } log prefix \"ocis-drop: \"
journalctl -k -f | grep ocis-drop
```
