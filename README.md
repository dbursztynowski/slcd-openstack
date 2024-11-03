# slcd-openstack

Tworzymy mini-datacenter z użyciem minikomputerów Raspberry Pi i oprogramowania Openstack.

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

Wszystkie hosty (Raspberry Pi) klastra mają interfejs fizyczny podłączony do switcha, a port eth0 dowiązany do wirtualnego bridge wewnątrz Raspberry Pi w celu uzyskania dwóch portów, tutaj veth0 i veth1. Wynika to z tego, że co najmniej dwa porty są wymagane przez używany w projekcie instalator Kolla-Ansible. Ponadto każdy host Raspberry Pi jest podłączony po WiFi do routera jako połączenie zapasowe na wypadek odcięcia się przy kombinowaniu nad główną siecią. Ruter ten separuje też infrastrukturę mini-datacenter od reszty sieci (domowej/akademika, etc.).

```
Zgodnie z powyższym, sieciówka każdego hosta klastra w aspekcie OpenStack (z pominięciem WiFi) wygląda następująco:

192.168.1.6x/24   bez adresu IP (tego chce Kolla-Ansible)
  +---------+       +---------+
  |  veth0  |       |  veth1  |    interfejsy wirtualne dla OpenStack (jeden z nich dostanie adres IP z sieci lokalnej)
  +---------+       +---------+
       |   veth  pairs   |
  +---------+       +---------+
  | veth0br |       | veth1br |
  +---------+       +---------+
     +-┴-----------------┴-+
     |        brmux        | # 192.168.1.6x/24 ten bridge docelowo nie musi mieć nadanego adresu IP
     +----------┬----------+
           +---------+
           |  eth0   |    fizyczny iterface RbPi
           +---------+
```

## Uwaga: Instrukcje instalacyjne sa zawarte w pliku poradni.md. Wskazóki na okoliczność występowania problemów zawarto w pliku problemy.md.
