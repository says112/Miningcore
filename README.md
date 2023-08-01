[![Build status](https://ci.appveyor.com/api/projects/status/nbvaa55gu3icd1q8?svg=true)](https://ci.appveyor.com/project/oliverw/miningcore)
[![.NET](https://github.com/oliverw/miningcore/actions/workflows/dotnet.yml/badge.svg)](https://github.com/oliverw/miningcore/actions/workflows/dotnet.yml)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg)]()

<img src="https://github.com/oliverw/miningcore/raw/master/logo.png" width="150">

### Features

- Supports clusters of pools each running individual currencies
- Ultra-low-latency, multi-threaded Stratum implementation using asynchronous I/O
- Adaptive share difficulty ("vardiff")
- PoW validation (hashing) using native code for maximum performance
- Session management for purging DDoS/flood initiated zombie workers
- Payment processing
- Banning System
- Live Stats [API](https://github.com/oliverw/miningcore/wiki/API) on Port 4000
- WebSocket streaming of notable events like Blocks found, Blocks unlocked, Payments and more
- POW (proof-of-work) & POS (proof-of-stake) support
- Detailed per-pool logging to console & filesystem
- Runs on Linux and Windows

Скачиваем последнюю версию UFO core:


wget https://github.com/fiscalobject/ufo/releases/download/v0.18.1/ufo-0.18.1-linux64.tar.gz
tar -xvf ufo-0.18.1-linux64.tar.gz
Меняем стандартный конфиг UFO core:


mkdir ~/.ufo
nano ~/.ufo/ufocoin.conf

zmqpubhashblock=tcp://127.0.0.1:15101
# юзернейм
rpcuser=ufocore
# пароль
rpcpassword=ufocore
deprecatedrpc=accounts
addresstype=legacy
changetype=legacy
Сохраняем изменения (ctrl+X, Y, Enter) и синхронизируемся с блокчейном:


./ufod
# или запускаем в screen, если используется серверный дистрибутив
screen -dmS ufo ./ufod
# в логе ноды нужно дождаться progress=1.000000
# посмотреть лог: screen -r ufo
# выйти из скрина: последовательно нажать ctrl+A, ctrl+D
Получаем пару новых адресов:


./ufo-cli getnewaddress
./ufo-cli getnewaddress
# адреса будут Legacy формата B или C, с ними работает пул, а вот в майнере можно указывать любой
Переходим к пулу. Устанавливаем зависимости:


wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb

sudo apt install ./packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt update

sudo apt install postgresql dotnet-sdk-6.0 git cmake build-essential libssl-dev pkg-config libboost-all-dev libsodium-dev libzmq5 libzmq3-dev nginx gcc g++ make
Клонируем репозиторий пула:

git clone https://github.com/oliverw/miningcore
Настраиваем PostgreSQL:


sudo -u postgres psql

# пароль от базы данных указывается после PASSWORD внутри одинарных кавычек
CREATE ROLE miningcore WITH LOGIN ENCRYPTED PASSWORD 'miningcore';
CREATE DATABASE miningcore OWNER miningcore;
\q

sudo -u postgres psql -d miningcore -f miningcore/src/Miningcore/Persistence/Postgres/Scripts/createdb.sql

# если вы получили ошибку вида:
# could not change directory to "/home/user": Permission denied
# psql: error: miningcore/src/Miningcore/Persistence/Postgres/Scripts/createdb.sql:
# No such file or directory
# то необходимо добавить юзера postgres в группу вашего пользователя
# и повторить команду
sudo usermod -a -G $USER postgres
Собираем пул:


cd ~/miningcore/src/Miningcore
dotnet publish -c Release --framework net6.0 -o ../../build
После завершения сборки необходимо изменить два конфигурационных файла в директории ~/miningcore/build


cd ~/miningcore/build
nano coins.json
Добавляем UFO по аналогии с Feathercoin с небольшим отличием в blockHasher:


   "ufocoin": {
        "name": "UFOcoin",
        "symbol": "UFO",
        "family": "bitcoin",
        "website": "",
        "market": "",
        "twitter": "",
        "telegram": "",
        "discord": "",
        "coinbaseHasher": {
            "hash": "sha256d"
        },
        "headerHasher": {
            "hash": "neoscrypt",
            "args": [ 0 ]
        },
        "blockHasher": {
              "hash": "reverse",
              "args": [ { "hash": "sha256d" } ]
        },
        "shareMultiplier": 65536,
        "explorerBlockLink": "https://chainz.cryptoid.info/ufo/block.dws?$height$.htm",
        "explorerTxLink": "https://chainz.cryptoid.info/ufo/tx.dws?{0}.htm",
        "explorerAccountLink": "https://chainz.cryptoid.info/ufo/address.dws?{0}.htm"
    },
Сохраняем и создаем конфиг пула. Подробнее о файле конфигурации можно узнать здесь https://github.com/oliverw/miningcore/wiki/Configuration


{
    "logging": {
        "level": "info",
        "enableConsoleLog": true,
        "enableConsoleColors": true,
        "logFile": "core.log",
        "apiLogFile": "api.log",
        "logBaseDirectory": "./logs",
        "perPoolLogFile": false
    },
    "banning": {
        "manager": "Integrated",
        "banOnJunkReceive": false,
        "banOnInvalidShares": false
    },
    "notifications": {
        "enabled": false,
        "email": {
            "host": "smtp.example.com",
            "port": 587,
            "user": "user",
            "password": "password",
            "fromAddress": "info@yourpool.org",
            "fromName": "pool support"
        },
        "admin": {
            "enabled": false,
            "emailAddress": "user@example.com",
            "notifyBlockFound": true
        }
    },
    "persistence": {
        "postgres": {
            "host": "127.0.0.1",
            "port": 5432,
            "user": "miningcore",
            "password": "miningcore", // пароль от БД, заданный в psql
            "database": "miningcore"
        }
    },
    // Generate payouts for recorded shares and blocks
    "paymentProcessing": {
        // ставим значение false, если майните в одиночку и не нужны выплаты
        // на другой кошелек
        "enabled": true,
        "interval": 600,
        "shareRecoveryFile": "recovered-shares.txt"
    },
    "api": {
        "enabled": true,
        "listenAddress": "0.0.0.0",
        "port": 4000,
       "metricsIpWhitelist": [],
        "rateLimiting": {
            "disabled": true,
            "rules": [
                {
                    "Endpoint": "*",
                    "Period": "1s",
                    "Limit": 20
                }
            ],
            "ipWhitelist": []
        }
    },
    "pools": [
        {
            "id": "ufo",
            "enabled": true,
            "coin": "ufocoin",
            // здесь указывается один из адресов, полученных с помощью ufo-cli
            "address": "B.................",
            "rewardRecipients": [
                {
                    // здесь указывается второй адрес, полученный с помощью ufo-cli
                    "address": "B.................",
                    // комиссия пула в %, обязательно указывать отличную от нуля,
                    // если осуществляются выплаты
                    "percentage": 0.5
                }
            ],
            "blockRefreshInterval": 0,
            "jobRebroadcastTimeout": 10,
            "clientConnectionTimeout": 600,
            "banning": {
                "enabled": false,
                "time": 600,
                "invalidPercent": 50,
                "checkThreshold": 50
            },
            "ports": {
                // Порт, на который будет коннектиться майнер
                "8888": {
                    "listenAddress": "0.0.0.0",
                    // Сложность подбирается индивидуально, в зависимости от
                    // хэшрейта оборудования
                    "difficulty": 10.0,
                    "tls": false,
                    "tlsPfxFile": "/var/lib/certs/mycert.pfx",
                    "varDiff": {
                        "minDiff": 10.0,
                        "maxDiff": null,
                        "targetTime": 15,
                        "retargetTime": 90,
                        "variancePercent": 30,
                        "maxDelta": 500
                    }
                }
            },
            "daemons": [
                {
                    "host": "127.0.0.1",
                    // rpc-порт ноды, узнать его можно так: ./ufod -h | grep rpcport
                    "port": 9888,
                    // юзернейм из ufocoin.conf
                    "user": "ufocore",
                    // пароль из ufocoin.conf
                    "password": "ufocore",
                    "zmqBlockNotifySocket": "tcp://127.0.0.1:15101",
                }
            ],
            "paymentProcessing": {
            // ставим значение false, если майните в одиночку и не нужны выплаты
            // на другой кошелек
                "enabled": true,
                "minimumPayment": 0.1,
                "payoutScheme": "PPLNS",
                "payoutSchemeConfig": {
                    "factor": 0.5
                }
            }
        },
    ]
}
Сохраняем и запускаем пул:

./Miningcore -c config.json

Прикручиваем веб-интерфейс:


cd
git clone https://github.com/minernl/Miningcore.WebUI webui/
nano webui/js/miningcore.js
# в открывшемся файле преобразуем 33 строку следующим образом
var API            = "http://192.168.XX.XX:4000/" + "api/";
# где 192.168.XX.XX - адрес вашей машины в локальной сети
# узнать его можно, например, с помощью команды ifconfig

# для красоты можно закинуть png-иконку UFO по пути ~/webui/img/coin/icon/ufo.png
Настраиваем веб-сервер nginx:


sudo nano /etc/nginx/sites-available/default
# удаляем всё и прописываем следующую конфигурацию

server {
        listen 0.0.0.0:4001;
        root /home/LINUX_ЮЗЕРНЕЙМ/webui;
        index index.html;
}

# сохраняем и перезапускаем nginx
sudo systemctl restart nginx.service

# открываем порты API, веб-сервера и stratum
sudo ufw allow 4000
sudo ufw allow 4001
sudo ufw allow 8888
Заходим в браузере на http://192.168.XX.XX:4001


Если в браузере получили ошибку "403 Forbidden", то необходимо добавить юзера www-data в группу вашего пользователя, перезапустить nginx, и обновить страницу в браузере через ctrl+F5:


sudo usermod -a -G $USER www-data
sudo systemctl restart nginx.service
Собственно всё, можно цепляться майнером на порт 8888 с любой машины в локальной сети.
