<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Примаченко Ксенія Анатоліївна |
| Група | КБЗІ-2.02 |
| Ідентифікатор | cs-202-20 |
| Дата виконання | 23.11.26 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.5.8 25 Aug 2026 (Library: OpenSSL 3.5.8 25 Aug 2026) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `edge.cs-202-20.lab` |
| Імена для сертифіката | `www.cs-202-20.lab`, `news.cs-202-20.lab` |
| Ваш сервер (IP) | `10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
[1] 35
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: edge.cs-202-20.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
13:56:04.737590 IP 10.53.1.1.44268 > 10.53.0.80.80: Flags [P.], seq 1:82, ack 1, win 502, options [nop,nop,TS val 1614533079 ecr 817496959], length 81: HTTP: GET / HTTP/1.1
`;..0...GET / HTTP/1.1
Host: edge.cs-202-20.lab
server: web
Host: edge.cs-202-20.lab
```

Вивід кроку 1.2:

```text
[1] 41
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: edge.cs-202-20.lab
SNI: edge.cs-202-20.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```

Додатково (кількість входжень імені в HTTP-дампі, для таблиці):
```text
root@cs-202-20:/lab/work# grep -a -o 'edge.cs-202-20.lab' t1-http.txt | wc -l
2
```



| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | є (`GET / HTTP/1.1`) | немає (0) |
| заголовок `Host` | є (`Host: edge.cs-202-20.lab`) | немає |
| тіло відповіді (`server: web`) | є | немає (0) |
| ім'я віддаленого сервера (кількість входжень) | 2 (заголовок `Host` у запиті і тіло відповіді) | 1 (розширення SNI у ClientHello) |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
Ім'я сервера передається в повідомленні **ClientHello**, у розширенні **`server_name` (SNI)**. Воно йде відкритим текстом, бо шифрування ще не почалося. Це єдине входження імені в дампі HTTPS.
2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
SNI є розширенням TLS, яке клієнт надсилає в ClientHello. У HTTP по порту 80 рукостискання TLS немає, тому сервер не отримує SNI і рядок порожній. У HTTPS клієнт передав ім'я в ClientHello, і сервер його виводить.
3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Версію HTTP визначив механізм **ALPN**. Під час рукостискання `curl` запропонував `h2` і `http/1.1`, сервер обрав `h2`, тому запит виконано по HTTP/2. У звичайному HTTP немає TLS, а отже й ALPN, тому використовується HTTP/1.1.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=329
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
    Certificate, Length=984
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=74
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

```text
root@cs-202-20:/lab/work# grep -a -E '^(Sent|Received) Record|Content Type|Length' t2-13.txt | head -60
Sent Record
  Content Type = Handshake (22)
  Length = 333
    ClientHello, Length=329
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
  Length = 1005
  Inner Content Type = Handshake (22)
    Certificate, Length=984
Received Record
  Content Type = ApplicationData (23)
  Length = 95
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=74
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
    ClientHello, Length=224
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=979
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=112
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
| 1 | К→С | Handshake (22) | — | ClientHello | 333 (повідомлення 329) |
| 2 | С→К | Handshake (22) | — | ServerHello | 122 (повідомлення 118) |
| 3 | С→К | ChangeCipherSpec (20) | — | ChangeCipherSpec | 1 |
| 4 | С→К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 36 (повідомлення 15) |
| 5 | С→К | ApplicationData (23) | Handshake (22) | Certificate | 1005 (повідомлення 984) |
| 6 | С→К | ApplicationData (23) | Handshake (22) | CertificateVerify | 95 (повідомлення 74) |
| 7 | С→К | ApplicationData (23) | Handshake (22) | Finished | 69 (повідомлення 48) |
| 8 | К→С | ChangeCipherSpec (20) | — | ChangeCipherSpec | 1 |
| 9 | К→С | ApplicationData (23) | Handshake (22) | Finished | 69 (повідомлення 48) |
| 10 | К→С | ApplicationData (23) | Alert (21) | Alert | 19 |

У стовпці `Length` наведено довжину запису; у дужках довжина повідомлення рукостискання з виводу `-trace`.

Вивід кроків 2.3 і 2.4:

