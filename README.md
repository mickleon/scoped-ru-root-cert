# scoped-ru-root-cert

Доверие корню Минцифры (Russian Trusted Root CA) только для доменов из списка.

Корень, поставленный в систему целиком, может выпустить сертификат на любой домен. Скрипт выпускает свой центр, подписывает им копию корня Минцифры с ограничением Name Constraints (RFC 5280), удаляет ключ центра и ставит пару в систему.

Разбор: [Корень Минцифры только для российских сервисов](https://nrydanov.me/posts/koren-mintsifry/).

## Запуск

```
git clone https://github.com/nrydanov/scoped-ru-root-cert
cd scoped-ru-root-cert
./install.sh                 # список minimal.txt
./install.sh мой-список.txt  # свой список, по домену на строку
./install.sh --dry-run       # только выпустить localca.crt и cross.crt
```

Нужен `openssl`, на macOS подходит встроенный. Чтобы поменять список, запустите скрипт снова.

| | куда ставится | что ещё |
|---|---|---|
| macOS | связка ключей пользователя: `localca.crt` доверенным для SSL, `cross.crt` без доверия | удаляет прежнюю пару и корень Минцифры, если он там был; корень в `System.keychain` нужно убрать вручную. Safari и Firefox берут пару из связки сами |
| Debian, Ubuntu | `/usr/local/share/ca-certificates/`, затем `update-ca-certificates` | работает для OpenSSL, curl и того, что на них построено; Firefox на Linux, скорее всего, придётся настроить отдельно |

## Списки

- `minimal.txt` — банки и НСПК, страхование, связь, доставка, Авито и hh.ru, транспорт, магазины, госсервисы.
- `candidates.txt` — финансовые и госсайты, которые уже на корне Минцифры, по журналам Certificate Transparency. Справка для ручного выбора.

Список составляется по тому, какие сайты нужны. Собирать его из выпусков Минцифры нельзя: в него попадёт и сертификат, выпущенный для подмены. Каждый домен из списка — разрешение Минцифры выпустить на него сертификат, поэтому почты и соцсетей (Яндекс, VK, Mail.ru) в нём нет.

Домен разрешает и свои поддомены. Если в сертификате сайта есть имя вне списка, сертификат отвергается целиком, поэтому имена берутся из `subjectAltName` (у Сбера это `sberbank.ru` и `sbrf.ru`):

```
echo | openssl s_client -connect example.ru:443 -servername example.ru 2>/dev/null \
  | openssl x509 -noout -issuer -ext subjectAltName
```

Правки — через pull request.

## Проверка

```
curl -sI https://www.tbank.ru/ | head -1    # из списка: открывается
curl -sI https://edu.gov.ru/ | head -1      # корень Минцифры, вне списка: ошибка
```

## Корень

`root.pem` — Russian Trusted Root CA, SHA-256 `D2:6D:2D:02:31:B7:C3:9F:92:CC:73:85:12:BA:54:10:35:19:E4:40:5D:68:B5:BD:70:3E:97:88:CA:8E:CF:31`, до 27.02.2032. Источник: `http://nuc-cdp.digital.gov.ru/cdp/rootca_ssl_rsa2022.crt`. Скрипт сверяет отпечаток.

## English

Trusts the Russian Trusted Root CA only for listed domains: creates a local CA, signs a copy of the root with X.509 Name Constraints, deletes the local CA key and installs the pair (macOS login keychain, Debian/Ubuntu system store). `minimal.txt` is the default list; `candidates.txt` lists finance and government sites already on this root, for manual picking.
