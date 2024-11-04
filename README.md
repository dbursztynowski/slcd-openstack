# slcd-openstack

## Uwaga: ##
**W niniejszym dokumnecie przedstawiono zarys projektu. Instrukcje instalacyjne są zawarte w pliku poradnik.md. Wskazówki na okoliczność występowania problemów przedstawiono w pliku problemy.md.**

Tworzymy mini-datacenter (DC) z użyciem minikomputerów Raspberry Pi i oprogramowania Openstack. Celem jest zapoznanie się z podstawami administrowania DC (na przykładzie OpenStack) ze szczególnycm uwzględnieniem aspektów sieciowych (konfigurowanie środowiska sieciowego DC). Każdy zespół otrzynuje egzemplarz klastra (4xRbPi + switch TP-Link + ruter WiFi), na którym konfiguruje środowisko OpenStack z wykorzystaniem pakietu Kolla-Ansible. Możliwa jest praca zdalna z dostępem do klastra przez sieć VPN (korzystamy np. z aplikacji Zero-Tier).

Ogólny schemat sieciowy naszego DC przedstawiono na rysunku poniżej. W każdym konkretnym przypadku adresy IP trzeba będzie dostosować do własnego środowiska sieciowego. Zaznaczona na rysunku sieć VLAN reprezentuje tzw. fizyczną _sieć dostawcy_ w OpenStack (ang. _provider physical network_) - o sieciach dostawców dowiemy się więcej w stosownym czasie.

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
     |                      switch tp-link                     |                 |
     +---------------------------------------------------------+                 |
                |                                                                |
                |                                                                |
     +--------------------------------+            wifi FreshTomato09            |
     |   router linksys 192.168.1.1   |------------------------------------------+
     +--------------------------------+            10.0.1.0/24 dhcp
                |
               WAN
```

Wszystkie hosty (Raspberry Pi) klastra mają interfejs fizyczny podłączony do switcha TP-Link, a port eth0 jest dowiązany do wirtualnego bridge wewnątrz Raspberry Pi w celu uzyskania dwóch interfejsów wirtualnych, tutaj veth0 i veth1 (jeden z tych interfejsów przejmie adres IP z sieci lokalnej - to adresy 192.168.1.6x na rysunku powyżej). Wynika to z tego, że używany w projekcie instalator Kolla-Ansible wymaga, aby każdy host w klastrze OpenStack miał co najmniej dwa interfejsy. Ponadto każdy host Raspberry Pi jest podłączony przez WiFi do routera jako połączenie zapasowe na wypadek "odcięcia" się przy konfigurowaniu głównej sieci naszego DC. Ruter ten separuje też infrastrukturę mini-datacenter od reszty sieci (domowej/akademika, etc.). Szczegóły te przedstawiono graficznie na rysunku poniżej, a dokładne opisy konfiguracyjne są zawarte w zasadniczej części poradnika.

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
