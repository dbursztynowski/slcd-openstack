# slcd-openstack

Tworzymy mini-datacenter z użyciem minikomputerów Raspberry Pi i oprogramowania Openstack. Ogólny schemat seciowy naszego DC przedstawiono na rysunku poniżej. W każdym konkretnym przypadku adresy IP trzeba będzie dostosować do własnego środowiska sieciowego. Zaznaczona na rysunku sieć VLAN reprezentuje tzw. fizyczna _sieć dostawcy_ w OpenStack (ang. _provider physical network_) - o sieciach dostawców dowiemy się więcej w stosownym czasie.

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

Wszystkie hosty (Raspberry Pi) klastra mają interfejs fizyczny podłączony do switcha, a port eth0 dowiązany do wirtualnego bridge wewnątrz Raspberry Pi w celu uzyskania dwóch portów, tutaj veth0 i veth1 (jeden z tych interfejsów przejmie adres IP z sieci lokalnej - to adresy 192.168.1.6x na rysunku powyżej). Wynika to z tego, że co najmniej dwa porty są wymagane przez używany w projekcie instalator Kolla-Ansible. Ponadto każdy host Raspberry Pi jest podłączony po WiFi do routera jako połączenie zapasowe na wypadek odcięcia się przy kombinowaniu nad główną siecią. Ruter ten separuje też infrastrukturę mini-datacenter od reszty sieci (domowej/akademika, etc.). Szczegóły te przedstawiono na rysunku poniżej.

```
Zgodnie z powyższym, sieciówka każdego hosta klastra w aspekcie OpenStack wygląda następująco:

192.168.1.6x/24   bez adresu IP (tego chce Kolla-Ansible)
  +---------+       +---------+
  |  veth0  |       |  veth1  |    interfejsy wirtualne dla OpenStack (jeden z nich dostanie adres IP z sieci lokalnej - widoczny na poprzednim rysunku)
  +---------+       +---------+
       |   veth  pairs   |
  +---------+       +---------+
  | veth0br |       | veth1br |
  +---------+       +---------+
     +-┴-----------------┴-+
     |        brmux        | # ten bridge docelowo nie musi mieć nadanego adresu IP
     +----------┬----------+
           +---------+
           |  eth0   |    fizyczny iterface RbPi
           +---------+
```

## Uwaga: ##
** Instrukcje instalacyjne są zawarte w pliku poradnik.md. Wskazówki na okoliczność występowania problemów przedstawiono w pliku problemy.md. **
