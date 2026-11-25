
# Лабораторна робота 2. TLS та сертифікати

| Поле           | Значення                                  |
| -------------- | ----------------------------------------- |
| ПІБ            | Кладніцький Станіслав                     |
| Група          | ЗПІ-21                                    |
| Ідентифікатор  | se-201-01                                 |
| Дата виконання | 25.11.2026                   |
| Середовище     | GitHub Codespaces                         |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)        |

## Параметри варіанта

| Параметр              | Значення                                                      |
| --------------------- | ------------------------------------------------------------- |
| Варіант               | №1                                                            |
| Віддалений сервер     | `www.se-201-01.lab`                                           |
| Імена для сертифіката | `www.se-201-01.lab`, `<!-- ВСТАВИТИ DNS-alias з картки -->`   |
| Ваш сервер (IP)       | 10.53.1.1                                             |

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```
Host: origin.se-201-01.lab
SNI:
protocol: HTTP/1.1
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
20:54:46.317479 IP 10.53.1.1.60258 > 10.53.0.80.80: Flags [P.], seq 1:84, ack 1, win 502, options [nop,nop,TS val 1366219007 ecr 784308209], length 83: HTTP: GET / HTTP/1.1
Qn......GET / HTTP/1.1
Host: origin.se-201-01.lab
server: web
Host: origin.se-201-01.lab
```

Вивід кроку 1.2:

```
Host: origin.se-201-01.lab
SNI: origin.se-201-01.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
```

| Що шукали в дампі                             | HTTP                | HTTPS               |
| --------------------------------------------- | ------------------- | ------------------- |
| рядок запиту `GET /`                          | Знайдено (GET / HTTP/1.1)  | Не видно (0)  |
| заголовок `Host`                              | знайдено (Host: gate.se-201-01.lab) | Не видно (0)  |
| тіло відповіді (`server: web`)                | знайдено (server: web)  | Не видно (0)  |
| ім'я віддаленого сервера (кількість входжень) | 2  | 1  |

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=331
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
    Certificate, Length=989
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

Вивід кроку 2.2 (TLS 1.2):

```
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=226
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=984
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
    NewSessionTicket, Length=230
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

Порівняння версій:

| Показник                                              | TLS 1.3            | TLS 1.2            |
| ----------------------------------------------------- | ------------------ | ------------------ |
| Довжина ClientHello, байт                             | 331 | 226 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | 10 |
| Кількість RTT від ClientHello до першого HTTP-запиту  | `1`                | `2`                |
| Чи видно сертифікат у дампі                           | Ні, зашифрований | Так, відкритий |
| Обраний протокол ALPN                                 | h2 | h2 |

**1. Як визначити початок шифрування в TLS 1.3?**

У TLS 1.3 зашифровані повідомлення рукостискання передаються всередині записів із зовнішнім типом `ApplicationData (23)` (у виводі це рядки `Inner Content Type = Handshake (22)`). Пасивний спостерігач бачить лише тип `ApplicationData` і не може прочитати реальний тип та вміст внутрішніх повідомлень.

**2. Чому ClientHello у TLS 1.3 може бути більшим?**

У ньому міститься більше параметрів та розширень для узгодження криптографічних налаштувань. Зокрема, розширення `key_share` одразу передає ефемерний відкритий ключ клієнта, що дозволяє скоротити рукостискання на 1 RTT.

**3. Що таке SNI?**

SNI — це розширення TLS, яке дозволяє клієнту повідомити серверу потрібне доменне ім'я ще на початковому етапі встановлення з'єднання (у `ClientHello`).

**4. Для чого використовується ChangeCipherSpec?**

У TLS 1.3 цей запис не змінює параметрів шифрування і зберігається переважно для зворотної сумісності зі старими проміжними мережевими пристроями (middleboxes).

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```
=== 3.1 ===
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 2a7c2b
notBefore=Nov 25 21:10:44 2026 GMT
notAfter=Nov 25 21:10:44 2056 GMT

