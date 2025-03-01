# Iptables

Домашнее задание

- реализовать knocking port (centralRouter может попасть на ssh inetrRouter через knock скрипт);

- добавить inetRouter2, который виден (маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост;

- запустить nginx на centralServer;

- пробросить 80й порт на inetRouter2 8080;

- дефолт в инет оставить через inetRouter;

----------------------------------------------------------------------------------------------------------------------------------------

Все задачи выполняются с помощью Ansible при создании виртуальных машин командой `vagrant up`

----------------------------------------------------------------------------------------------------------------------------------------

ЗАДАЧА 1:

Настроить "knocking port" на роутере inetRouter. Результат: к роутеру inetRouter можно подключиться по ssh только если предварительно последовательно постучаться на порты tcp/8881, tcp/7777, tcp/9991.

Подключаемся к роутеру inetRouter: `vagrant ssh inetRouter`

Отключаем ufw:

`systemctl stop ufw`

`systemctl disable ufw`

Устанавливаем пакеты для сохранения правил iptables: `apt install netfilter-persistent iptables-persistent`

Включаем маскарадинг: `iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE`

Настраиваем правила для реализации port knocking:

`iptables -N TRAFFIC`

`iptables -N SSH-INPUT`

`iptables -N SSH-INPUTTWO`

`iptables -A TRAFFIC -p icmp --icmp-type any -j ACCEPT`

`iptables -A TRAFFIC -m state --state ESTABLISHED,RELATED -j ACCEPT`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP`

`iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP`

`iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP`

`iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP`

`iptables -A INPUT -i eth1 -j TRAFFIC`

`iptables -A TRAFFIC -i eth1 -j DROP`

Сохраняем правила: `netfilter-persistent save`

Проверяем. Делаем попытку подключиться по ssh с centralRouter на inetRouter: `ssh root@192.168.255.1`

Видим что подключится не удается. Пробуем теперь постучаться последовательно на порты tcp/8881, tcp/7777, tcp/9991, и после этого подключиться по ssh:

`nmap -Pn --host-timeout 100 --max-retries 0 -p 8881 192.168.255.1`

`nmap -Pn --host-timeout 100 --max-retries 0 -p 7777 192.168.255.1`

`nmap -Pn --host-timeout 100 --max-retries 0 -p 9991 192.168.255.1`

`ssh root@192.168.255.1`

Подключение по ssh работает. Значит port knocking настроен верно. Чтобы не стучать каждый раз вручную (не вводить несколько команд), можно сделать скрипт:

![Image alt](https://github.com/NikPuskov/Iptables/blob/main/iptables1.jpg)

и запускать перед подключением по ssh `knock HOST PORT1 PORT2 PORTx`. Например: `knock 192.168.255.1 8881 7777 9991`

----------------------------------------------------------------------------------------------------------------------------------------

ЗАДАЧА 2:

Добавить inetRouter2, который виден (маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост

Добавляем в нашу схему виртуальную машину inetRouter2 с сетевой адресацией, указанной на схеме. Изменения отображены в Vagrantfile.

----------------------------------------------------------------------------------------------------------------------------------------

ЗАДАЧА 3:

Запустить nginx на centralServer

Подключаемся к centralServer. Устанавливаем и запускаем nginx:

`vagrant ssh centralServer`

`sudo -i`

`apt update`

`apt install nginx`

`systemctl enable --now nginx`

----------------------------------------------------------------------------------------------------------------------------------------

ЗАДАЧА 4:

Пробросить 80й порт на inetRouter2 8080. То есть при подключении с хоста на inet2Router по http:192.168.56.2:8080 трафик должен перенаправиться на centralServer порт 192.168.1.2:80

Подключаемся к inet2Router. `vagrant ssh inet2Router` `sudo -i`

Отключаем ufw: `systemctl stop ufw` `systemctl disable ufw`

Устанавливаем пакеты для сохранения правил iptables: `apt install netfilter-persistent iptables-persistent`

Включаем маршрутизацию транзитных пакетов: `echo "net.ipv4.conf.all.forwarding = 1" >> /etc/sysctl.conf` `sysctl -p`

Настраиваем DNAT: `iptables -t nat -I PREROUTING 1 -i eth2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80`

Включаем маскарадинг: `iptables -t nat -A POSTROUTING -o eth2 -j MASQUERADE`

На centralServer добавляем статический маршрут к сети 192.168.56.0/24 через inet2Router: `nano /etc/netplan/50-vagrant.yaml`

![Image alt](https://github.com/NikPuskov/Iptables/blob/main/iptables2.jpg)

`netplan try`

Для проверки заходим в браузере на хосте по адресу http://192.168.56.2:8080. Если видим приглашение от nginx, значит проброс работает.

----------------------------------------------------------------------------------------------------------------------------------------

ЗАДАЧА 5:

Маршрут по умолчанию для centralServer и centralRouter настроить через inetRouter

На centralServer добавляем маршрут по умолчанию через centralRouter. Отключаем получение маршрута по умолчанию по DHCP:

`nano /etc/netplan/00-installer-config.yaml`

![Image alt](https://github.com/NikPuskov/Iptables/blob/main/iptables3.jpg)

Добавляем статический маршрут по умолчанию через centralRouter: 

`nano /etc/netplan/50-vagrant.yaml`

![Image alt](https://github.com/NikPuskov/Iptables/blob/main/iptables4.jpg)

`netplan try`

Те же действия выполняем на centralServer.

Для проверки заходим на centralServer и выполняем traceroute: `traceroute -n 8.8.8.8`

image