```text
extension_type=server_name(0), length=23
          0000 - 00 15 00 00 12 65 64 67-65 2e 63 73 2d 32 30   .....edge.cs-20
          000f - 32 2d 32 30 2e 6c 61 62-                       2-20.lab
        extension_type=ec_point_formats(11), length=4

[1] 61
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 67
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
| Довжина ClientHello, байт | 329 (запис 333) | 224 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | немає: тип кожного запису відкритий |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 | 2 |
| Чи видно сертифікат у дампі (результат 2.4) | ні (0) | так (2) |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
Шифрування починається після ServerHello і запису `Content Type = ChangeCipherSpec (20)`: перший зашифрований запис — № 4 (`Content Type = ApplicationData (23)`, `Inner Content Type = Handshake (22)`, EncryptedExtensions). Спостерігач замість справжнього типу кожного наступного запису бачить лише зовнішній тип ApplicationData (23). Справжній тип (`Inner Content Type`) показує `openssl`, бо він розшифровує записи.
2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
ClientHello у TLS 1.3 довший (329 байт проти 224), бо клієнт одразу надсилає в ньому розширення `key_share` з відкритою частиною ключа для обміну ключами. Завдяки цьому сервер може відповісти вже зі спільним ключем, і рукостискання скорочується на один RTT.
3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
Значення `server_name` має довжину 23 байти для імені `edge.cs-202-20.lab` з 18 символів. Воно складається з довжини списку імен (2 байти, `00 15`), типу імені host_name (1 байт, `00`), довжини імені (2 байти, `00 12`) і самого імені (18 байт): 2 + 1 + 2 + 18 = 23. Для прикладу з методички: 5 службових байтів + 12 символів = 17.
4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
У TLS 1.3 ChangeCipherSpec нічого не змінює: шифрування вмикається само після ServerHello. Запис потрібен лише для сумісності з проміжним мережевим обладнанням (middlebox), яке очікує його побачити й інакше може розірвати з'єднання. Отримувач запис ігнорує.

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 7870d8
notBefore=Nov 23 13:54:31 2026 GMT
notAfter=Nov 23 13:54:31 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
2026/11/23 14:15:15 No key found for account admin@cs-202-20.lab. Generating a P256 key.
2026/11/23 14:15:15 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@cs-202-20.lab/keys/admin@cs-202-20.lab.key
2026/11/23 14:15:15 [INFO] acme: Registering account for admin@cs-202-20.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/23 14:15:15 [INFO] [www.cs-202-20.lab, news.cs-202-20.lab] acme: Obtaining SAN certificate
2026/11/23 14:15:15 [INFO] [news.cs-202-20.lab] AuthURL: https://ca.lab:14000/authZ/Ru672wayRHS7rer283OppJoyMlPpcFDTfJNmmuLxfMg
2026/11/23 14:15:15 [INFO] [www.cs-202-20.lab] AuthURL: https://ca.lab:14000/authZ/W1PMhF4y_DpBzn28NggkC4EAMNKVPw-ZUgpqgco8jd0
2026/11/23 14:15:15 [INFO] [www.cs-202-20.lab] acme: Could not find solver for: tls-alpn-01
2026/11/23 14:15:15 [INFO] [www.cs-202-20.lab] acme: use http-01 solver
2026/11/23 14:15:15 [INFO] [news.cs-202-20.lab] acme: Could not find solver for: tls-alpn-01
2026/11/23 14:15:15 [INFO] [news.cs-202-20.lab] acme: use http-01 solver
2026/11/23 14:15:15 [INFO] [www.cs-202-20.lab] acme: Trying to solve HTTP-01
2026/11/23 14:15:23 [INFO] [www.cs-202-20.lab] The server validated our request
2026/11/23 14:15:23 [INFO] [news.cs-202-20.lab] acme: Trying to solve HTTP-01
2026/11/23 14:15:28 [INFO] [news.cs-202-20.lab] The server validated our request
2026/11/23 14:15:28 [INFO] [www.cs-202-20.lab, news.cs-202-20.lab] acme: Validations succeeded; requesting certificates
2026/11/23 14:15:28 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/23 14:15:28 [INFO] [www.cs-202-20.lab] Server responded with a certificate.
10.53.0.10 - - [23/Nov/2026:14:15:15 +0000] "GET /.well-known/acme-challenge/bFPzQ7v6Dg3bQrcyJ6Dp0CxYjjdVKDAM_Y0wH7QAmT0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [23/Nov/2026:14:15:15 +0000] "GET /.well-known/acme-challenge/bFPzQ7v6Dg3bQrcyJ6Dp0CxYjjdVKDAM_Y0wH7QAmT0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [23/Nov/2026:14:15:15 +0000] "GET /.well-known/acme-challenge/bFPzQ7v6Dg3bQrcyJ6Dp0CxYjjdVKDAM_Y0wH7QAmT0 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [23/Nov/2026:14:15:23 +0000] "GET /.well-known/acme-challenge/QX5I-eLX4mX3hfsAfm0cmIaq0ipE8ekF6ACvttvBeiI HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [23/Nov/2026:14:15:23 +0000] "GET /.well-known/acme-challenge/QX5I-eLX4mX3hfsAfm0cmIaq0ipE8ekF6ACvttvBeiI HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [23/Nov/2026:14:15:23 +0000] "GET /.well-known/acme-challenge/QX5I-eLX4mX3hfsAfm0cmIaq0ipE8ekF6ACvttvBeiI HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
subject=CN = www.cs-202-20.lab
issuer=CN = Pebble Intermediate CA 4f00e7
notBefore=Nov 23 14:15:28 2026 GMT
notAfter=Nov 23 14:15:27 2031 GMT
serial=7011F9C7D3D00621
X509v3 Subject Alternative Name: 
    DNS:www.cs-202-20.lab, DNS:news.cs-202-20.lab
subject=CN = Pebble Intermediate CA 4f00e7
issuer=CN = Pebble Root CA 7870d8
```

