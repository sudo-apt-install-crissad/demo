# Демо экзамен

### Модуль 1: Выполнение работ по проектированию сетевой инфраструктуры  
 
## **Задание 1 модуля 1** 
 
**1.	Выполните базовую настройку всех устройств:**  
 
**a.	Присвоить имена в соответствии с топологией:**  
 
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/Topologiya.png)  

### **MACHINES**
 
```
su -
toor
hostnamectl set-hostname HOSTNAME; exec bash 
нажать enter
```

![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/name_change.png)  

Повторяем идентичные действие с остольными машинами


**b.	Рассчитайте IP-адресацию IPv4 и IPv6. Необходимо заполнить таблицу №1, чтобы эксперты могли проверить ваше рабочее место.**  
 
**Таблица №1**  
 
| Имя устройства | IPv4            | IPv6                | Имя интерфейса |
|-----------------|-----------------|--------------------|----------------|
| CLI             | 10.0.1.2/24 255.255.255.0     | 2001:33::33/64       | enp0s8 to ISP         |
| ISP             | 10.0.1.1/24 255.255.255.0     | 2001:33::1/64	       | enp0s9 ISP to CLI          |
|                 | 192.168.0.1/24 255.255.255.0     | 2001:11::1/64                   | enp0s10 ISP to HQ-R               |
|                 | 192.168.1.1/24 255.255.255.0     | 2001:22::1/64                   | enp0s9 ISP to BR-R              |
| HQ-R            | 192.168.0.2/24 255.255.255.0        | 2000:100::3f/122        | enp0s8 HQ-R to ISP           |
|                 | 10.0.0.1/26 255.255.255.0        |    2001:100::1/64		                | enp0s8 to HQ               |
| HQ-SRV          | 10.0.0.2/26 255.255.255.192        | 2000:100::1/122       | enp0s8 to HQ-R           |
| BR-R            | 192.168.1.2/24 255.255.255.0     | 2000:200::f/124       | enp0s8 BR-R to ISP           | 
|                 | 10.0.2.1/28 255.255.255.240     | 2001:100::2/64	                   | enp0s9 to BR               | 
| BR-SRV          | 10.0.2.2/28 255.255.255.240     | 2000:200::1/124       | enp0s8 to BR-R           | 
| HQ-CLI          | 10.0.0.3/26 255.255.255.192        | 2000:100::5/122       |            | 
| HQ-AD           | 10.0.0.4/26 255.255.255.192       | 2000:100::2/122       |            |  
 

 
**c.	Пул адресов для сети офиса BRANCH - не более 16**  
 
255.255.255.240 = /28  

**d.	Пул адресов для сети офиса HQ - не более 64**  
 
255.255.255.192 = /26  
 
## Настройка адресации  
 
**Назначаем адресацию согласно ранее заполненной таблицы №1**  

![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/everything_adapter_config.png)  
 
```
ip -c a
```
 
```
su -
toor
enter
nmtui
```

После установки ip-адресов необходимо переподключить интерфейсы.  
 
Произведём настройку маршрута для CLI

```
ip route add default via 192.168.0.2
```

необходимо включить опцию forwarding:  
 
``` 
vim /etc/net/sysctl.conf
:wq
enter
```
 
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/forwarding_on.png)  
 
```
systemctl restart network
ip -c a
```
## **MACHINES**  

```
ip -c a
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/addresses.png)

**2.	Настройте внутреннюю динамическую маршрутизацию по средствам FRR. Выберите и обоснуйте выбор протокола динамической маршрутизации из расчёта, что в дальнейшем сеть будет масштабироваться.**  
**Настройка FRR**  
Для настройки потребуется включённый forwarding, настройка выполнялась ранее.  

Предварительно настроем интерфейс туннеля:
```
nmtui
``` 
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/tunnel_conf.png)  


**Обоснование**: Настройку динамическое маршрутизации производим с помощью протокола **OSPF** – Данный протокол динамической сети позволяет разделять сеть на логические области, что делает его масштабируемым для больших сетей.  
Каждая область может иметь свою таблицу маршрутизации, что уменьшает нагрузку на маршрутизаторы и улучшает производительность сети.  
## **ROUTERS**

```
nano /etc/frr/daemons
меняем строчку
ospfd=no на строчку
ospfd=yes
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/frr-daemons.png)  

```
ctrl-x
y
systemctl enable --now frr
vtysh
conf t
router ospf
passive-interface default
network 10.0.0.0/26 area 0
network 172.16.0.0/24 area 0
exit
interface tunnel1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
```
Зададим TTL для OSPF  
```
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
```

Временно выключаем сервис службы **firewalld**  
```
systemctl stop firewalld.service
systemctl disable --now firewalld.service
```
```
systemctl restart frr
```

![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/configuration.png)

**3.	Настройте автоматическое распределение IP-адресов на роутере HQ-R.**
**a.	Учтите, что у сервера должен быть зарезервирован адрес.**
## **HQ-R**
```
nano /etc/sysconfig/dhcpd
DHCPDARGS=enp0s9
ctrl-x
y
enter
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-R(9).png)  
```
cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
```
поправляем файл:  
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/dhcpd%20config.png)  


Проверяем файл на правильность заполнения. Обратите внимание, что файл заполнен в точности со скриншотом выше. (фигурные скобки в начале и конце секции, знаки **;** и тд.)
```
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