=== 3.2 ===
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```
2026/11/25 21:21:47 [INFO] [app.se-201-01.lab] acme: Trying to solve HTTP-01
2026/11/25 21:21:47 [INFO] [app.se-201-01.lab] The server validated our request
2026/11/25 21:21:53 [INFO] [www.se-201-01.lab, app.se-201-01.lab] acme: Validations succeeded; requesting certificates
2026/11/25 21:21:53 [INFO] [www.se-201-01.lab] Server responded with a certificate.
10.53.0.10 - - [25/Nov/2026:21:21:47 +0000] "GET /.well-known/acme-challenge/iWZyO_N_9IxzD9EfCXkNeZHy58Fa4BHV9X-tktyJFTs HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:21:21:47 +0000] "GET /.well-known/acme-challenge/N3DxAWCxrfNTjlr5Ajrlsgcie--ko93oacPl_81iLQA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"

```

Вивід кроку 3.4:

```
subject=CN = www.se-201-01.lab
issuer=CN = Pebble Intermediate CA 54c6f7
notBefore=Nov 25 21:25:03 2026 GMT
notAfter=Nov 25 21:25:02 2031 GMT
serial=7C1D9D405D2B1908
X509v3 Subject Alternative Name: 
    DNS:www.se-201-01.lab, DNS:app.se-201-01.lab
subject=CN = Pebble Intermediate CA 54c6f7
issuer=CN = Pebble Root CA 725b16
```

Вивід кроку 3.5:

```
curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 54c6f7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 21:25:03 2026 GMT; NotAfter: Nov 25 21:25:02 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```
server: se-201-01 (https)
Host: www.se-201-01.lab
SNI: www.se-201-01.lab
protocol: TLSv1.3
server: se-201-01 (https)
Host: app.se-201-01.lab
SNI: app.se-201-01.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 54c6f7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 21:25:03 2026 GMT; NotAfter: Nov 25 21:25:02 2031 GMT
 1 s:CN = Pebble Intermediate CA 54c6f7
   i:CN = Pebble Root CA 725b16
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 21:10:44 2026 GMT; NotAfter: 25 21:10:44 2056 GMT
---
Verify return code: 0 (ok)  
```

| Параметр                                                               | Значення           |
| ---------------------------------------------------------------------- | ------------------ |
| Видавець сертифіката сервера                                           | Pebble Intermediate CA 54c6f7 |
| Видавець сертифіката видавця                                           | Pebble Root CA 2a7c2b |
| Імена в SAN                                                            | www.se-201-01.lab, app.se-201-01.lab |
| Строк дії (від – до)                                                   | 25.11.2026 21:25:03 – 25.11.2031 21:25:02 GMT |
| Тип ключа сервера і алгоритм підпису                                   | EC (id-ecPublicKey), 256 біт; RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | Крок 3.5 — 1 сертифікат; крок 3.6 — 2 сертифікати |

**1. Як відбувається підтвердження володіння доменом?**

Під час перевірки HTTP-01 клієнт `lego` розміщує унікальний токен у каталозі `/.well-known/acme-challenge/` на сервері, а ACME-центр сам звертається до цього файлу за HTTP. Якщо файл доступний за очікуваною адресою та містить правильне значення, домен вважається підтвердженим.

**2. Чому може виникати помилка перевірки сертифіката?**

Однією з причин є відсутність проміжного сертифіката: сервер надсилає лише свій сертифікат, а клієнт, маючи в довірених тільки корінь, не може побудувати повний ланцюжок довіри (`Verify return code: 21`).

**3. Що означає SAN?**

SAN (Subject Alternative Name) — це поле сертифіката, у якому перелічені доменні імена (або інші ідентифікатори), для яких сертифікат є дійсним.

**4. Для чого потрібен проміжний сертифікат?**

Він з'єднує сертифікат сервера з кореневим центром сертифікації, дозволяючи клієнту побудувати ланцюжок довіри від сервера до довіреного кореня.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```
=== f1.se-201-01.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  8 21:25:18 2026 GMT; NotAfter: Nov  4 21:25:18 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 21:25:17 2026 GMT; NotAfter: Nov 23 21:25:17 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f2.se-201-01.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 24 21:25:18 2026 GMT; NotAfter: Feb 22 21:25:18 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f3.se-201-01.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f3.se-201-01.lab'
Certificate chain
 0 s:CN = staging.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 24 21:25:18 2026 GMT; NotAfter: Feb 22 21:25:18 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 21:25:17 2026 GMT; NotAfter: Nov 23 21:25:17 2031 GMT
