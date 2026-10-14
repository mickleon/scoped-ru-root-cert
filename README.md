# scoped-ru-root

Доверие корню Минцифры (Russian Trusted Root CA) только для сайтов из списка.

Российские банки и госсервисы переходят на сертификаты от корня Минцифры. Если поставить этот корень в систему целиком, он сможет выпустить сертификат на любой домен, и система его примет. Скрипт делает так, чтобы корень работал только для доменов из списка: выпускает свой центр, подписывает им копию корня Минцифры с расширением Name Constraints (RFC 5280) и удаляет ключ своего центра.

Подробный разбор: [Почему Т-Банк уводит на домен, похожий на фишинговый](https://nrydanov.me/posts/zerkala-t-banka/).

## Как пользоваться

```
git clone https://github.com/nrydanov/scoped-ru-root
cd scoped-ru-root
./install.sh                # список minimal.txt
./install.sh мой-список.txt # свой список, по домену на строку
```

Скрипт выпускает `localca.crt` и `cross.crt` и ставит их в систему; с `--dry-run` только выпускает. Нужен `openssl` (на macOS подходит встроенный).

Установка:

- **macOS** — `localca.crt` ставится доверенным для SSL в связку ключей пользователя, `cross.crt` — без доверия. Из той же связки удаляются прежняя пара и сам корень Минцифры, если он там был: доверенный оригинал обходит ограничение. Если корень стоит в системной связке (`/Library/Keychains/System.keychain`), его нужно убрать вручную. Safari, Chrome и Firefox (через `security.enterprise_roots.enabled`) берут пару из связки сами.
- **Linux** (Debian, Ubuntu) — оба файла кладутся в `/usr/local/share/ca-certificates/` и вызывается `update-ca-certificates`. Это хранилище читают OpenSSL, curl и всё на них построенное. Firefox на Linux системное хранилище, скорее всего, не читает: там `localca.crt` нужно импортировать в настройках как доверенный для сайтов, а `cross.crt` — без доверия.

Чтобы поменять список, достаточно запустить скрипт снова: он выпустит новую пару и заменит старую.

Ключ своего центра после подписи удаляется. Поэтому центр нельзя использовать ни для чего другого, а кто бы ни получил доступ к вашей машине позже, выпустить им сертификат не сможет.

## Списки

- `minimal.txt` — то, чем пользуется почти каждый: крупнейшие банки и НСПК, страхование, связь, почта и доставка, Авито и hh.ru, РЖД и авиакомпании, торговые сети и маркетплейсы, госсервисы. Часть сайтов уже на корне Минцифры, часть добавлена заранее.
- `candidates.txt` — справка: банки, страховые, платёжные и госсайты, которые уже на корне Минцифры, по журналам Certificate Transparency. Нужное из него переносите в свой список.

Сайты, добавленные заранее, — это сознательный риск: если Минцифры выпустит на такой сайт сертификат, система его примет, хотя сейчас отказала бы. Поэтому в список не входят почта и соцсети (Яндекс, VK, Mail.ru): для них подмена опаснее всего, а неожиданный сертификат Минцифры — повод насторожиться.

Список составляется руками, по тому, какие сайты нужны, а не по тому, на какие домены Минцифры уже выпустило сертификаты. Если собирать список из журналов Certificate Transparency, в него попадёт и чужой сайт, на который Минцифры выпустит сертификат для подмены, и ограничение потеряет смысл.

Домен разрешает и все свои поддомены. Имена берутся из `subjectAltName` сертификата, а не из адресной строки: если в сертификате есть хоть одно имя вне списка, он отвергается целиком (так у Сбера `sberbank.ru` идёт вместе с `sbrf.ru`). Проверить имена сайта:

```
echo | openssl s_client -connect example.ru:443 -servername example.ru 2>/dev/null \
  | openssl x509 -noout -issuer -ext subjectAltName
```

Правки — через pull request.

## Проверка

```
curl -sI https://www.tbank.ru/ | head -1        # домен из списка: открывается
curl -sI https://<домен вне списка>/ | head -1  # корень Минцифры, но не из списка: ошибка
```

## Корень

`root.pem` — Russian Trusted Root CA, SHA-256 `D2:6D:2D:02:31:B7:C3:9F:92:CC:73:85:12:BA:54:10:35:19:E4:40:5D:68:B5:BD:70:3E:97:88:CA:8E:CF:31`, действует до 27.02.2032. Его же присылают серверы банков в цепочке, и по ссылке из сертификатов Минцифры его можно скачать: `http://nuc-cdp.digital.gov.ru/cdp/rootca_ssl_rsa2022.crt`. Скрипт сверяет отпечаток перед работой.

---

## English

Trust the Russian Ministry of Digital Development root CA (Russian Trusted Root CA) only for listed domains. The script creates a local CA, uses it to sign a copy of the Ministry root with X.509 Name Constraints, deletes the local CA key and puts the pair into the macOS login keychain or the Debian/Ubuntu system store. `minimal.txt` covers the largest banks and key government services; `candidates.txt` is a Certificate Transparency snapshot of finance and government sites already on the Ministry root, for picking by hand. The list is curated manually on purpose: building it from what the Ministry CA has issued would defeat the restriction.
