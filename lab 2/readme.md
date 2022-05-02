# Лабораторная работа 2
Создаём 2 виртуальные машины на DO

Одна нужна для прокси, вторая для ELK.

Заходим на вм proxy

Производим стандартные sudo apt update/upgrade

# Установка squid

sudo apt install squid

# Настройка squid

заходим в конфиг:
sudo nano /etc/squid/squid.conf

Переходим до строчки include /etc/squid/conf.d/*

и дописываем следующую информацию:

![](screens/1.png)

Сохраняем файл, выходим

# Установка Apache

sudo apt install apache2-utils

sudo htpasswd -c /etc/squid/passwords *username*

и запускаем сервер

sudo systemctl start squid

sudo systemctl enable squid

sudo ufw allow 3128 (порт squid)

Проверка Proxy:
![](screens/2.png)
![](screens/3.png)
![](screens/4.png)


# Установка elasticsearch

Переходим на вм ELK

Проводим стандартные sudo apt update/upgrade

Устанавливаем java

sudo apt install default-jre

sudo apt install default-jdk

Переходим к самому elasticsearch

Для начала получаем ключ elasticsearch gpg

curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

и выполняем следующую команду:

echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

Наконец, устанавливаем elasticsearch

sudo apt install elasticsearch

# Настройка Elasticsearch

Переходим в конфиг файл:

sudo nano /etc/elasticsearch/elasticsearch.yml

Вписываем следующие строки:

![](screens/5.png)

![](screens/6.png)

Создаём пользователей:

sudo -u root /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto

Сохраняем полученные данные (потом понадобятся)

Запускаем:

sudo systemctl start elasticsearch

sudo systemctl enable elasticsearch

# Установка Kibana

sudo apt install kibana

# Настройка Kibana

Переходим в конфиг файл:

sudo nano /etc/kibana/kibana.yml

раскоменчиваем эти строки 

![](screens/7.png)

![](screens/8.png)

Добавляем пароль (полученный ранее)

sudo -u root /usr/share/kibana/bin/kibana-keystore create

sudo -u root /usr/share/kibana/bin/kibana-keystore add elasticsearch.password

И запускаем 

sudo systemctl start kibana

sudo systemctl enable kibana

# Установка Logstash

sudo apt install logstash

# Настройка Logstash

Переходим в конфиг файл:

sudo nano /etc/logstash/conf.d/logstash.conf

Вписываем следующее:
``` properties
input {
    beats {
        port => 5044
    }
}
output {
  elasticsearch {
    hosts => ["167.99.242.171:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata[version]}-%{+YYYY.MM.dd}"
    user => "broha"
    password => "123456"
  }
}
```

Проверяем, что всё работает:

sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

В выводе должно быть OK

Запускаем:

sudo systemctl start logstash

sudo systemctl enable logstash

# Установка Filebeat

Переходим на proxy сервер

Прописываем следующие команды для обновления пакетов:

curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

И устанавливаем Filebeat

sudo apt install filebeat

# Настройка Filebeat

sudo filebeat modules enable system

Затем переходим в конфиг файл:

sudo nano /etc/filebeat/filebeat.yml

вносим следующие изменения:

![](screens/9.png)

Проверяем конфигурацию следующей командой:

sudo filebeat -e test output

Далее выполняем следующие 3 команды:

```properties
sudo filebeat setup --pipelines --modules system
sudo filebeat setup -e --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["167.99.242.171:9200"]' -E 'output.elasticsearch.username="broha"' -E 'output.elasticsearch.password="123456"'
sudo filebeat setup -e --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["167.99.242.171:9200"]' -E 'output.elasticsearch.username="broha"' -E 'output.elasticsearch.password="123456"' -E setup.kibana.host=167.99.242.171:5601
```

Запускаем Filebeat

sudo systemctl start filebeat

sudo systemctl enable filebeat

Проверяем работу:

curl -u broha:123456 -XGET 'http://167.99.242.171:9200/filebeat-*/_search?pretty'


GROK:

#Pattern 1: 

Входное сообщение:

May  2 12:41:40 proxy sshd[50528]: pam_unix(sshd:session): session closed for user root

```
%{SYSLOGTIMESTAMP} %{WORD:server_name} %{DATA:protocol}: %{GREEDYDATA:desc}
```
Выход:

```JSON
{
  "server_name": "proxy",
  "protocol": "sshd[50528]",
  "desc": "pam_unix(sshd:session): session closed for user root"
}
```

#Pattern 2:

Входное сообщение:

May 2 12:35:10 proxy sshd[50528]: Accepted publickey for root from 109.126.4.148 port 3103 ssh2: RSA SHA256:121XA1vCtlpyzu1BluUgaCKfxqfeNwV/z6ry0ZeN0s4

```
%{SYSLOGTIMESTAMP} %{WORD:server_name} %{GREEDYDATA:protocol}: %{GREEDYDATA:desc} from %{IP:src_ip} port %{NUMBER:port} ssh2: RSA SHA256:%{GREEDYDATA:key}
```

Выход:

```JSON
{
  "src_ip": "109.126.4.148",
  "server_name": "proxy",
  "protocol": "sshd[50528]",
  "port": "3103",
  "key": "121XA1vCtlpyzu1BluUgaCKfxqfeNwV/z6ry0ZeN0s4",
  "desc": "Accepted publickey for root"
}
```

#Pattern 3:

Входное сообщение:

May  2 13:35:08 proxy kernel: [365572.440039] [UFW BLOCK] IN=eth0 OUT= MAC=d6:11:17:6f:27:3d:fe:00:00:00:01:01:08:00 SRC=185.191.34.200 DST=164.90.163.85 LEN=40 TOS=0x00 PREC=0x00 TTL=249 ID=33106 PROTO=TCP SPT=42458 DPT=4708 WINDOW=1024 RES=0x00 SYN URGP=0 

```
%{SYSLOGTIMESTAMP} %{WORD:server_name} %{GREEDYDATA:service}: \[%{GREEDYDATA}\] \[UFW BLOCK\] IN=%{WORD:input} OUT=%{GREEDYDATA:output} MAC=%{GREEDYDATA:mac} SRC=%{IP:src_ip} DST=%{IP:dst_ip} LEN=%{NUMBER:len} TOS=%{GREEDYDATA:tos} PREC=%{GREEDYDATA:prec} TTL=%{NUMBER:ttl} ID=%{NUMBER:id} PROTO=%{WORD:protocol} SPT=%{NUMBER:spt} DPT=%{NUMBER:dpt} WINDOW=%{NUMBER:window} RES=%{GREEDYDATA:res} SYN URGP=%{NUMBER:urgp}
```

Выход:

```JSON
{
  "server_name": "proxy",
  "res": "0x00",
  "dpt": "4708",
  "ttl": "249",
  "mac": "d6:11:17:6f:27:3d:fe:00:00:00:01:01:08:00",
  "dst_ip": "164.90.163.85",
  "output": "",
  "src_ip": "185.191.34.200",
  "urgp": "0",
  "input": "eth0",
  "protocol": "TCP",
  "len": "40",
  "prec": "0x00",
  "service": "kernel",
  "spt": "42458",
  "tos": "0x00",
  "id": "33106",
  "window": "1024"
}
```