---
Verify return code: 0 (ok)
=== f4.se-201-01.lab
server: web, f4
Certificate chain
 0 s:CN = f4.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 24 21:25:18 2026 GMT; NotAfter: Feb 22 21:25:18 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 21:25:17 2026 GMT; NotAfter: Nov 23 21:25:17 2031 GMT
---
Verify return code: 0 (ok)
=== f5.se-201-01.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f5.se-201-01.lab
   i:CN = f5.se-201-01.lab
   v:NotBefore: Nov 24 21:25:18 2026 GMT; NotAfter: Feb 22 21:25:18 2027 GMT
---
Verify return code: 18 (self-signed certificate)
```

Типові помилки та способи їх усунення (заповніть стовпці `f1`–`f5` за своїм виводом):

| Сервер | Помилка `curl`     | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
| ------ | ------------------ | -------------------- | ------------------------ | ---------------------------- | ------------------------------ |
| `f1`   | certificate has expired | 10 (certificate has expired)   | Перевірка строку дії сертифіката — сертифікат прострочений       | NotAfter: Nov 4 21:25:18 2026 GMT           | Оновити сертифікат і встановити новий із чинним строком дії             |
| `f2`   | unable to get local issuer certificate | 21 (unable to verify the first certificate)   | Перевірка ланцюжка довіри — відсутній проміжний сертифікат       | У ланцюжку є лише сертифікат f2, немає OPISM Stand Intermediate CA           | Налаштувати сервер на передавання повного ланцюжка сертифікатів, включно з проміжним             |
| `f3`   | no alternative certificate subject name matches target host name | 0 (ok)   | Перевірка відповідності імені хоста сертифікату (SAN)       | CN = staging.se-201-01.lab, тоді як запитується f3.se-201-01.lab           | Перевидати сертифікат із правильним SAN, що містить f3.se-201-01.lab             |
| `f4`   | Помилки немає | 0 (ok)   | Усі       | Verify return code: 0 (ok)           | Нічого змінювати не потрібно, сертифікат налаштований правильно             |
| `f5`   | self-signed certificate | 18 (self-signed certificate)   | Перевірка довіри до центру сертифікації — сертифікат самопідписаний       | s:CN = f5.se-201-01.lab та i:CN = f5.se-201-01.lab           | Отримати сертифікат від довіреного центру сертифікації та встановити його разом із необхідним ланцюжком             |

**1. Чим відрізняється відсутність проміжного сертифіката від невідомого CA?**

У першому випадку сервер не надсилає один із елементів ланцюжка (у `Certificate chain` лише сертифікат сервера, код `21`), хоча видавець належить до довіреної ієрархії. У другому випадку кореневий центр, який підписав ланцюжок, взагалі не входить до переліку довірених клієнта (коди `18`, `19`, `20`).

**2. Чому OpenSSL та curl можуть показувати різні результати?**

`openssl s_client` без параметра `-verify_hostname` перевіряє лише підписи, довіру до кореня і строк дії, але не відповідність імені вузла. `curl` завжди додатково порівнює запитуваний домен зі значеннями SAN, тому для сертифіката з чужим іменем `openssl` може показати `Verify return code: 0 (ok)`, а `curl` — помилку.

**3. Для чого використовується параметр `-k`?**

Параметр `-k` (`--insecure`) вимикає перевірку сертифіката в `curl`. Він корисний для діагностики, але втрачається автентифікація сервера: з'єднання шифрується, проте клієнт не знає, чи спілкується зі справжнім сервером, а не з атакувальником (MITM). У звичайній роботі його використовувати не слід.

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап                 | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
| -------------------- | ----------------------- | ----------------------- | ------------------------------- |
| `time_connect`       | 2 RTT       | 2 RTT      | 0 RTT              |
| `time_appconnect`    | 3 RTT      | 4 RTT      | 0 RTT              |
| `time_starttransfer` | 4 RTT      | 5 RTT      | 1 RTT              |

Вивід кроків 5.2, 5.3:

```
=== 5.2 TLS 1.3 (3 спроби)
namelookup=0.050931 connect=0.076075 appconnect=0.104834 starttransfer=0.130088 total=0.130243 connects=1
namelookup=0.025421 connect=0.050557 appconnect=0.078688 starttransfer=0.104437 total=0.104487 connects=1
namelookup=0.025421 connect=0.050546 appconnect=0.078684 starttransfer=0.104011 total=0.104069 connects=1
=== 5.2 TLS 1.2 (3 спроби)
namelookup=0.025488 connect=0.050620 appconnect=0.104137 starttransfer=0.129431 total=0.129493 connects=1
namelookup=0.025410 connect=0.050576 appconnect=0.105081 starttransfer=0.130466 total=0.130530 connects=1
namelookup=0.025424 connect=0.050558 appconnect=0.105159 starttransfer=0.130471 total=0.130529 connects=1
=== 5.3 Повторне використання з'єднання (3 запити)
namelookup=0.025436 connect=0.050588 appconnect=0.078881 starttransfer=0.104171 total=0.104230 connects=1
```

Таблиця результатів (мс):

| Запит             | `namelookup` | `connect`          | `appconnect`       | `starttransfer`    | `total`            | `num_connects`     |
| ----------------- | ------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ |
| TLS 1.3, нове     | 33.924     | 59.059	           | 87.402           | 112.845           | 112.933	           | 1           |
| TLS 1.2, нове     | 25.441     | 50.585           | 104.792           | 130.123           | 130.184	           | 1          |
| Повторний запит 1 | 25.436     | 50.588           | 78.881           | 104.171           | 104.230           | 1           |
| Повторний запит 2 | -----     | -----           | -----           | -----           | -----           | -----           |
| Повторний запит 3 | -----     | -----           | -----           | -----           | -----           | -----           |

RTT, визначений вимірюванням, і з якого значення його визначено: <!-- ВСТАВИТИ, напр. «1 RTT ≈ … мс — з starttransfer повторного запиту» -->

**1. Чому виміряний час може відрізнятися від теоретичного?**

На тривалість з'єднання впливають навантаження системи, швидкість обробки пакетів, мережеві затримки та витрати процесора на криптографічні операції (обмін ключами, перевірка підпису сертифіката), через що значення зазвичай не є точно цілим числом RTT.

**2. Чому `time_namelookup` не завжди дорівнює нулю?**

DNS-запит потребує часу на виконання та отримання відповіді. Кожен запуск `curl` — це новий процес зі своїм кешем, тому навіть якщо резолвер має запис у кеші, відповідь надходить через мережу і займає щонайменше 1 RTT.

**3. Чому повторне використання з'єднання швидше?**

Під час повторного запиту в межах того самого з'єднання не потрібно заново виконувати DNS-запит, TCP-рукостискання та TLS-рукостискання, тому лишається приблизно 1 RTT на сам HTTP-запит і відповідь.

## Висновки

Під час виконання лабораторної роботи було досліджено принципи роботи протоколу TLS та його роль у захисті мережевого обміну даними.

Було порівняно HTTP і HTTPS, розглянуто відмінності між TLS 1.2 та TLS 1.3, а також досліджено механізм SNI та узгодження протоколів через ALPN.

Окремо було розглянуто процес отримання сертифіката за допомогою ACME, його встановлення на сервер та перевірку повного ланцюжка довіри. Також було досліджено типові помилки сертифікатів і способи їх усунення.

На завершення розглянуто вплив TLS-рукостискання на час встановлення HTTPS-з'єднання та переваги повторного використання вже створеного з'єднання.

## Декларація використання ШІ

Використовувалися інструменти ШІ: Так

| № | Інструмент | Промпт (дослівно)            | Що використано у звіті і що змінено самостійно |
| - | ---------- | ---------------------------- | ---------------------------------------------- |
| 1 | ChatGPT    | Допоможи розібратись в роботі | Перефразування пояснень, структура документа, редагування тексту |
| 2 | Claude     | напиши відповіді на питання | Оформлення звіту за структурою зразка          |

Усі фактичні результати виконання команд, вимірювання та остаточні висновки перевірені автором.
