# slcd-openstack

Projekt stworzenia mini datacenter z użyciem minikomputerów Raspberry Pi i oprogramowania Openstack

```
Fizyczna struktura sieciowa klastra
       +-----------------+-----------------+-----------------+-------------------+
       | wlan0           | wlan0           | wlan0           | wlan0             |
  +---------+       +---------+       +---------+       +---------+              |
  |  ost61  |       |  ost62  |       |  ost63  |       |  ost64  |              |
  +---------+       +---------+       +---------+       +---------+              |
       | 192.168.1.61    | 192.168.1.62    | 192.168.1.63    | 192.168.1.64      |
       | vlan 2          |vlan 2           |vlan 2           |vlan 2             |
     +---------------------------------------------------------+                 |
     |               switch tp-link                            |                 |
     +---------------------------------------------------------+                 |
                |                                                                |
                |                                                                |
     +--------------------------------+            wifi FreshTomato09            |
     |   router linksys 192.168.1.1   |------------------------------------------+
     +--------------------------------+            10.0.1.0/24 dhcp
                |
               WAN
```

Wszystkie hosty (Raspberry Pi) klastra mają port eth0 podłączony do switcha i wirtualizowany wewnątrz Raspberry Pi jako dwa porty veth0 i veth1 oraz połączenie po wifi do routera jako połączenie zapasowe na wypadek odcięcia się przy kombinowaniu nad główną siecią.  

```
Sieciówka każdego hosta klastra

192.168.1.6x/24   bez adresu IP (tego chce Kolla-Ansible)
  +---------+       +---------+
  |  veth0  |       |  veth1  |    interfejsy wirtualne dla openstacka
  +---------+       +---------+
       |   veth  pairs   |
  +---------+       +---------+
  | veth0br |       | veth1br |
  +---------+       +---------+
     +-┴-----------------┴-+
     |        brmux        | # 192.168.1.6x/24
     +----------┬----------+
           +---------+
           |  eth0   |    fizyczny iterface RbPi
           +---------+
```