Вивід кроку 3.5:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/23 14:18:29 [notice] 98#98: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.cs-202-20.lab
   i:CN = Pebble Intermediate CA 4f00e7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 23 14:15:28 2026 GMT; NotAfter: Nov 23 14:15:27 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/23 14:19:34 [notice] 107#107: signal process started
server: cs-202-20 (https)
Host: www.cs-202-20.lab
SNI: www.cs-202-20.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.cs-202-20.lab
   i:CN = Pebble Intermediate CA 4f00e7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 23 14:15:28 2026 GMT; NotAfter: Nov 23 14:15:27 2031 GMT
 1 s:CN = Pebble Intermediate CA 4f00e7
   i:CN = Pebble Root CA 7870d8
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 23 13:54:31 2026 GMT; NotAfter: Nov 23 13:54:31 2056 GMT
---
Verify return code: 0 (ok)
server: cs-202-20 (https)
Host: news.cs-202-20.lab
SNI: news.cs-202-20.lab
protocol: TLSv1.3
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | Pebble Intermediate CA 4f00e7 |
| Видавець сертифіката видавця | Pebble Root CA 7870d8 |
| Імена в SAN | www.cs-202-20.lab, news.cs-202-20.lab |
| Строк дії (від – до) | 23.11.2026 14:15:28 GMT – 23.11.2031 14:15:27 GMT (5 років) |
| Тип ключа сервера і алгоритм підпису | ключ id-ecPublicKey, 256 біт (ECDSA); підпис RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 3.5: 1; 3.6: 2 |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Володіння іменами перевірив центр сертифікації Pebble за протоколом ACME методом HTTP-01. `lego` отримав від Pebble випробування для кожного імені й розмістив файл у `/var/www/acme/.well-known/acme-challenge/` на моєму сервері. Pebble (`LetsEncrypt-Pebble-VA`, адреса `10.53.0.10`) сам звернувся до цих імен по HTTP на порт 80 і прочитав файл. Це підтверджують рядки журналу nginx `GET /.well-known/acme-challenge/…` з кодом 200 від `10.53.0.10` (два різні токени: перший о 14:15:15, другий о 14:15:23, що збігається з `Trying to solve HTTP-01` для `www` і `news`), а також рядки `lego` `The server validated our request` для обох імен.
2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
У кроці 3.5 сервер надіслав лише сертифікат сервера, без сертифіката видавця. У `--cacert` вказано тільки корінь Pebble, а сертифікат сервера підписано проміжним CA (`Pebble Intermediate CA 4f00e7`), якого клієнт не має. Клієнт не може побудувати ланцюжок до кореня, тому `curl` повідомляє `unable to get local issuer certificate`, а `openssl s_client` повертає код 21. Корінь правильний і сертифікат виданий правильно: проблема лише в тому, що сервер не надіслав проміжний сертифікат.
3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?
`id-ecPublicKey, 256 (bit)` описує відкритий ключ самого сертифіката сервера: це еліптична крива (ECDSA), 256 біт. `RSA-SHA256` — алгоритм підпису, яким видавець підписав цей сертифікат. Підписував проміжний CA, чий ключ RSA (у виводі 3.6: `rsaEncryption, 2048 (bit)`). Суперечності немає: це ключі різних учасників, один належить серверу, інший — центру сертифікації.
4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Після перезапуску стенда Pebble створює новий корінь і новий проміжний сертифікат (суфікс у назві змінюється; зараз корінь `7870d8`). Сервер надсилав би старий сертифікат, виданий старим проміжним CA, і перевірка з новим коренем завершувалася б помилкою `unable to get local issuer certificate`: новий корінь не знає старого проміжного CA. Сертифікат довелося б видати заново. У моєму виконанні стенд не перезапускався, тому це описано за логікою роботи Pebble (методичка: корінь і проміжний ключ створюються заново під час кожного запуску), а не за спостереженням. У реальній інфраструктурі цьому відповідає ротація або відкликання сертифікатів центру сертифікації (чи втрата довіри до нього): усі видані ним сертифікати треба перевидати.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
=== f1.cs-202-20.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f1.cs-202-20.lab'
Certificate chain
 0 s:CN = old.cs-202-20.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 08:27:38 2026 GMT; NotAfter: Feb 21 08:27:38 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 08:27:37 2026 GMT; NotAfter: Nov 22 08:27:37 2031 GMT
