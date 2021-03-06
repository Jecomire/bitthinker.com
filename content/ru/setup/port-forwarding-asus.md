<!--
Title: Проброс (открытие) портов на роутере на примере Asus&nbsp;RT&#x2010;N66U
Description: В этой статье я расскажу про проброс портов на роутере на примере настройки Asus RT-N66U для работы с DC++
Date: 2013/05/09
Tags: setup, asus
-->

**Порты...?!**

В некоторых ситуациях **проброс портов на роутере** просто необходим.
Например, Вы "сидите за роутером (маршрутизатором)" и хотите что бы люди при обращении
к Вашему роутеру попадали прямиком на Вашу машину (например, это может пригодиться для DC++-клиента
или игрового/web сервера).

В общем то, здесь нет ничего сложного. Детали, конечно, отличаются от роутера к роутеру,
но *общая идея* остаётся прежней. Во всех относительно новых роутерах есть такая функциональность.
Если же Вы не можете найти ничего подобного, обратитесь к "Руководству пользователя" Вашего роутера. 
Я покажу как это сделать на примере **проброса портов для настройки DC++-клиента на роутере Asus RT-N66U**<!--cut-here-->.

## Настраиваем внутрисетевой IP

Для начала, убедитесь в что роутер внутри сети статически выдаёт вам IP-адрес (постоянно один и тот же).
На роутере за это отвечает DHCP  Server. Если он включен (посмотрите в настройках,
что-то в районе раздела LAN), то нужно назначить IP-адрес для вашей машины статически в обход DHCP.

На вкладке *LAN-DHCP Server* включаем ручное назначение IP (*Enable Manual Assignment*),
выбираем в поле ниже свой MAC-адрес (если к роутеру подключено несколько машин,
свой MAC можно узнать на вкладке *Карта Сети*, нажав "*Клиенты*", и "*Обновить*".
Зная свой текущий IP, справа в таблице увидите и MAC), желаемый IP (в разумных пределах),
жмём *Добавить* и *Применить*. Всё, после сохранения, у вашей машины внутри сети постоянно будет
указанный IP адрес.

![img][asus-dhcp]


## Проброс (открытие) портов

Теперь смело можно переходить непосредственно к самому главному.
Вкладка *WAN-Virtual Server / Port Forwarding* (*Интернет / Переадрессация портов*)
Включаем опцию *Enable Port Forwarding* (*Включить переадрессацию портов*), если ещё не сделали этого.

И заполняем необходимые поля:

* **Service Name (имя службы)** - ни к чему не обязывающее имя "правила" перенаправления портов;
* **Port Range (диапазон портов)** - диапазон портов (или 1 конкретный), **С** которых роутер
будет перенаправлять входящие соединения;
* **Local IP (локальный IP)** - локальный (внутри вашей сети) IP, **НА** который роутер будет
перенаправлять входящие соединения с портов **&lt;Port Range&gt;**;
* **Local Port (локальный порт)** - номер порта на машине с IP **&lt;Local IP&gt;**
на который роутер будет перенаправлять соединения;
* **Protocol (протокол)** - соединения какого типа следует отлавливать роутеру.

Для настройки DC-клиента, пусть например, требуется открыть (пробросить) 2 порта :
3000 для TCP/UDP и 3001 для TLS (работает по протоколу TCP).

Таким образом, добавляем и заполняем 2 строчки следующим содержанием и жмём *Применить*:

	# name    Port-Range   Local-Ip      Local.Port   Protocol type
	  dc-tcp     3000      192.168.1.2     3000          BOTH    
	  dc-tsl     3001      192.168.1.2     3001          BOTH

![img][asus-port-forwarding]

Здесь - всё. Теперь роутер все входящие TCP/UDP соединения на порты 3000:3001 будет перенаправлять
прямиком на 3000:3001 порты машинки c IP 192.168.1.2.


## Настройка DC++-клиента

Осталось настроить DC-клиент. Открываем настройки соединения
(*Файл-Настройки-Соединение* | *Tools-Preferences-Connection* ; у меня **eiskaltdcpp**).

И выбираем:

* радио-кнопку *Ручной проброс портов*;
* вписываем наши порты в соответствующие поля.

**P. s.** Если роутер поддерживает технологию UPnP, то можно выбрать этот пункт в настройках,
и не вписывать порты&nbsp;- программа должна сама определить, а возможно ещё и на роутере сама пробросит все необходимые порты. Но я люблю ясность во всём :)

![img][eiskalttdcpp]



## Does it work..?

Всё. Осталось проверить что всё работает. Но... есть такие вещи, как Антивирус, Браундмауэр,
и куча куча всего ещё! Они вполне могут блокировать обращение по необходимым портам.
Я думаю, Вам не составит труда их настроить (:

Например, мне, на Fedora 16, пришлось повозиться с **iptables**


	su
	# посмотреть все существующие правила:
	iptables -nvL
	 
	# если, и скорее всего, нужные порты не настроены - исправим это
	# открываем входящие соединения на нужные порты
	iptables -A INPUT -p tcp -m tcp --dport 3000 -j ACCEPT
	iptables -A INPUT -p tcp -m tcp --dport 3001 -j ACCEPT
	iptables -A INPUT -p udp -m udp --dport 3000 -j ACCEPT
	 
	# открываем исходящие соединения с этих портов
	iptables -A OUTPUT -p tcp -m tcp --dport 3000 -j ACCEPT
	iptables -A OUTPUT -p tcp -m tcp --dport 3001 -j ACCEPT
	iptables -A OUTPUT -p udp -m udp --dport 3000 -j ACCEPT
	 
	# ну и проверьте что нет никаких правил, закрывающих *всё*
	# у меня такое было одно:
	# -A INPUT -j REJECT --reject-with icmp-host-prohibited
	# что бы его удалить, нужно выполнить
	iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
	 
	# обязательно проверьте, что все ваши изменения отобразились в выводе
	# iptables-save

Осталось **сохранить правила iptables в fedora 16** (в моём случае)

	# В большинстве дистрибутивов хватит
	service iptables save
	 
	# Там же где во всю царствует Systemd, типа Fedora 16,
	# используем старую версию команды:
	/usr/libexec/iptables.init save
	 
	# на крайний случай, Вы всегда можете поправить конфиги ручками
	vim /etc/sysconfig/iptables

Вот теперь - всё.

---

[Здесь][beeline-faq] можно найти инструкции для настройки тех же самых вещей для многих роутеров
(если у вас не биилайн - это **вообще** не важно). Смотрите пункт *Настраиваем доступ к локальным ресурсам*.



*[DHCP]: Dynamic Host Configuration Protocol

[asus-dhcp]: /blog/content/ru/setup/imgs/asus-dhcp.jpg
"Asus DNS and WINS server settings"

[asus-port-forwarding]: /blog/content/ru/setup/imgs/asus-port-forwarding.gif
"Asus port forwarding configuration"

[eiskalttdcpp]: /blog/content/ru/setup/imgs/eiskalttdcpp-sett-connection.jpg
"Settings-Connection in Eiskalttdcpp-qt"

[beeline-faq]: http://homenet.beeline.ru/routers/index.php
"Рекомендации по использованию роутеров"
