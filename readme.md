# ГОСТ TLS Gateway

Локальный ГОСТ TLS-сервер на Nginx в Docker. Он принимает HTTPS на порту `8483`, возвращает тестовую страницу на `/` и проксирует `/api/` на `http://host.docker.internal:8080`.

Ниже — порядок подготовки с нуля. Для ежедневного запуска уже подготовленного стенда достаточно раздела 7.

## 1. Подготовить окружение и образы

Нужны Docker с Linux-контейнерами и Docker Compose. В Windows можно использовать CMD или PowerShell. Все команды Docker выполняются из корня проекта.

Проверить инструменты и собрать образы в указанном порядке.

```cmd
docker version
```

```cmd
docker compose version
```

```cmd
docker build -t tls-gost-tools ./gost-tools
```

```cmd
docker compose build
```

```cmd
docker compose config --quiet
```

`tls-gost-tools` содержит OpenSSL и ГОСТ engine. Dockerfile Nginx использует его как базовый образ, поэтому он собирается первым. `compose config --quiet` проверяет описание Compose, но не сертификаты и не конфигурацию самого Nginx.

## 2. Открыть контейнер для работы с сертификатами

Команда зависит только от оболочки на компьютере.

**Windows CMD**

```cmd
docker run --rm -it --mount "type=bind,source=%cd%\gost-certs,target=/work" --entrypoint sh tls-gost-tools
```

**PowerShell, Bash в Linux или macOS**

```text
docker run --rm -it --mount "type=bind,source=${PWD}/gost-certs,target=/work" --entrypoint sh tls-gost-tools
```

Далее команды разделов 3–6 вводятся **в открывшейся Linux-оболочке контейнера**. Рабочий каталог — `/work`, соответствующий папке `gost-certs` на компьютере. Созданные файлы останутся после удаления контейнера. OpenSSL уже настроен на ГОСТ engine.

```sh
openssl engine -t -c gost
```

В выводе должно быть `[ available ]`. При ошибке остановиться и устранить её перед следующим шагом.

## 3. Выпустить ключ и корневой сертификат УЦ

Эти команды предназначены для нового УЦ. Они записывают файлы с указанными именами. Если УЦ уже существует, сохраните его ключ, сертификат и базу выдачи отдельно перед созданием нового. Обычный запуск Nginx не требует повторного выпуска УЦ.

Создать закрытый ключ ГОСТ Р 34.10-2012, 256 бит, TC26 paramSetA.

```sh
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:1.2.643.7.1.2.1.1.1 -out gost-ca.key
```

Выпустить самоподписанный корневой сертификат на 3650 дней.

```sh
openssl req -new -x509 -key gost-ca.key -md_gost12_256 -days 3650 -subj "/CN=Local GOST Test CA" -addext "basicConstraints=critical,CA:TRUE,pathlen:0" -addext "keyUsage=critical,keyCertSign,cRLSign" -addext "subjectKeyIdentifier=hash" -out gost-ca.crt
```

- `gost-ca.key` — закрытый ключ УЦ, подписывающий сертификаты и CRL.
- `gost-ca.crt` — публичный корневой сертификат для импорта в доверенные МП БР.
- `CA:TRUE` обозначает УЦ, `pathlen:0` задаёт профиль без подчинённых УЦ.
- `keyCertSign,cRLSign` разрешают подпись сертификатов и CRL.
- `-md_gost12_256` выбирает ГОСТ Р 34.11-2012 с хешем 256 бит.

Проверить самоподпись.

```sh
openssl verify -check_ss_sig -CAfile gost-ca.crt gost-ca.crt
```

Ожидается `gost-ca.crt: OK`. Ключи в этой инструкции создаются без пароля для локального стенда. Закрытые `.key` не нужно передавать клиенту или добавлять в Git. В контейнер Nginx корневой ключ не подключается.

## 4. Подготовить учёт сертификатов и настройки УЦ

Чтобы CRL учитывал реальные отзывы, сертификаты будем выпускать через `openssl ca`. Он ведёт базу выданных сертификатов.

Вставить блок целиком в оболочку контейнера. Файл сохранится как `gost-certs/gost-ca.cnf`.

```sh
cat > gost-ca.cnf <<'EOF'
.include /etc/ssl/openssl-gost.cnf

[ca]
default_ca = local_ca

[local_ca]
dir = /work
database = $dir/index.txt
new_certs_dir = $dir/newcerts
certificate = $dir/gost-ca.crt
private_key = $dir/gost-ca.key
serial = $dir/serial
crlnumber = $dir/crlnumber
default_md = md_gost12_256
default_days = 365
default_crl_days = 30
policy = server_policy
unique_subject = no
copy_extensions = none
crl_extensions = crl_ext

[server_policy]
commonName = supplied

[crl_ext]
authorityKeyIdentifier = keyid:always
EOF
```

