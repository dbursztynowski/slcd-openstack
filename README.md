# slcd-openstack

Użyliśmy jednego raspberry pi 4GB jako maszyny do zarządzania klastrem.

Reszta robi jako zwykłe hosty w klastrze.

Komendy konfiguracyjne raspberry pi managementowej i pozostałych są oddzielnie w plikach mgmt.txt i raspi.txt.

Wszystkie hosty klastra mają połączenie ethernet wykorzystane jako 2 veth i połączenie po wifi jako interfejs managementowy.

Zapomnieliśmy już od spotkania czy połączenie wifi na hostach miało służyć jako interfejs managementowy, ale przy stworzeniu veth na więcej niż jednym hoście, były problemy z komunikacją. Tylko jedno raspberry pi spośród skonfigurowanych do veth było osiągalne.

Na razie liczymy że to celowe i w nie będzie miało znaczenia potem, a jeśli nie to będziemy szukać powodu.

Nie mieliśmy potrzeby używać monitora i klawiatury do włączenia ssh, w aktualnym Raspberry Pi Imager przy użyciu wbudowanego obrazu ubuntu server, przed nagraniem karty proponowana jest zmiana niektórych domyślnych ustawień, w tym włączenie ssh i ustawienie loginu i nazwy użytkownika.

```
Sieciówka każdego hosta klastra

192.168.1.61/24   bez adresu IP (tego chce Kolla-Ansible)
  +---------+       +---------+
  |  veth0  |       |  veth1  |    <==== to będą interfejsy "pseudofizyczne" dla Kolla-Ansible/OpenStack'a
  +---------+       +---------+
       |   veth  pairs   |
  +---------+       +---------+
  | veth0br |       | veth1br |
  +---------+       +---------+
     +-┴-----------------┴-+
     |        brmux        | # 192.168.1.6x/24  <= obecnie nie ustawiamy tego IP 6x (brmux jest tylko w L2)
     +----------┬----------+
           +---------+
           |  eth0   |    fizyczny itf RbPi
           +---------+
```