```
systemctl enable --now dhcpd
systemctl status dhcpd
journalctl -f -u dhcpd
```

## **HQ-SRV**
```
systemctl restart network
```
После проделанных манирпуляций HQ-SRV должен получить статический адрес.
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/screenshot_bulk/result.png)  

**4.	Настройте локальные учётные записи на всех устройствах в соответствии с таблицей 2.**  
**Таблица №2**  

| Логин | Пароль | Примечание |
| :---         |     :---:      |          ---: |
| Admin   | P@ssw0rd     | CLI HQ-SRV HQ-R    |
| Branch admin     | P@ssw0rd       | BR-SRV BR-R      |
| Network admin     | P@ssw0rd       | HQ-R BR-R BR-SRV      |

## **CLI**
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/CLI(5).png)

## **BR-R**

```
useradd branch-admin -m -c "Branch admin" -U
passwd branch-admin
useradd network-admin -m -c "Network admin" -U
passwd network-admin
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/BR-R(9).png)  

**BR-SRV**
```
useradd branch-admin -m -c "Branch admin" -U
passwd branch-admin
useradd network-admin -m -c "Network admin" -U
passwd network-admin
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/BR-SRV(5).png)  

## **HQ-R**

```
useradd network-admin -m -c "Network admin" -U
passwd network-admin
useradd admin -m -c "Admin" -U
passwd admin
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-R(12).png)

## **HQ-SRV**
```
useradd admin -m -c "Admin" -U
passwd admin
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-SRV(4).png)  

**5.	Измерьте пропускную способность сети между двумя узлами HQ-R-ISP по средствам утилиты iperf 3. Предоставьте описание пропускной способности канала со скриншотами.**
## **ISP**

```
systemctl enable --now iperf3
```
## **HQ-R**
```
systemctl enable --now iperf3
iperf3 -c 192.168.0.1 -f m --get-server-output
```

![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-R(13).png)
 
**6.	Составьте backup скрипты для сохранения конфигурации сетевых устройств, а именно HQ-R BR-R. Продемонстрируйте их работу.**  

## **HQ-R**
Cоздадим дирректорию для сохранения файла:  
```
mkdir /opt/backup/
```
Запишем скрипт:  
```
nano backup-script.sh
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-R(14).png)  
```
ctrl-x
y
жмём enter
chmod +x backup-script.sh
./backup-script.sh
```
Проверим содержимое файла бэкапа. Обратите внимание, что название архива варьируется, в зависимости от даты, когда была сделана резервная копия! Будьте внимательны.
```
tar -tf /opt/backup/hq-r-06.01.24.tgz | less
```

## **BR-R**
Cоздадим дирректорию для сохранения файла:  
```
mkdir /opt/backup/
```
Запишем скрипт: 
```
nano backup-script.sh
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/BR-R(10).png)  
```
ctrl-x
y
жмём enter
chmod +x backup-script.sh
./backup-script.sh
```
Проверим содержимое файла бэкапа. Обратите внимание, что название архива варьируется, в зависимости от даты, когда была сделана резервная копия! Будьте внимательны.
```
tar -tf /opt/backup/hq-r-06.01.24.tgz | less
```

**7.	Настройте подключение по SSH для удалённого конфигурирования устройства HQ-SRV по порту 2222. Учтите, что вам необходимо перенаправить трафик на этот порт посредством контролирования трафика.**
## **HQ-SRV**

```
nano /etc/openssh/sshd_config
```

![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-SRV(6).png)  

```
ctrl-x
y
enter
systemctl restart sshd
```
## **HQ-R**
```
nft add table inet nat
nft add chain inet nat prerouting '{ type nat hook prerouting priority 0; }'
nft add rule inet nat prerouting ip daddr 192.168.0.2 tcp dport 22 dnat to 10.0.0.2:2222
nft list ruleset | tail -n 7 | tee -a /etc/nftables/nftables.nft
systemctl enable --now nftables
systemctl restart nftables
nft list ruleset
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-R(15).png)  

**8.	Настройте контроль доступа до HQ-SRV по SSH со всех устройств, кроме CLI.**

## **HQ-SRV**

```
systemctl enable --now nftables.service
nft add rule inet filter input ip saddr 10.0.1.2 tcp dport 2222 counter drop
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-SRV(7).png)  

```
nano /etc/nftables/nftables.nft
```
Удаляем всё содержимое файла до закоментированных строк

```
nft list ruleset | tee -a /etc/nftables/nftables.nft
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-SRV(8).png)  

```
systemctl restart nftables
nft list ruleset
```
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/HQ-SRV(9).png)  

Выполняем проверку:  
## **CLI**
![image](https://github.com/sudo-apt-install-crissad/demo/blob/main/Screenshot/CLI(6).png)  
В результате настройки, соединение с сервером **HQ-SRV** по ssh установить не удастся.  
  
