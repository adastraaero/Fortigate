
## Настройка Threat Feeds

Раздел Security Fabric  - External Connectors - Threat Feeds  позволяет подключать внешние источники данных, такие как:
* обновляемые txt файлы со списками вредоносных ip адресов расположенные на внешних веб-серверах (работает даже если лицензии истекли)
* обновляемые списки хэшей файлов вредоносного ПО расположенные на внешних веб-серверах 


![https-process](img/thf1.png)



### Где брать списки 



### Как подключать IP address  обновляемые txt файлы и хеши

В интернете есть большое количество компаний по ИБ, которые предоставляют платные и бесплатные доступы к таким обновляемым спискам:

* https://urlhaus.abuse.ch/
    Здесь нас интересует раздел https://urlhaus.abuse.ch/api/ и ссылки Plain-Text

* https://www.dan.me.uk/torlist/

* https://phishing.army

* https://bazaar.abuse.ch/export/  - malware hashes - здесь нам нужны  SHA256 hashes (https://bazaar.abuse.ch/export/txt/sha256/recent/)



Создаём Threat Feed

![FGT](img/thf2.png)

![FGT](img/thf4.png)

Рекомендую выставлять поле Refresh Rate на 30 и более минут, часто сайты которые предоставляют такие списки, ограничивают частоту доступа к таким спискам.

Обновляем Threat Feed

![FGT](img/thf3.png)


Далее создаём входящие и исходящие правила с блокировкой (DENY) доступа к данным спискам, данные правила должны располагаться выше разрешающих правил.

Для включения Malware hashes необходимо зайти в Security Profiles - Antivirus

выбрать необходимый профиль и включить "User External Malware Block List"

![FGT](img/thf5.png)

после этого зайти в консоль FTG и проверить, что хэши подгрузились, команды для проверки в зависимости от версии прошивки FGT:

```
diagnose sys scanunit file-hash list 
diagnose sys scanunit malware-list list
```


![FGT](img/thf6.png)


### Минимальная настройка nginx для создания и подключения статического файла в Threat Feeds

Настройка nginx


Создаём директорию для сайта
```
sudo mkdir -p /var/www/your_domain/html
sudo chown -R $USER:$USER /var/www/your_domain/html
sudo chmod -R 755 /var/www
```

Проверяем, что работает, создаём в /var/www/your_domain/html файл index.html

```
echo "<html>
  <head>
    <title>TEST</title>
  </head>
  <body>
    <h1>TEST Working !</h1>
  </body>
</html>
```

Создаём файл конфигурации  для сайта 

vim /etc/nginx/sites-available/fgtreport

где /var/www/your_domain/html/lists - это папка в которой лежат файлы со списком ip адресов для блока

```
server {
    listen 80;
    # download
    autoindex on;               # enable directory listing output
    autoindex_exact_size off;   # output file sizes rounded to kilobytes, megabytes, and gigabytes
    autoindex_localtime on;     # output local times in the directory


    location / {
        root /var/www/your_domain/html/lists;
    }
}
```

Линкуем available к enabled

```
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

Удаляем дефолтный сайт

```
sudo unlink /etc/nginx/sites-enabled/default
```

Проверяем конфиг nginx

```
sudo nginx -t
```

Перезапускаем nginx

```
sudo systemctl restart nginx
```



























## Настройка безопасности SSL VPN


Посмотреть текущие настройки ssl vpn:

```
show vpn ssl settings
```


Создаём сетевые объекты по ГЕОпризнаку.

![FGT](img/FGT_VPN_1.png)


Ограничиваем доступ к SSL VPN подключениям по определенным странам.


![FGT](img/FGT_VPN_2.png)


Ограничиваем количество одновременных подключений 

через графический интерфейс:

![FGT](img/FGT_VPN_3.png)

через консоль:

```
config vpn ssl web portal
    edit "full-access"
        set limit-user-logins enable
end

```

Настраиваем блокировку пользователя на 15 минут, после 3ех неудачных попыток ввода пароля

```
config vpn ssl settings
    set login-attempt-limit 3
    set login-block-time 900