Создать служебные файлы, сохранив существующие.

```sh
mkdir -p newcerts
```

```sh
test -e index.txt || touch index.txt
```

```sh
test -e serial || printf '1000\n' > serial
```

```sh
test -e crlnumber || printf '1000\n' > crlnumber
```

| Файл или параметр | Назначение |
|---|---|
| `index.txt` | База выданных сертификатов и их статусов, включая отзыв |
| `serial` | Следующий серийный номер сертификата в шестнадцатеричном виде |
| `crlnumber` | Следующий номер CRL |
| `newcerts/` | Архив выданных сертификатов по серийным номерам |
| `default_days = 365` | Срок нового серверного сертификата |
| `default_crl_days = 30` | Срок актуальности нового CRL |
| `copy_extensions = none` | Расширения задаются нашим `.ext`, а не копируются из CSR |

Не обнуляйте базу и счётчики при повторном выпуске сертификатов тем же УЦ. Для нового УЦ создавайте отдельную базу. Эти файлы и архив сертификатов нужно сохранять вместе с материалами соответствующего УЦ.

## 5. Выпустить ключ и сертификат сервера

Перед выпуском проверить `gost-server.ext`. В SAN должны быть реальные имена и IP подключения. Сейчас предусмотрены `localhost`, `127.0.0.1` и `10.0.2.2` для Android Emulator. Объяснение каждого поля находится в [gost-certs/readme.md](gost-certs/readme.md).

Создать отдельный серверный ключ с тем же набором параметров.

```sh
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:1.2.643.7.1.2.1.1.1 -out gost-server.key
```

Создать CSR — запрос на сертификат, подписанный серверным ключом.

```sh
openssl req -new -key gost-server.key -md_gost12_256 -subj "/CN=localhost" -out gost-server.csr
```

Подписать запрос нашим УЦ. Команда добавит сертификат в базу и архив `newcerts/`.

```sh
openssl ca -batch -config gost-ca.cnf -extfile gost-server.ext -in gost-server.csr -out gost-server.crt -notext
```

Посмотреть параметры и проверить назначение, подпись и SAN для Android.

```sh
openssl x509 -in gost-server.crt -noout -text
```

```sh
openssl verify -CAfile gost-ca.crt -purpose sslserver -verify_ip 10.0.2.2 gost-server.crt
```

Ожидается `gost-server.crt: OK`. Nginx использует `gost-server.crt` и `gost-server.key`. CSR после выпуска для запуска не требуется.

## 6. Выпустить и проверить CRL

**CRL выпускается УЦ и содержит отозванные сертификаты этого УЦ.**

```sh
openssl ca -gencrl -config gost-ca.cnf -out gost-ca.crl
```

```sh
openssl crl -in gost-ca.crl -noout -text
```

```sh
openssl crl -in gost-ca.crl -noout -verify -CAfile gost-ca.crt
```

```sh
openssl verify -CAfile gost-ca.crt -CRLfile gost-ca.crl -crl_check_all -purpose sslserver -verify_ip 10.0.2.2 gost-server.crt
```

Ожидаются `verify OK` для подписи CRL и `gost-server.crt: OK` для сертификата. Пустой список отзывов допустим. CRL всё равно содержит подпись, издателя и даты `lastUpdate` / `nextUpdate`.

В МП БР для успешного TLS нужен импорт корня как `TRUSTED_CA` и импорт актуального CRL. Их выполняют через мост отдельно. Серверный сертификат импортировать как доверенный корень не нужно.

Сроки 3650/365/30 дней выбраны для тестового стенда. Они не подтверждают соответствие всем требованиям конкретной сборки МП БР.

Выйти из контейнера перед выполнением команд Compose.

```sh
exit
```

## 7. Проверить и запустить Nginx

Эти команды снова выполняются **на компьютере, из корня проекта**.

Проверить Nginx во временном контейнере и только при успешной проверке запустить сервис.

```cmd
docker compose run --rm --no-deps nginx -t
```

```cmd
docker compose up -d
```

```cmd
docker compose ps
```

Адреса стенда

- С компьютера — `https://127.0.0.1:8483/`.
- Из стандартного Android Emulator — `https://10.0.2.2:8483/`.
- Тест API — `https://10.0.2.2:8483/api/v1/configs`.

API должен быть доступен контейнеру по `host.docker.internal:8080`. В Docker Desktop этот адрес предусмотрен окружением. Для обычного Docker Engine под Linux потребуется отдельно настроить доступ к хосту, например через `extra_hosts: ["host.docker.internal:host-gateway"]`, и доступный контейнеру интерфейс прослушивания API. Привязка API только к `127.0.0.1` хоста в такой схеме может быть недоступна.