---
Verify return code: 0 (ok)
=== f2.cs-202-20.lab
server: web, f2
Certificate chain
 0 s:CN = f2.cs-202-20.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 08:27:38 2026 GMT; NotAfter: Feb 21 08:27:38 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 08:27:37 2026 GMT; NotAfter: Nov 22 08:27:37 2031 GMT
---
Verify return code: 0 (ok)
=== f3.cs-202-20.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f3.cs-202-20.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 23 08:27:38 2026 GMT; NotAfter: Feb 21 08:27:38 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 23 08:27:37 2026 GMT; NotAfter: Nov 22 08:27:37 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f4.cs-202-20.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.cs-202-20.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 08:27:38 2026 GMT; NotAfter: Feb 21 08:27:38 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f5.cs-202-20.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f5.cs-202-20.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  7 08:27:38 2026 GMT; NotAfter: Nov  3 08:27:38 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 08:27:37 2026 GMT; NotAfter: Nov 22 08:27:37 2031 GMT
---
Verify return code: 10 (certificate has expired)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | `no alternative certificate subject name matches target host name 'f1.cs-202-20.lab'` | 0 (ok) | відповідність імені (SAN) | `0 s:CN = old.cs-202-20.lab` у відповіді на запит до `f1` | випустити сертифікат, у SAN якого є `f1.cs-202-20.lab` |
| f2 | немає (відповідь `server: web, f2`) | 0 (ok) | усі пройдено, сервер справний | `Verify return code: 0 (ok)`, ланцюжок із 2 сертифікатів | нічого |
| f3 | `unable to get local issuer certificate` | 20 (unable to get local issuer certificate) | довіра до видавця: сертифікат виданий центром, якому клієнт не довіряє | `i:O = Foreign Org, CN = Foreign Intermediate CA` (`Foreign Root CA` немає в `$STAND_CA`) | отримати сертифікат у центрі, якому довіряють клієнти |
| f4 | `unable to get local issuer certificate` | 21 (unable to verify the first certificate) | побудова ланцюжка: сервер не надіслав проміжний сертифікат | у ланцюжку лише `0 s:CN = f4.cs-202-20.lab`, елемента `1` немає | додати проміжний сертифікат у файл сертифіката (повний ланцюжок) |
| f5 | `certificate has expired` | 10 (certificate has expired) | строк дії | `NotAfter: Nov  3 08:27:38 2026 GMT` (минув до дати виконання 23.11.2026) | перевидати сертифікат і налаштувати автоматичне поновлення |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?
Таку помилку мають два сервери, і причини різні. **f4** не надіслав проміжний сертифікат: у ланцюжку один сертифікат (елемента `1` немає), видавець — `OPISM Stand Intermediate CA`, той самий проміжний CA, який на справному f2 підписаний `OPISM Stand Root CA` і проходить перевірку; код 21. **f3** має сертифікат, виданий центром, якому клієнт не довіряє: сервер надсилає повний ланцюжок із двох сертифікатів, але видавець — `Foreign Intermediate CA`, підписаний `Foreign Root CA`, якого немає в `$STAND_CA`; код 20. У виводі `openssl s_client` це відрізняється так: у f4 ланцюжок неповний, а видавець належить довіреній ієрархії; у f3 ланцюжок повний, але вся ієрархія чужа. `curl` для обох видає однакове повідомлення, тож розрізнити їх можна лише за ланцюжком і кодом (21 проти 20).
2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.
`openssl s_client` без ключа `-verify_hostname` перевіряє лише ланцюжок підписів і строки, а не відповідність імені, тому для f1 повертає `0 (ok)`: ланцюжок до кореня побудований. `curl` додатково звіряє ім'я сайту з SAN сертифіката: у f1 це `old.cs-202-20.lab`, а запит іде до `f1.cs-202-20.lab`, тому `curl` повідомляє про помилку.
3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?
З `curl -k` відповіли б усі п'ять серверів (перевірено командою, вивід нижче): `-k` вимикає перевірку сертифіката, а рукостискання TLS завершується в усіх. Втрачається автентифікація сервера: клієнт не знає, чи це справжній сервер, а не підмінений (атака посередника). Шифрування лишається, але з невідомим співрозмовником.
root@cs-202-20:/lab/work/lego/certificates# 
```text
for i in 1 2 3 4 5; do curl -sk https://f$i.cs-202-20.lab/ | head -1; done
server: web, f1
server: web, f2
server: web, f3
server: web, f4
server: web, f5
```
## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 RTT | 1 RTT |0 |
| `time_appconnect` | 2 RTT | 3 RTT |0 |
| `time_starttransfer` | 3 RTT | 4 RTT|1 RTT |