end
```


По умолчанию все пользователи, которые пытаются подключиться(даже если их не существует), попадают на дефолтный портал,
это можно увидеть в VPN-SSL VPN Settings в разделе Authentication/Portal Mapping.

Для минимизации площади атак создаём портал без доступа и ставим его по умолчанию, то есть все пользователи вне назначенных для них
порталов будут попадат на данный портал.


Создаём портал:

```
config vpn ssl web portal
    edit FakePortal
        set tunnel-mode disable
        set web-mode disable
        set ipv6-tunnel-mode disable
    next
end
```

Настраиваем его порталом по умолчанию:

```
config vpn ssl setting
        set default-portal FakePortal
end
```

Включаем более высокую версию протокола шифрования

```
config vpn ssl settings
        set ssl-min-proto-ver tls1-2
end
```


Лучше включать 1.3, но у Forticlient с этим есть проблемы на части ОС.



## Сбор логов и настройки

<details>


 <summary>Настройка логирования</summary>

В политике, из который хотим получать логи, должно быть включено логирование:

```
config firewall policy
    edit <Policy_id>
        set logtraffic all/utm
end
```

show log setting - смотрим какие настройки логирования включены.

базово должны быть включены:

```
set local-in-allow enable
set local-in-deny-unicast enable
set local-in-deny-broadcast enable
set local-out enable
```

Полный набор настроек логирования:

```
resolve-ip                Add resolved domain name into traffic log if possible.
resolve-port              Add resolved service name into traffic log if possible.
log-user-in-upper         Enable/disable collect log with user-in-upper.
fwpolicy-implicit-log     Enable/disable collect firewall implicit policy log.
fwpolicy6-implicit-log    Enable/disable collect firewall implicit policy6 log.
log-invalid-packet        Enable/disable collect invalid packet traffic log.
local-in-allow            Enable/disable collect local-in-allow log.
local-in-deny-unicast     Enable/disable collect local-in-deny-unicast log.
local-in-deny-broadcast   Enable/disable collect local-in-deny-broadcast log.
local-out                 Enable/disable collect local-out log.
daemon-log                Enable/disable collect daemon log.
neighbor-event            Enable/disable collect neighbor event log.
brief-traffic-format      Enable/disable use of brief format for traffic log.
user-anonymize            Enable/disable anonymize log user name.
expolicy-implicit-log     Enable/disable collect explicit proxy firewall implicit policy log.
log-policy-comment        Enable/disable insertion of policy comment in to traffic log.
```


Настройка того где хранить логи:
```
config log memory/disk/fortianalyzer/syslog setting
set status enable
end
```

За логирование на Fortigate отвечает процесс miglog (может быть несколько).
Бывает так, что процессы есть, но новые логи с какого момента не появляются, для этого перезапускаем процесс
логирования 

```
diag sys top 2 50
diag sys kill 11 <PID> 

```






</details>








## Работа с Active Directory

```

Коды ошибок LDAP и их описание

Code                              Value  Description
---------------------------------------------------------------------------
LDAP_SUCCESS                      0x00   Successful request.
LDAP_OPERATIONS_ERROR             0x01   Initialization of LDAP library failed.

LDAP_PROTOCOL_ERROR               0x02   Protocol error occurred.
LDAP_TIMELIMIT_EXCEEDED           0x03   Time limit has exceeded.
LDAP_SIZELIMIT_EXCEEDED           0x04   Size limit has exceeded.
LDAP_COMPARE_FALSE                0x05   Compare yielded FALSE.
LDAP_COMPARE_TRUE                 0x06   Compare yielded TRUE.
LDAP_AUTH_METHOD_NOT_SUPPORTED    0x07   The authentication method is not supported.

LDAP_STRONG_AUTH_REQUIRED         0x08   Strong authentication is required.
LDAP_REFERRAL_V2                  0x09   LDAP version 2 referral.
LDAP_PARTIAL_RESULTS              0x09   Partial results and referrals received.

LDAP_REFERRAL                     0x0a   Referral occurred.
LDAP_ADMIN_LIMIT_EXCEEDED         0x0b   Administration limit on the server has exceeded.

