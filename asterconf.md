# Шаг 0
Свежеустановленный контейнер с Debian 9, сконфигурирован ssh, консоль и таймзона - Debian 9

# Шаг 1
Обновление и немного пакетов:
```bash
apt update && apt -y dist-upgrade
apt -y install mc wget net-tools \
tcpdump curl socat sngrep pwgen \
nginx php-fpm mysql-server php-mysql \
liblua5.1-dev
```

Установка acme.sh и получение сертификата:

```bash
wget -O - https://get.acme.sh | sh
```

Перелогиниться (acme.sh создает alias)

```bash
acme.sh --issue -d asterconf2019.lanta.me -w /var/www/html/
acme.sh --install-cert -d asterconf2019.lanta.me \
--key-file /etc/ssl/key.pem \
--fullchain-file /etc/ssl/cert.pem \
--reloadcmd "service nginx force-reload"
```

# Шаг 2
Скачивание и установка Asterisk (v16):

```bash
wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz \
-q -O - | gzip -dc | tar -xvf -
```

Зависимости, конфигурация (выбрать opus):

```bash
./contrib/scripts/install_prereq install
./configure --with-jansson-bundled
make menuconfig
```

Компиляция, установка:

```bash
make -j5 && make install
```

# Шаг 3
конфиг NGINX (/etc/init.d/nginx/sites-enabled/default):

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;

        ssl_certificate /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;

        root /var/www/html;

        index index.php index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
        }

        location /ws {
                proxy_pass http://127.0.0.1:8088/ws;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 43200000;
        }
}
```

```bash
/etc/init.d/nginx restart
```

# Шаг 4
Минимальная конфигурация Asterisk:

```bash
mkdir /etc/asterisk
touch /etc/asterisk/asterisk.conf
touch /etc/asterisk/extensions.lua
touch /etc/asterisk/http.conf
touch /etc/asterisk/modules.conf
touch /etc/asterisk/musiconhold.conf
touch /etc/asterisk/pjsip.conf
```

/etc/asterisk/asterisk.conf:

```ini
[options]
transmit_silence_during_record = yes
defaultlanguage = ru
```

/etc/asterisk/extensions.lua:

```lua
extensions = {
    ["default"] = {
        ["_."] = function ()
            app.Answer()
            app.MusicOnHold()
        end,
        ["1001"] = function ()
            app.Dial("PJSIP/1001", 60, 'mtT')
        end,
    },
}
```

/etc/asterisk/http.conf:

```ini
[general]
enabled = yes
bindaddr = 127.0.0.1
bindport = 8088
tlsenable = no
```

/etc/asterisk/modules.conf:

```ini
[global]

[modules]
autoload = yes
noload = chan_sip.so
```

/etc/asterisk/musiconhold.conf:

```ini
[default]
mode = files
directory = /var/lib/asterisk/moh
sort = random
```

/etc/asterisk/pjsip.conf:

```ini
; Общие настройки

[transport-udp]
type = transport
protocol = udp
bind = 0.0.0.0

; Шаблоны

[webrtc-aor-template](!)
type = aor
max_contacts = 1
remove_existing = yes

[webrtc-endpoint-template](!)
type = endpoint
context = default
disallow = all
allow = opus
webrtc = yes

; Телефоны

[1001](webrtc-aor-template)

[1001]
type = auth
username = 1001
password = Giethoh4

[1001](webrtc-endpoint-template)
auth = 1001
outbound_auth = 1001
aors = 1001
callerid = "М. Иванов" <1001>
```

# Шаг 5
Запускаем Asterisk, переходим к тестам

```bash
asterisk -fvvv
```

# Шаг 6

[https://tryit.jssip.net](https://tryit.jssip.net)

.call файл для проверки входящего звонка

```
Channel: Local/1001
Callerid: 89051200829
MaxRetries: 1
RetryTime: 600
WaitTime: 45
Context: default
Extension: 89051200829
Archive: No
```