Обычный браузер без поддержки ГОСТ TLS может показать `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`. Импорт корня сам по себе не добавляет поддержку ГОСТ-алгоритмов.

### Проверка ГОСТ TLS

Из корня проекта открыть клиент OpenSSL. Для CMD

```cmd
docker run --rm -it --mount "type=bind,source=%cd%\gost-certs,target=/work,readonly" tls-gost-tools s_client -connect host.docker.internal:8483 -servername localhost -CAfile /work/gost-ca.crt -verify_hostname localhost -verify_return_error -tls1_2 -brief
```

Для PowerShell, Linux или macOS

```cmd
docker run --rm -it --mount "type=bind,source=${PWD}/gost-certs,target=/work,readonly" tls-gost-tools s_client -connect host.docker.internal:8483 -servername localhost -CAfile /work/gost-ca.crt -verify_hostname localhost -verify_return_error -tls1_2 -brief
```

Ожидаются `Verification: OK`, TLS 1.2 и ГОСТ-набор шифров. Эта команда проверяет соединение из Docker Desktop, но не импорт CRL и не работу МП БР. Ограничение сети для обычного Docker Engine под Linux здесь такое же, как для доступа к API. Завершить клиент можно через `Ctrl+C`.

## 8. Основные команды Compose

| Действие | Команда |
|---|---|
| Создать или запустить сервис | `docker compose up -d` |
| Остановить с сохранением контейнера | `docker compose stop` |
| Посмотреть состояние | `docker compose ps` |
| Смотреть логи | `docker compose logs -f nginx` |
| Проверить работающий Nginx | `docker compose exec nginx nginx -t` |
| Перечитать конфигурацию после успешной проверки | `docker compose exec nginx nginx -s reload` |
| Пересобрать образ Nginx и применить | `docker compose up -d --build` |
| Пересоздать контейнер с текущими файлами | `docker compose up -d --force-recreate` |
| Остановить и удалить контейнеры и сеть проекта | `docker compose down` |

`down` не удаляет подключённые файлы из `gost-certs`. Команды выпуска сертификатов не являются частью `up` и при запуске автоматически не выполняются.

## 9. Обновление сертификатов и CRL

### Новый серверный ключ и сертификат

Сохранить предыдущие `gost-server.key`, `gost-server.crt` и CSR в отдельный архив. Затем повторить раздел 5 в контейнере инструментов. При том же УЦ сохраняются `gost-ca.key`, `gost-ca.crt`, база, счётчики и `newcerts/`. Новый сертификат получит новый серийный номер.

Проверить результат по разделу 6 и пересоздать контейнер Nginx.

```cmd
docker compose run --rm --no-deps nginx -t
```

```cmd
docker compose up -d --force-recreate
```

Пересоздание обновляет подключения отдельных файлов, в том числе если редактор или OpenSSL заменил файл вместо изменения его содержимого. При неизменном УЦ заново импортировать корень не требуется.

### Обновить CRL

До `nextUpdate` снова открыть контейнер инструментов и выполнить

```sh
openssl ca -gencrl -config gost-ca.cnf -out gost-ca.crl
```

```sh
openssl crl -in gost-ca.crl -noout -verify -CAfile gost-ca.crt
```

```sh
openssl crl -in gost-ca.crl -noout -lastupdate -nextupdate
```

Повторно импортировать обновлённый CRL в МП БР. Перезапуск Nginx для CRL не требуется — текущий сервер его не использует для проверки клиентов, поскольку настроен односторонний TLS.

### Отозвать старый серверный сертификат

Это отдельная операция, а не обязательный шаг запуска. В контейнере указать сохранённый старый сертификат. В примере `old-server.crt` — его копия в `gost-certs`, не текущий сертификат сервера.

```sh
openssl ca -config gost-ca.cnf -revoke old-server.crt -crl_reason superseded
```

```sh
openssl ca -gencrl -config gost-ca.cnf -out gost-ca.crl
```

После проверки подписи импортировать новый CRL в МП БР. Проверка отозванного сертификата с этим CRL должна завершаться ошибкой `certificate revoked`.

### Новый УЦ

Сохранить старые материалы и начать с отдельной базы по разделам 3–6. Новый УЦ требует выпуска нового серверного сертификата и CRL, а также импорта нового корня и CRL в МП БР. Совпадение имени УЦ не делает новый ключ доверенным автоматически.

## 10. Данные для импорта через мост

Импорт выполняется в МП БР на Android через мост `ws://127.0.0.1:8787/bridge`. К мосту должен быть подключён эмулятор, а `initialize` в плагине — успешно выполнен.

