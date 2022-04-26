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


