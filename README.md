<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Зізевський Михайло Юрійович |
| Група | Кбзі-2.02 |
| Ідентифікатор | cs-202-08 |
| Дата виконання | 14.11, 16-17.11 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `front.cs-202-08.lab` |
| Імена для сертифіката | `www.cs-202-08.lab`, `docs.cs-202-08.lab` |
| Ваш сервер (IP) | `10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->
Стенд перезапускався після завдання 3: корінь Pebble змінився з `Pebble Root CA 5c4310` на `Pebble Root CA 3c1938`; сертифікат видано повторно (кроки 3.1, 3.3, 3.6).

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
[1] 36
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: front.cs-202-08.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
10:13:46.718598 IP 10.53.1.1.47864 > 10.53.0.80.80: Flags [P.], seq 1:83, ack 1, win 502, options [nop,nop,TS val 1293267543 ecr 2527894760], length 82: HTTP: GET / HTTP/1.1
M..W....GET / HTTP/1.1
Host: front.cs-202-08.lab
server: web
Host: front.cs-202-08.lab
```

Вивід кроку 1.2:

```text
[1] 42
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: front.cs-202-08.lab
SNI: front.cs-202-08.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` |є (GET / HTTP/1.1) |немає (0) |
| заголовок `Host` |є (Host: front.cs-202-08.lab)|немає|
| тіло відповіді (`server: web`) | є |немає |
| ім'я віддаленого сервера (кількість входжень) | 2 (заголовок Host у запиті і тіло відповіді) | 1 (розширення SNI у ClientHello)|

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
Ім'я сервера передається в повідомленні **ClientHello**, у розширенні **`server_name` (SNI)**. Воно йде відкритим текстом, бо шифрування ще не почалося. Саме це єдине входження імені в дампі HTTPS.

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
SNI є розширенням TLS, яке клієнт надсилає в ClientHello. У HTTP по порту 80 рукостискання TLS немає, тому сервер не отримує SNI і рядок залишається порожнім. У HTTPS клієнт передав ім'я в ClientHello, і сервер його виводить.

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Версію HTTP визначив механізм **ALPN**. Під час рукостискання `curl` запропонував протоколи `h2` і `http/1.1`, сервер обрав `h2`, тому запит виконано по HTTP/2. У звичайному HTTP немає TLS, а отже й ALPN, тому використовується HTTP/1.1.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=330
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=987
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```
Додатково (для стовпця `Length` таблиці):

```text
root@cs-202-08:/lab/work# grep -a -E '^(Sent|Received) Record|Content Type|Length' t2-13.txt | head -60
Sent Record
  Content Type = Handshake (22)
  Length = 334
    ClientHello, Length=330
Received Record
  Content Type = Handshake (22)
  Length = 122
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
  Content Type = ApplicationData (23)
  Length = 36
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Length = 1008
  Inner Content Type = Handshake (22)
    Certificate, Length=987
Received Record
  Content Type = ApplicationData (23)
  Length = 96
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Sent Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Length = 19
  Inner Content Type = Alert (21)
```

Вивід кроку 2.2 (TLS 1.2):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=225
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=982
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=110
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | К→С | Handshake (22) | — | ClientHello | 334 (повідомлення 330) |
| 2 | С→К | Handshake (22) | — | ServerHello | 122 (повідомлення 118) |
| 3 | С→К | ChangeCipherSpec (20) | — | ChangeCipherSpec | 1 |
| 4 | С→К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 36 (повідомлення 15) |
| 5 | С→К | ApplicationData (23) | Handshake (22) | Certificate | 1008 (повідомлення 987) |
| 6 | С→К | ApplicationData (23) | Handshake (22) | CertificateVerify | 96 (повідомлення 75) |
| 7 | С→К | ApplicationData (23) | Handshake (22) | Finished | 69 (повідомлення 48) |
| 8 | К→С | ChangeCipherSpec (20) | — | ChangeCipherSpec | 1 |
| 9 | К→С | ApplicationData (23) | Handshake (22) | Finished | 69 (повідомлення 48) |
| 10 | К→С | ApplicationData (23) | Alert (21) | Alert | 19 |

Вивід кроків 2.3 і 2.4:

```text
extension_type=server_name(0), length=24
          0000 - 00 16 00 00 13 66 72 6f-6e 74 2e 63 73 2d 32   .....front.cs-2
          000f - 30 32 2d 30 38 2e 6c 61-62                     02-08.lab
        extension_type=ec_point_formats(11), length=4