LDAP_UNAVAILABLE_CRIT_EXTENSION   0x0c   Critical extension is unavailable.
LDAP_CONFIDENTIALITY_REQUIRED     0x0d   Confidentiality is required.
LDAP_NO_SUCH_ATTRIBUTE            0x10   Requested attribute does not exist.

LDAP_UNDEFINED_TYPE               0x11   The type is not defined.  
LDAP_INAPPROPRIATE_MATCHING       0x12   An inappropriate matching occurred. 

LDAP_CONSTRAINT_VIOLATION         0x13   A constraint violation occurred.
LDAP_ATTRIBUTE_OR_VALUE_EXISTS    0x14   The attribute exists or the value has been assigned.

LDAP_INVALID_SYNTAX               0x15   The syntax is invalid.
LDAP_NO_SUCH_OBJECT               0x20   Object does not exist.
LDAP_ALIAS_PROBLEM                0x21   The alias is invalid.
LDAP_INVALID_DN_SYNTAX            0x22   The distinguished name has an invalid syntax.

LDAP_IS_LEAF                      0x23   The object is a leaf.
LDAP_ALIAS_DEREF_PROBLEM          0x24   Cannot de-reference the alias.
LDAP_INAPPROPRIATE_AUTH           0x30   Authentication is inappropriate.
LDAP_INVALID_CREDENTIALS          0x31   The supplied credential is invalid.

LDAP_INSUFFICIENT_RIGHTS          0x32   The user has insufficient access rights.

LDAP_BUSY                         0x33   The server is busy.
LDAP_UNAVAILABLE                  0x34   The server is unavailable.
LDAP_UNWILLING_TO_PERFORM         0x35   The server does not handle directory requests.

LDAP_LOOP_DETECT                  0x36   The chain of referrals has looped back to a referring server.

LDAP_NAMING_VIOLATION             0x40   There was a naming violation.
LDAP_OBJECT_CLASS_VIOLATION       0x41   There was an object class violation.

LDAP_NOT_ALLOWED_ON_NONLEAF       0x42   Operation is not allowed on a non-leaf object.

LDAP_NOT_ALLOWED_ON_RDN           0x43   Operation is not allowed on RDN.
LDAP_ALREADY_EXISTS               0x44   The object already exists.
LDAP_NO_OBJECT_CLASS_MODS         0x45   Cannot modify object class.
LDAP_RESULTS_TOO_LARGE            0x46   Results returned are too large.
LDAP_AFFECTS_MULTIPLE_DSAS        0x47   Multiple directory service agents are affected.

LDAP_OTHER                        0x50   Unknown error occurred.
LDAP_SERVER_DOWN                  0x51   Cannot contact the LDAP server.
LDAP_LOCAL_ERROR                  0x52   Local error occurred.
LDAP_ENCODING_ERROR               0x53   Encoding error occurred.
LDAP_DECODING_ERROR               0x54   Decoding error occurred.
LDAP_TIMEOUT                      0x55   The search was timed out.
LDAP_AUTH_UNKNOWN                 0x56   Unknown authentication error occurred.

LDAP_FILTER_ERROR                 0x57   The search filter is incorrect.
LDAP_USER_CANCELLED               0x58   The user has canceled the operation.

LDAP_PARAM_ERROR                  0x59   An incorrect parameter was passed to a routine.

LDAP_NO_MEMORY                    0x5a   The system is out of memory.
LDAP_CONNECT_ERROR                0x5b   Cannot establish a connection to the server.

LDAP_NOT_SUPPORTED                0x5c   The feature is not supported.
LDAP_CONTROL_NOT_FOUND            0x5d   The ldap function did not find the specified control.

LDAP_NO_RESULTS_RETURNED          0x5e   The feature is not supported.
LDAP_MORE_RESULTS_TO_RETURN       0x5f   Additional results are to be returned.

LDAP_CLIENT_LOOP                  0x60   Client loop was detected.
LDAP_REFERRAL_LIMIT_EXCEEDED      0x61   The referral limit was exceeded.
LDAP_SASL_BIND_IN_PROGRESS        0x0E   Intermediary bind result for multi-stage binds

```











