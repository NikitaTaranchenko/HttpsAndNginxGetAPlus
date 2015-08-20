# NGINX и HTTPS - получаем класс А+

> Источник - Статья с Хабрахабр http://habrahabr.ru/post/252821/


Итоговый конфигурационный файл NGINX:
```bash``` 
server {
    server_name dsmirnov.pro www.dsmirnov.pro;
    listen 80;
    return 301 https://dsmirnov.pro$request_uri;
}

server {
    listen 443 ssl spdy;
    server_name dsmirnov.pro;
    resolver 127.0.0.1;
    ssl_stapling on;
    ssl on;
    ssl_certificate /etc/pki/nginx/dsmirnov.pro.pem;
    ssl_certificate_key /etc/pki/nginx/dsmirnov.pro.clean.key;
    ssl_dhparam /etc/pki/nginx/dhparam.pem;
    ssl_session_timeout 24h;
    ssl_session_cache shared:SSL:2m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers kEECDH+AES128:kEECDH:kEDH:-3DES:kRSA+AES128:kEDH+3DES:DES-CBC3-SHA:!RC4:!aNULL:!eNULL:!MD5:!EXPORT:!LOW:!SEED:!CAMELLIA:!IDEA:!PSK:!SRP:!SSLv2;
    ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security "max-age=31536000;";
    add_header Content-Security-Policy-Report-Only "default-src https:; script-src https: 'unsafe-eval' 'unsafe-inline'; style-src https: 'unsafe-inline'; img-src https: data:; font-src https: data:; report-uri /csp-report";
}
```

Следущим шагом включаем ssl_stapling on; — позволяем серверу прикреплять OCSP-ответы, тем самым уменьшая время загрузки страниц у пользователей. Цепочка сертификатов (доменный — промежуточный центр авторизации — корневой центр авторизации) может содержать 3-4 уровня. И на каждый уровень браузер должен устанавливать соединение и получать сертификат. Можно отправить все сертификаты (включая промежуточный: именно за этим была настройка TCP-окон отправки, чтобы цепочка сертификатов гарантированно поместилась в в одну пересылку пакетов) разом, тогда браузер проверит всю цепочку локально, а запросит только корневой (который в большинстве случаев уже находится на клиенте). Для работы этой функции обязательно описать resolver — у меня поднят свой собственный DNS сервер, поэтому в качестве значения указан 127.0.0.1, Вы можете указать 8.8.8.8, но многие в последнее время на него ругаются. Что такое ssl on; я думаю нет смысла рассказывать.

Далее директивами ssl_certificate и ssl_certificate_key указываем пути к полученным через StartSsl сертификатам. У вас уже есть 3 файла: domain.ru.key, domain.ru.crt и sub.class1.server.ca.pem. Копируем ключи в ( моём случае ) /etc/pki/nginx.

Не забываем, что pem файл для nginx должен быть смержен с сертификатом CA ( Должно получиться из 3х — 2 файла. ):

```bash```
cp domain.ru.key /etc/pki/nginx
cat domain.ru.crt sub.class1.server.ca.pem > /etc/pki/nginx/domain.ru.pem
```

Теперь о ssl_dhparam /etc/pki/nginx/dhparam.pem; — это нужно, чтобы у нас заработал Forward Secrecy. Прямая секретность означает, что если третья сторона узнает какой-либо сеансовый ключ, то она сможет получить лишь доступ к данным, защищенным лишь этим ключом. Для сохранения совершенной прямой секретности ключ, используемый для шифрования передаваемых данных, не должен использоваться для получения каких-либо дополнительных ключей. Также, если ключ, используемый для шифрования передаваемых данных, был получен (derived) на базе какого-то еще ключевого материала, этот материал не должен использоваться для получения каких-либо других ключей.
Сгенерировать ключ можно так:

```bash```
openssl dhparam -out /etc/pki/nginx/dhparam.pem 4096
```
Далее несложные настройки ssl_session_timeout 24h; и ssl_session_cache shared:SSL:2m;, которые не требуют особенных описаний — срок истечения сессии и размер памяти, выделяемой для хранения кеша — у меня бложик маленький, поэтому 2 Мб вполне достаточно.

Дальше — важные параметы: ssl_protocols TLSv1 TLSv1.1 TLSv1.2; и ssl_ciphers kEECDH+AES128:kEECDH:kEDH:-3DES:kRSA+AES128:kEDH+3DES:DES-CBC3-SHA:!RC4:!aNULL:!eNULL:!MD5:!EXPORT:!LOW:!SEED:!CAMELLIA:!IDEA:!PSK:!SRP:!SSLv2; — тут мы указываем, что мы желаем только TLS и второй строкой выжигаем калёным железом все SSL. В свете последних фейлов с SSL — очень актуально, что и Вам советую. Ну и директивой ssl_prefer_server_ciphers on; принуждаем nginx это всё строго соблюдать.

Директива add_header Strict-Transport-Security «max-age=31536000;»; указывает браузерам сколько они должны помнить данные требования безопасности для моего домена. В данном случае — 1 год. Кстати, если директиву написать вот так: add_header Strict-Transport-Security «max-age=31536000; includeSubDomains; preload»;, то данные условия будут применимы ко всем доменам третьего и выше уровня вашего домена. Тут будте осторожны! Я изначально описал именно так, но так, как StartSsl выдаёт сертификаты на ограниченное количество поддоменов, я наткнулся на невозможность даже попасть на свои поддомены, которые обслуживают разнообразные админки, работали только те, на которые были выписаны trusted сертификаты. Поэтому я для себя выбрал первый вариант.

Далее — add_header Content-Security-Policy-Report-Only «default-src https:; script-src https: 'unsafe-eval' 'unsafe-inline'; style-src https: 'unsafe-inline'; img-src https: data:; font-src https: data:; report-uri /csp-report»; — я толком глубоко ещё не изучил свойства данного заголовка. Content Security Policy (CSP) — новый стандарт, определяющий HTTP-заголовки Content-Security-Policy и Content-Security-Policy-Report-Only, которые сообщают браузеру белый список хостов, с которых он может загружать различные ресурсы. 

[SPDY Check](https://spdycheck.org)

[SSL labs - проверка качества защиты сервера](https://www.ssllabs.com/ssltest/index.html)

