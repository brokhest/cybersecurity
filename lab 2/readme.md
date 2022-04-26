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