[1] 45
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 51
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт |330 |225 |
| Номер першого запису, тип якого спостерігач не бачить | 4|немає: тип кожного запису відкритий |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1|2 |
| Чи видно сертифікат у дампі (результат 2.4) |ні (0)|так(2) |
| Обраний протокол ALPN |h2 |h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
За рядками `Content Type = ChangeCipherSpec (20)` після ServerHello і далі `Content Type = ApplicationData (23)` з `Inner Content Type = Handshake (22)` на записі EncryptedExtensions: з нього шифрування починається. Спостерігач замість справжнього типу кожного наступного запису бачить лише зовнішній тип ApplicationData (23). Справжній тип (`Inner Content Type`) показує `openssl`, бо він розшифровує записи.

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
ClientHello у TLS 1.3 довший (330 байт проти 225), бо клієнт одразу надсилає в ньому розширення `key_share` з відкритою частиною ключа для обміну ключами. Завдяки цьому сервер може відповісти вже зі спільним ключем, і рукостискання скорочується на один RTT.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
Значення `server_name` має довжину 24 байти для імені `front.cs-202-08.lab` з 19 символів. Воно складається з: довжини списку імен (2 байти, `00 16`), типу імені host_name (1 байт, `00`), довжини імені (2 байти, `00 13`) і самого імені (19 байт). Разом 2 + 1 + 2 + 19 = 24. Для прикладу з методички: 5 службових байтів + 12 символів = 17.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
ChangeCipherSpec у TLS 1.3 нічого не змінює, він потрібен лише для сумісності з проміжним мережевим обладнанням, яке очікує побачити цей запис і інакше може розірвати з'єднання. Тому рукостискання 1.3 імітує структуру 1.2, а запис ігнорується.


## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 5c4310
notBefore=Nov 16 11:43:38 2026 GMT
notAfter=Nov 16 11:43:38 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
2026/11/16 11:55:25 [INFO] [www.cs-202-08.lab, docs.cs-202-08.lab] acme: Obtaining SAN certificate
2026/11/16 11:55:25 [INFO] [www.cs-202-08.lab] AuthURL: https://ca.lab:14000/authZ/FrGdQAeKguvUYH24wacpC-10QRInWlxhAs_FVWj3t-c
2026/11/16 11:55:25 [INFO] [docs.cs-202-08.lab] AuthURL: https://ca.lab:14000/authZ/e_ETc46XqRjtiP8KEn8q6qcdvcFjvBvx1bNWZPu24Q0
2026/11/16 11:55:25 [INFO] [docs.cs-202-08.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 11:55:25 [INFO] [docs.cs-202-08.lab] acme: use http-01 solver
2026/11/16 11:55:25 [INFO] [www.cs-202-08.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 11:55:25 [INFO] [www.cs-202-08.lab] acme: use http-01 solver
2026/11/16 11:55:25 [INFO] [docs.cs-202-08.lab] acme: Trying to solve HTTP-01
2026/11/16 11:55:28 [INFO] [docs.cs-202-08.lab] The server validated our request
2026/11/16 11:55:28 [INFO] [www.cs-202-08.lab] acme: Trying to solve HTTP-01
2026/11/16 11:55:33 [INFO] [www.cs-202-08.lab] The server validated our request
2026/11/16 11:55:33 [INFO] [www.cs-202-08.lab, docs.cs-202-08.lab] acme: Validations succeeded; requesting certificates
2026/11/16 11:55:34 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/16 11:55:34 [INFO] [www.cs-202-08.lab] Server responded with a certificate.
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
subject=CN = www.cs-202-08.lab
issuer=CN = Pebble Intermediate CA 130497
notBefore=Nov 16 11:55:34 2026 GMT
notAfter=Nov 16 11:55:33 2031 GMT
serial=27030D0D6DFEB8CC
X509v3 Subject Alternative Name: 
    DNS:www.cs-202-08.lab, DNS:docs.cs-202-08.lab
subject=CN = Pebble Intermediate CA 130497
issuer=CN = Pebble Root CA 5c4310
```

Вивід кроку 3.5:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 12:05:51 [notice] 56#56: signal process started
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.cs-202-08.lab
   i:CN = Pebble Intermediate CA 130497
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 11:55:34 2026 GMT; NotAfter: Nov 16 11:55:33 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 12:08:14 [notice] 64#64: signal process started
server: cs-202-08 (https)
Host: www.cs-202-08.lab
SNI: www.cs-202-08.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.cs-202-08.lab
   i:CN = Pebble Intermediate CA 130497
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 11:55:34 2026 GMT; NotAfter: Nov 16 11:55:33 2031 GMT
 1 s:CN = Pebble Intermediate CA 130497
   i:CN = Pebble Root CA 5c4310
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 11:43:38 2026 GMT; NotAfter: Nov 16 11:43:38 2056 GMT
---
Verify return code: 0 (ok)
server: cs-202-08 (https)
Host: docs.cs-202-08.lab
SNI: docs.cs-202-08.lab
protocol: TLSv1.3
```

Під час роботи стенд перезапускався (після завдання 3). Корінь Pebble змінився з Pebble Root CA 5c4310 на Pebble Root CA 3c1938, тому сертифікат видано повторно (кроки 3.1, 3.3, 3.6).

### Повторна видача після перезапуску стенда:

```text
root@cs-202-08:/lab/work# cd /lab/work
curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-202-08.lab/ 2>&1 | head -1
curl: (60) SSL certificate problem: unable to get local issuer certificate
root@cs-202-08:/lab/work# openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
subject=CN = Pebble Root CA 3c1938
notBefore=Nov 16 12:28:01 2026 GMT
notAfter=Nov 16 12:28:01 2056 GMT
root@cs-202-08:/lab/work# mv /lab/work/lego /lab/work/lego-old
lego --server $ACME_DIR --email admin@cs-202-08.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.cs-202-08.lab --domains docs.cs-202-08.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log

2026/11/16 12:55:34 [INFO] [www.cs-202-08.lab, docs.cs-202-08.lab] acme: Obtaining SAN certificate
2026/11/16 12:55:35 [INFO] [www.cs-202-08.lab] AuthURL: https://ca.lab:14000/authZ/KKB4-LHYaCWev8Wh3l49VUJf5AuUvA6mkAuDa1y68mE
2026/11/16 12:55:35 [INFO] [docs.cs-202-08.lab] AuthURL: https://ca.lab:14000/authZ/ZTYgyScOJWgagbHBglyP7rj02bu-L4GLhc7QQNRQjFw
2026/11/16 12:55:35 [INFO] [www.cs-202-08.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 12:55:35 [INFO] [www.cs-202-08.lab] acme: use http-01 solver
2026/11/16 12:55:35 [INFO] [docs.cs-202-08.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 12:55:35 [INFO] [docs.cs-202-08.lab] acme: use http-01 solver
2026/11/16 12:55:35 [INFO] [www.cs-202-08.lab] acme: Trying to solve HTTP-01
2026/11/16 12:55:40 [INFO] [www.cs-202-08.lab] The server validated our request
2026/11/16 12:55:40 [INFO] [docs.cs-202-08.lab] acme: Trying to solve HTTP-01
2026/11/16 12:55:44 [INFO] [docs.cs-202-08.lab] The server validated our request
2026/11/16 12:55:44 [INFO] [www.cs-202-08.lab, docs.cs-202-08.lab] acme: Validations succeeded; requesting certificates
2026/11/16 12:55:44 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/16 12:55:44 [INFO] [www.cs-202-08.lab] Server responded with a certificate.
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:25 +0000] "GET /.well-known/acme-challenge/dERQ3xHy79HEpcLbRwok1TKenx6aAlij5WNZiMA_tos HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:11:55:28 +0000] "GET /.well-known/acme-challenge/Ilm_ohCBBe3cYWuvmONlpe9NEg4tO3X2eVhdSmX1VJU HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:35 +0000] "GET /.well-known/acme-challenge/ayvuZ-66k12zLUBBTfo8YILe9kxOb8IF4LgD3G1BxNg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:35 +0000] "GET /.well-known/acme-challenge/ayvuZ-66k12zLUBBTfo8YILe9kxOb8IF4LgD3G1BxNg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:35 +0000] "GET /.well-known/acme-challenge/ayvuZ-66k12zLUBBTfo8YILe9kxOb8IF4LgD3G1BxNg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:40 +0000] "GET /.well-known/acme-challenge/Jbs_TuRPhj30d-JkwuVLOXsV0C9xjVpWi5OUvPmaEG4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:40 +0000] "GET /.well-known/acme-challenge/Jbs_TuRPhj30d-JkwuVLOXsV0C9xjVpWi5OUvPmaEG4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:12:55:40 +0000] "GET /.well-known/acme-challenge/Jbs_TuRPhj30d-JkwuVLOXsV0C9xjVpWi5OUvPmaEG4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
root@cs-202-08:/lab/work# cd /lab/work/lego/certificates
cat www.cs-202-08.lab.crt www.cs-202-08.lab.issuer.crt > /lab/tls/server.crt
cp www.cs-202-08.lab.key /lab/tls/server.key
web-reload; sleep 2
curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-202-08.lab/
openssl s_client -connect www.cs-202-08.lab:443 -servername www.cs-202-08.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
curl -sS --cacert /lab/tls/acme-root.crt https://docs.cs-202-08.lab/
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 12:56:24 [notice] 83#83: signal process started
server: cs-202-08 (https)
Host: www.cs-202-08.lab
SNI: www.cs-202-08.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.cs-202-08.lab
   i:CN = Pebble Intermediate CA 211c83
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 12:55:44 2026 GMT; NotAfter: Nov 16 12:55:43 2031 GMT
 1 s:CN = Pebble Intermediate CA 211c83
   i:CN = Pebble Root CA 3c1938
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 12:28:01 2026 GMT; NotAfter: Nov 16 12:28:01 2056 GMT
---
Verify return code: 0 (ok)
server: cs-202-08 (https)
Host: docs.cs-202-08.lab
SNI: docs.cs-202-08.lab
protocol: TLSv1.3
```

У журналі nginx перші 6 рядків (час 11:55) — перша видача, останні 6 (час 12:55) — повторна.

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | Pebble Intermediate CA 130497 |
| Видавець сертифіката видавця |Pebble Root CA 5c4310|
| Імена в SAN |www.cs-202-08.lab, docs.cs-202-08.lab |
| Строк дії (від – до) | 16.11.2026 11:55:34 GMT – 16.11.2031 11:55:33 GMT (5 років)|
| Тип ключа сервера і алгоритм підпису | ключ id-ecPublicKey, 256 біт (ECDSA); підпис RSA-SHA256|
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 3.5: 1; 3.6: 2|

Значення наведено за *першою видачею*; після перезапуску сертифікат видано повторно (видавець Pebble Intermediate CA 211c83, корінь Pebble Root CA 3c1938, строк 12:55:44 – 12:55:43 GMT).

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Володіння іменами перевірив центр сертифікації Pebble за протоколом ACME методом HTTP-01. `lego` отримав від Pebble випробування для кожного імені й розмістив відповідний файл у `/var/www/acme/.well-known/acme-challenge/` на сервері. Pebble (`LetsEncrypt-Pebble-VA`, адреса `10.53.0.10`) сам звернувся до цих імен по HTTP на порт 80 і прочитав файл. Це підтверджують рядки журналу nginx `GET /.well-known/acme-challenge/…` з кодом 200 від `10.53.0.10`, а також рядки `lego`: `Trying to solve HTTP-01` і `The server validated our request` для обох імен.


2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
У кроці 3.5 сервер надіслав лише сертифікат сервера, без сертифіката видавця. У `--cacert` вказано тільки корінь Pebble, а сертифікат сервера підписано проміжним CA, якого клієнт не має. Клієнт не може побудувати ланцюжок до кореня, тому перевірка завершується кодом 21 (`unable to verify the first certificate`). Корінь правильний і сертифікат виданий правильно: проблема лише в тому, що сервер не надіслав проміжний сертифікат.

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?
`id-ecPublicKey, 256 (bit)` описує відкритий ключ самого сертифіката сервера: це еліптична крива (ECDSA), 256 біт. `RSA-SHA256` — алгоритм підпису, яким видавець підписав цей сертифікат. Підписував проміжний CA, чий ключ RSA (у виводі 3.6 видно `rsaEncryption, 2048 (bit)`). Суперечності немає: це ключі різних учасників, один належить серверу, інший — центру сертифікації.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Після перезапуску стенда Pebble створив новий корінь (`5c4310` → `3c1938`) і новий проміжний сертифікат (`130497` → `211c83`). Сервер ще надсилав старий сертифікат, і `curl --cacert` з новим коренем завершився помилкою `unable to get local issuer certificate`: новий корінь не знає старого проміжного CA. Сертифікат довелося видати заново. У реальній інфраструктурі цьому відповідає ротація або відкликання сертифікатів центру сертифікації (чи втрата довіри до нього): усі видані ним сертифікати треба перевидати.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
=== f1.cs-202-08.lab
server: web, f1
Certificate chain
 0 s:CN = f1.cs-202-08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 19:50:31 2026 GMT; NotAfter: Feb 12 19:50:31 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 19:50:30 2026 GMT; NotAfter: Nov 13 19:50:30 2031 GMT
---
Verify return code: 0 (ok)
=== f2.cs-202-08.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f2.cs-202-08.lab
   i:CN = f2.cs-202-08.lab
   v:NotBefore: Nov 14 19:50:31 2026 GMT; NotAfter: Feb 12 19:50:31 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f3.cs-202-08.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f3.cs-202-08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 28 19:50:31 2026 GMT; NotAfter: Oct 25 19:50:31 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 19:50:30 2026 GMT; NotAfter: Nov 13 19:50:30 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f4.cs-202-08.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.cs-202-08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 19:50:31 2026 GMT; NotAfter: Feb 12 19:50:31 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f5.cs-202-08.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f5.cs-202-08.lab'
Certificate chain
 0 s:CN = old.cs-202-08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 14 19:50:31 2026 GMT; NotAfter: Feb 12 19:50:31 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 14 19:50:30 2026 GMT; NotAfter: Nov 13 19:50:30 2031 GMT
---
Verify return code: 0 (ok)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | немає (відповідь `server: web, f1`) | 0 (ok) | усі пройдено, сервер справний | `Verify return code: 0 (ok)`, ланцюжок із 2 сертифікатів | нічого |
| f2 | `self-signed certificate` | 18 (self-signed certificate) | довіра до видавця: сертифікат самопідписаний | `s:CN = f2.cs-202-08.lab` і `i:CN = f2.cs-202-08.lab` (видавець = власник) | отримати сертифікат від довіреного центру сертифікації (наприклад, за ACME) |
| f3 | `certificate has expired` | 10 (certificate has expired) | строк дії | `NotAfter: Oct 25 19:50:31 2026 GMT` (минув до сьогоднішньої дати 16.11.2026) | перевидати сертифікат і налаштувати автоматичне поновлення |
| f4 | `unable to get local issuer certificate` | 21 (unable to verify the first certificate) | побудова ланцюжка: сервер не надіслав проміжний сертифікат | у ланцюжку лише `0 s:CN = f4…`, елемента `1` немає; видавець `OPISM Stand Intermediate CA` | додати проміжний сертифікат у файл сертифіката (повний ланцюжок) |
| f5 | `no alternative certificate subject name matches target host name 'f5.cs-202-08.lab'` | 0 (ok) | відповідність імені (SAN) | `0 s:CN = old.cs-202-08.lab` у відповіді на запит до `f5` | випустити сертифікат, у SAN якого є `f5.cs-202-08.lab` |


1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?
Це несправність «сервер не надіслав проміжний сертифікат». У виводі f4 сервер надсилає лише один сертифікат (елемента `1` у ланцюжку немає), а видавець у ньому — `OPISM Stand Intermediate CA`, той самий проміжний CA, який на справному f1 підписаний `OPISM Stand Root CA` і проходить перевірку (`Verify return code: 0`). Отже, сертифікат виданий довіреною ієрархією, але проміжна ланка не передана. Якби сертифікат був виданий центром, якому клієнт не довіряє, видавець у ланцюжку мав би ім'я, якого немає серед довірених (`$STAND_CA`), і додавання проміжного сертифіката на сервері не допомогло б. У цьому варіанті сервера з такою несправністю немає, тому цю відмінність описано за загальним правилом, а не за виводом.

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.
`openssl s_client` без ключа `-verify_hostname` перевіряє лише ланцюжок підписів і строки, а не відповідність імені, тому для f5 він повертає `0 (ok)`: ланцюжок до кореня побудований. `curl` додатково звіряє ім'я сайту з SAN сертифіката, у f5 це `old.cs-202-08.lab`, а запит іде до `f5.cs-202-08.lab`, тому `curl` повідомляє про помилку.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?
З `curl -k` відповіли б усі п'ять серверів: `-k` вимикає перевірку сертифіката, а рукостискання TLS завершується в усіх. Перевірка командою:
```text
root@cs-202-08:/lab/work# for i in 1 2 3 4 5; do curl -sk https://f$i.cs-202-08.lab/ | head -1; done
server: web, f1
server: web, f2
server: web, f3
server: web, f4
server: web, f5
```
Втрачається автентифікація сервера: клієнт не знає, чи це справжній сервер, а не підмінений (атака посередника). Шифрування лишається, але з невідомим співрозмовником.

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 RTT | 1 RTT | 0 |
| `time_appconnect` | 2 RTT | 3 RTT | 0 |
| `time_starttransfer` | 3 RTT | 4 RTT | 1 RTT |

Прогноз не враховує пошук імені (+1 RTT до кожного значення в нових з'єднаннях).

Вивід кроків 5.2 і 5.3:

```text
0.080904 0.121117 0.167386 0.207718 0.207795 2 1
0.040508 0.080689 0.124131 0.164544 0.164607 2 1
0.040537 0.080738 0.124174 0.164555 0.164615 2 1
0.040599 0.080802 0.164388 0.204756 0.204814 2 1
0.040517 0.080683 0.164236 0.204584 0.204643 2 1
0.040539 0.080839 0.165373 0.205745 0.205813 2 1
0.040526 0.080715 0.124189 0.165449 0.165513 2 1
0.000013 0.000000 0.000000 0.040268 0.040326 2 0
0.000017 0.000000 0.000000 0.040365 0.040499 2 0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 40,5 | 80,7 | 124,2 | 164,6 | 164,6 | 1 |
| TLS 1.2, нове | 40,5 | 80,8 | 164,4 | 204,8 | 204,8 | 1 |
| 5.3, перший запит | 40,5 | 80,7 | 124,2 | 165,4 | 165,5 | 1 |
| 5.3, другий запит | 0,0 | 0,0 | 0,0 | 40,3 | 40,3 | 0 |
| 5.3, третій запит | 0,0 | 0,0 | 0,0 | 40,4 | 40,5 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено:
**RTT:** ≈ 40 мс. Визначено з різниці `time_connect − time_namelookup` (рукостискання TCP — рівно 1 RTT): 80,7 − 40,5 ≈ 40,2 мс. Підтверджує другий запит на повторно використаному з'єднанні: `starttransfer` ≈ 40,3 мс — це один RTT на сам запит.

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?
У RTT (≈40 мс) значення розкладаються так. TLS 1.3, нове з'єднання: `namelookup` ≈ 1 RTT, `connect` ≈ 2, `appconnect` ≈ 3, `starttransfer` ≈ 4. TLS 1.2: `appconnect` ≈ 4, `starttransfer` ≈ 5, бо рукостискання довше на один RTT. Повторне з'єднання: DNS і TCP/TLS немає (`num_connects` = 0), лише 1 RTT на запит. Відхилення від цілих RTT складають 3–5 мс (наприклад, 124,2 замість 120) і пояснюються часом обчислень: обмін ключами, підпис і перевірка сертифіката, обробка запиту сервером. Окреме відхилення — перше вимірювання TLS 1.3: `namelookup` 80,9 мс, тобто 2 RTT, і весь рядок зсунутий на цей один RTT.

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?
`namelookup` ≈ 40 мс, бо запит до резолвера `10.53.0.53` проходить через інтерфейс зі штучною затримкою і коштує 1 RTT, навіть якщо відповідь є в кеші резолвера: кеш економить роботу резолвера, а не дорогу до нього. Приблизно вдвічі більшим значення було б, якщо запису в кеші немає і резолверу доводиться опитувати авторитетний сервер зони, який працює в моєму контейнері за каналом із затримкою (приклад — перший рядок виводу 5.2: 80,9 мс). Це припущення, яке не було перевірено.

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.
Розрахунок для 200 послідовних запитів (RTT = 40 мс, без урахування DNS, якщо ім'я кешоване клієнтом):
   - нове з'єднання TLS 1.3 на кожен запит: 3 RTT (TCP + TLS + запит) = 120 мс, 200 × 120 мс = **24 с**;
   - нове з'єднання TLS 1.2: 4 RTT = 160 мс, 200 × 160 мс = **32 с**;
   - одне з'єднання: 3 RTT на встановлення і запит, далі 199 × 1 RTT, разом 202 RTT ≈ **8,1 с**.
   
   Якщо кожне з'єднання ще й виконує власний пошук імені (+1 RTT), TLS 1.3 дасть 32 с, TLS 1.2 — 40 с.
   
## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:
У кроці 3.5 `curl` повідомив `SSL certificate problem: self-signed certificate`, хоча очікувалося `unable to get local issuer certificate`: так само, як на сервері f4 у завданні 4, де сервер також не надсилає проміжний сертифікат. `openssl s_client` у тому самому кроці 3.5 показав один сертифікат із видавцем `Pebble Intermediate CA 130497` і `Verify return code: 21 (unable to verify the first certificate)`, тобто сервер уже віддавав новий сертифікат. Імовірно, `curl` виконано одразу після `web-reload`, коли nginx ще міг віддавати тимчасовий самопідписаний сертифікат. Окремою перевіркою це не підтверджено.

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:
Чому в першому вимірюванні TLS 1.3 (рядок 1 виводу 5.2: `0.080904 0.121117 0.167386 0.207718 …`) `time_namelookup` становить 80,9 мс, тобто вдвічі більше, ніж у решти запитів (≈ 40,5 мс)? Перевірено: різниця `connect − namelookup` у цьому рядку дорівнює 40,2 мс, як і в інших, тож додаткова затримка виникла саме на пошуку імені, а не на TCP чи TLS; у п'яти наступних вимірюваннях значення стабільне. Припущення: імені не було в кеші резолвера, і резолвер опитував авторитетний сервер зони, що працює в контейнері за каналом із затримкою.

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Claude Sonnet 5.5 |https://github.com/TheFactRio/opism-lab-TLS
Мені потрібно виконати цю роботу. Я буду писати, ти повинен перевіряти на помилки та консультувати. | Отримано консультанцію з виконання роботи, прямого впливу на звіт не було. |
| 2 | Claude Sonnet 5.5 | допоможи виправити формулювання у README. додай, якщо потрібно точності в поясненнях | Пропозиції перевірено і використано для редагування звіту. |
| 3 | Claude Sonnet 5.5 | перевір README. напиши конкретні помилки й потрібні виправлення, не змінюючи результати виконаних команд.| Зауваження перевірено і підтверджені помилки виправлено у звіті. |