Порядок действий — импортировать корневой сертификат, проверить его alias, затем импортировать актуальный CRL того же УЦ.

### Импорт корневого сертификата

Используется содержимое **`gost-certs/gost-ca.crt`** в формате PEM вместе со строками `BEGIN CERTIFICATE` и `END CERTIFICATE`. Серверный сертификат и закрытые ключи для этого запроса не нужны.

Шаблон запроса ниже показывает структуру. `...BASE64...` нужно заменить реальными данными; готовый запрос проще получить командой далее.

```json
{
  "type": "request",
  "id": "import-gost-ca-1",
  "action": "importNotUserCertificate",
  "args": [
    {
      "cert": "-----BEGIN CERTIFICATE-----\n...BASE64...\n-----END CERTIFICATE-----\n",
      "type": "TRUSTED_CA",
      "alias": "TLS_LOCAL_GOST_CA"
    }
  ]
}
```

- `cert` — полный текст корневого сертификата в PEM.
- `type: "TRUSTED_CA"` — импорт в качестве доверенного УЦ.
- `alias` — имя записи в хранилище МП БР. В дальнейшем проверяется под тем же именем.
- `id` — идентификатор запроса для сопоставления с ответом; для нового запроса желательно новое значение.

**Получить готовый JSON из файла.** На компьютере, из корня проекта, выполнить команду с установленным Node.js. Она одинаковая для CMD, PowerShell и Bash.

```text
node -e "const fs=require('node:fs'); const cert=fs.readFileSync('gost-certs/gost-ca.crt','utf8'); console.log(JSON.stringify({type:'request',id:'import-gost-ca-1',action:'importNotUserCertificate',args:[{cert,type:'TRUSTED_CA',alias:'TLS_LOCAL_GOST_CA'}]},null,2));"
```

Скопировать весь выведенный JSON в поле запроса моста и отправить. Команда только читает файл и печатает данные, сама импорт не выполняет.

Ожидаемый ответ

```json
{
  "type": "response",
  "id": "import-gost-ca-1",
  "ok": true,
  "result": "import cert success"
}
```

Проверить наличие записи отдельным запросом

```json
{
  "type": "request",
  "id": "check-gost-ca-1",
  "action": "containsAlias",
  "args": [
    {
      "alias": "TLS_LOCAL_GOST_CA"
    }
  ]
}
```

Ожидаются `ok: true` и `result: true`. Наличие alias подтверждает запись в хранилище, но не проверяет всё TLS-соединение.

### Импорт CRL

Используется полный PEM-текст **`gost-certs/gost-ca.crl`**, включая строки `BEGIN X509 CRL` и `END X509 CRL`. Перед импортом проверить подпись и сроки по разделу 6 или 9.

```json
{
  "type": "request",
  "id": "import-gost-crl-1",
  "action": "importCRL",
  "args": [
    {
      "crl": "-----BEGIN X509 CRL-----\n...BASE64...\n-----END X509 CRL-----\n"
    }
  ]
}
```

Получить готовый запрос из реального файла на компьютере

```text
node -e "const fs=require('node:fs'); const crl=fs.readFileSync('gost-certs/gost-ca.crl','utf8'); console.log(JSON.stringify({type:'request',id:'import-gost-crl-1',action:'importCRL',args:[{crl}]},null,2));"
```

Отправить полученный JSON через мост. Для CRL в этом методе передаётся только поле `crl`, без `alias` и `type`.

Ожидаемый ответ

```json
{
  "type": "response",
  "id": "import-gost-crl-1",
  "ok": true,
  "result": true
}
```

После выпуска новой версии CRL нужно получить JSON из обновлённого файла и повторить импорт. Новый УЦ требует импорта его корня и его CRL.

### Как передаётся PEM в JSON

В файлах PEM есть настоящие переводы строк. В JSON-строке они записываются как `\n`; `JSON.stringify` делает это автоматически. Не нужно вручную заменять переносы строк или кодировать весь PEM ещё раз в Base64.

Команды выше рассчитаны на PEM, который создаётся по этой инструкции. Бинарный DER-файл нельзя читать таким способом как UTF-8 — его сначала нужно преобразовать в PEM.

## Справка

- [Расширения серверного сертификата](gost-certs/readme.md).
- [OpenSSL ca — выпуск сертификатов и CRL](https://docs.openssl.org/3.0/man1/openssl-ca/).
- [OpenSSL verify — проверка цепочки и отзыва](https://docs.openssl.org/3.0/man1/openssl-verify/).
- [Docker Compose — описание сервисов](https://docs.docker.com/reference/compose-file/services/).