Рукостискання TCP займає 1 RTT, тому time_connect = 1 RTT для обох версій TLS. Рукостискання TLS 1.3 займає 1 RTT (клієнт надсилає key_share у ClientHello), тому time_appconnect = 2 RTT. Рукостискання TLS 1.2 потребує двох обмінів, тому time_appconnect = 3 RTT. Після рукостискання запит і перший байт відповіді коштують ще 1 RTT: time_starttransfer = 3 RTT для TLS 1.3 і 4 RTT для TLS 1.2. При повторному використанні з'єднання TCP і TLS уже встановлені, лишається тільки 1 RTT на запит. Значення наведено без пошуку імені: time_namelookup додає до кожного значення нового з'єднання ще приблизно 1 RTT, якщо запит до резолвера проходить через канал із затримкою. Припущення про цей зсув прогноз лише фіксує, перевіряється воно вимірюванням.

Вивід кроків 5.2 і 5.3:

```text
0.070984 0.106162 0.144942 0.180307 0.180368 2 1
0.035554 0.070767 0.109854 0.145234 0.145297 2 1
0.035434 0.070598 0.109472 0.144994 0.145041 2 1
0.035416 0.070564 0.144454 0.179831 0.179898 2 1
0.035567 0.070716 0.144663 0.180263 0.180337 2 1
0.035428 0.070597 0.144399 0.179798 0.179870 2 1
0.035553 0.070756 0.109553 0.145044 0.145188 2 1
0.000022 0.000000 0.000000 0.035360 0.035420 2 0
0.000018 0.000000 0.000000 0.035413 0.035486 2 0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
| TLS 1.3, нове | 35,6 | 70,8 | 109,9 | 145,2 | 145,3 | 1 |
| TLS 1.2, нове | 35,4 | 70,6 | 144,5 | 179,8 | 179,9 | 1 |
| 5.3, перший запит | 35,6 | 70,8 | 109,6 | 145,0 | 145,2 | 1 |
| 5.3, другий запит | 0,0 | 0,0 | 0,0 | 35,4 | 35,4 | 0 |
| 5.3, третій запит | 0,0 | 0,0 | 0,0 | 35,4 | 35,5 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено:
RTT ≈ 35 мс (35,2 мс). Визначено з різниці `time_connect − time_namelookup` (рукостискання TCP — рівно 1 RTT): 70,8 − 35,6 ≈ 35,2 мс. Підтверджують другий і третій запити в 5.3 на повторно використаному з'єднанні: `starttransfer` ≈ 35,4 мс — це один RTT на сам запит.

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?
У RTT (≈ 35,2 мс): TLS 1.3, нове з'єднання — `namelookup` ≈ 1,0, `connect` ≈ 2,0, `appconnect` ≈ 3,1, `starttransfer` ≈ 4,1 RTT. TLS 1.2 — 1,0 / 2,0 / 4,1 / 5,1, бо рукостискання довше на один RTT. Повторне з'єднання (5.3, запити 2–3) — DNS, TCP і TLS відсутні (`num_connects` = 0), лише 1,0 RTT на запит. Відхилення від цілого числа RTT складає ≈ 4 мс і з'являється після рукостискання TLS (`appconnect` 109,9 мс проти 105,6 для TLS 1.3; 144,5 проти 140,8 для TLS 1.2): це час на обчислення (обмін ключами, підпис, перевірка сертифіката). Різниця `starttransfer − appconnect` ≈ 35,4 мс — рівно 1 RTT. Окреме відхилення — перше вимірювання TLS 1.3: `namelookup` 71,0 мс, тобто 2 RTT (див. питання 2 і «Висновки»). Порівняно з прогнозом усі значення для нових з'єднань зсунуті на +1 RTT через пошук імені (TLS 1.3: 2 / 3 / 4 RTT замість 1 / 2 / 3; TLS 1.2: 2 / 4 / 5 замість 1 / 3 / 4); для повторного з'єднання виміряне значення (≈ 1 RTT) збігається з прогнозом.
2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?
`namelookup` ≈ 35,6 мс, бо запит до резолвера `10.53.0.53` виходить через інтерфейс зі штучною затримкою і коштує 1 RTT, навіть якщо відповідь є в кеші резолвера: кеш економить роботу резолвера, а не дорогу до нього. Приблизно вдвічі більшим значення було б, якщо запису в кеші немає: тоді резолвер звертається до авторитетного сервера зони, який працює в моєму контейнері, і його відповідь теж проходить через інтерфейс із затримкою, що додає ще один RTT. Приклад — перший рядок виводу 5.2: 71,0 мс ≈ 2 × 35,5.
3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.
Розрахунок для 200 послідовних запитів (RTT = 35 мс, лише очікування мережі, ім'я вважається вже відомим клієнту):
   - нове з'єднання TLS 1.3 на кожен запит: 3 RTT (TCP + TLS + запит) = 105 мс; 200 × 105 мс = **21 с**;
   - нове з'єднання TLS 1.2: 4 RTT = 140 мс; 200 × 140 мс = **28 с**;
   - одне з'єднання: 3 RTT на встановлення і перший запит, далі 199 × 1 RTT, разом 202 RTT = **≈ 7,1 с**.

Якщо кожне з'єднання ще й виконує власний пошук імені (+1 RTT), TLS 1.3 дасть 28 с, TLS 1.2 — 35 с.

## Висновки

**Що виявилося несподіваним.** У першому з трьох вимірювань TLS 1.3 `time_namelookup` становить 70,98 мс (рядок 1 виводу 5.2: `0.070984 0.106162 0.144942 …`), тобто вдвічі більше, ніж в усіх інших вимірюваннях (≈ 35,5 мс). Різниця `connect − namelookup` у цьому рядку становить 35,2 мс, як в інших, тож зайвий RTT виник саме на пошуку імені, а не на TCP чи TLS.

**Яке питання залишилося без відповіді.** Чому саме перший запит мав подвоєний `namelookup` (рядок 1 виводу 5.2). Припущення: запису про `edge.cs-202-20.lab` не було в кеші резолвера, і він звернувся до авторитетного сервера зони, що працює в моєму контейнері; відповідь звідти виходить через інтерфейс із затримкою ≈ 35 мс, що додає ще один RTT (2 × 35,5 ≈ 71 мс узгоджується зі значенням). Не перевірено: кеш і журнали резолвера не аналізувалися, холодний запит окремо не відтворювався.

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Claude Sonnet 5.5 |Перевір мій звіт з лабораторної роботи. Зверни увагу на правильність відповідей, відповідність висновків отриманим результатам та наявність помилок у тексті. | Використано рекомендації щодо перевірки технічної коректності відповідей і відповідності висновків результатам. Вимірювання та результати команд отримано самостійно.|
| 2 | Claude Sonnet 5.5 |Поясни, чому під час перевірки TLS-сертифіката може виникати помилка unable to get local issuer certificate. | Використано пояснення причини помилки для аналізу результатів перевірки сертифіката. Власні результати команд та висновки перевірено самостійно.|
| 3 | Claude Sonnet 5.5 |Перевір формулювання моїх висновків і виправ граматичні та стилістичні помилки, не змінюючи технічного змісту. | Використано окремі рекомендації щодо формулювань і виправлення мовних помилок. Остаточний текст висновків сформовано та перевірено самостійно.|