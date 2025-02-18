# slcd-openstack

## Uwaga: ##
**Poniżej przedstawiono zarys projektu. Bardziej szczegółowe instrukcje instalacyjne są zawarte w pliku poradnik.md. Wskazówki na okoliczność występowania problemów przedstawiono w pliku problemy.md.**

## Cel i zasady ## 

Tworzymy mini-datacenter (DC) z użyciem minikomputerów Raspberry Pi i oprogramowania Openstack. Celem jest zapoznanie się z podstawami administrowania DC (na przykładzie OpenStack) ze szczególnycm uwzględnieniem aspektów sieciowych (konfigurowanie środowiska sieciowego DC). Każdy zespół otrzynuje egzemplarz klastra (4xRbPi + switch TP-Link + ruter WiFi), na którym konfiguruje środowisko OpenStack z wykorzystaniem pakietu Kolla-Ansible. Możliwa jest praca zdalna z dostępem do klastra przez sieć VPN (korzystamy np. z aplikacji Zero-Tier).

## Architektura DC ##
Ogólny schemat sieciowy naszego DC przedstawiono na rysunku poniżej. W każdym konkretnym przypadku adresy IP trzeba będzie dostosować do własnego środowiska sieciowego. Zaznaczona na rysunku sieć VLAN reprezentuje tzw. fizyczną _sieć dostawcy_ w OpenStack (ang. _provider physical network_) - o sieciach dostawców dowiemy się więcej w stosownym czasie.

```
Fizyczna struktura sieciowa klastra
       +-----------------+-----------------+-----------------+-------------------+
       | wlan0           | wlan0           | wlan0           | wlan0             |
  +---------+       +---------+       +---------+       +---------+              |
  |  ost61  |       |  ost62  |       |  ost63  |       |  ost64  |     vlan1    |
  +---------+       +---------+       +---------+       +---------+              |
       | 192.168.1.61    | 192.168.1.62    | 192.168.1.63    | 192.168.1.64      | <- adresy IP sda przypidsane interfejsom fizycznym RPi  
       | vlan2, vlan1    |vlan2, vlan1     |vlan2, vlan1     |vlan2              |    przed koniguracją malin dla OpenStack; po konfiguracji 
     +---------------------------------------------------------+                 |    adresy te będą przypisane wirtualnym urządzeniom w poszczegłonych malinach - por. rys poniżej
     |                      switch tp-link                     |                 |
     +---------------------------------------------------------+                 |
                |                                                                |
      vlan1     |                                                                |
     +--------------------------------+            wifi FreshTomato09            |
     |   router linksys 192.168.1.1   |------------------------------------------+
     +--------------------------------+            10.0.1.0/24 dhcp
                |
               WAN
- vlan2 (numer przykładowy) to VLAN tagowany, obejmuje switch tp-link i wyszystkie interfejesy/bidge na drodze aż po
urządzenia veth1 w poszczególnych RPi (ale na łączu veth0br-veth0 vlan2 nie ma, por.rys. poniżej)
- VLAN to nietagowany (vlan1) jest obecny wszędzie począwszy od urządzenia linksys
```

Wszystkie hosty (Raspberry Pi) klastra mają interfejs fizyczny podłączony do switcha TP-Link, a port eth0 jest dowiązany do wirtualnego bridge wewnątrz Raspberry Pi w celu uzyskania dwóch interfejsów wirtualnych, tutaj veth0 i veth1 (jeden z tych interfejsów przejmie adres IP z sieci lokalnej - to adresy 192.168.1.6x na rysunku powyżej). Wynika to z tego, że używany w projekcie instalator Kolla-Ansible wymaga, aby każdy host w klastrze OpenStack miał co najmniej dwa interfejsy. Ponadto każdy host Raspberry Pi jest podłączony przez WiFi do routera jako połączenie zapasowe na wypadek "odcięcia" się przy konfigurowaniu głównej sieci naszego DC. Ruter ten separuje też infrastrukturę mini-datacenter od reszty sieci (domowej/akademika, etc.). Szczegóły te przedstawiono graficznie na rysunku poniżej, a dokładne opisy konfiguracyjne są zawarte w dokumencie poradnik.md.

```
Zgodnie z powyższym, sieciówka każdego hosta klastra w aspekcie OpenStack wygląda następująco:

     192.168.1.6x/24    bez adresu IP <- tego chce Kolla-Ansible jako stan początkowy przed instalacją
       +---------+       +---------+     vlan2, vlan1 - na veth1 są obecne: tagowany vlan2 i nietagowany vlan1; na veth0 jest tylko vlan1
vlan1  |  veth0  |       |  veth1  |  <- interfejsy wirtualne; w modelu OpenStack odpowiadają fizycznym interfejsom serwera (dla veth0 
       +---------+       +---------+     skonfigurujemy w netplan adres IP z sieci lokalnej, ale już podczas instalacji OpenStack adres ten 
            |   veth  pairs   |          zostanie przeniesiony na dodatkowo utworzony bridge br-ext)
       +---------+       +---------+
vlan1  | veth0br |       | veth1br | vlan2, vlan1
       +---------+       +---------+
          +-┴-----------------┴-+
          |        brmux        | # ten bridge docelowo nie musi mieć nadanego adresu IP
          +----------┬----------+   vlan2, vlan1
                +---------+ 
                |  eth0   |    fizyczny iterface RbPi, docelowo bez adresu IP
                +---------+

- vlan2 (numer przykładowy) to VLAN tagowany dla sieci providerskiej OpenStack, obejmuje switch tp-link i wyszystkie
  interfejesy/bidge na drodze aż po veth1 (ale na łączu veth0br-veth0 vlan2 nie ma); vlan2 zostanie zadeklarowany w pliku
  konfiguracyjnym OpenStac'a ml2
- VLAN nietagowany (vlan1) jest obecny wszędzie, począwszy od urządzenia linksys. Dla podniesienia stopnia izolacji ruchu
  mógłby to też być VLAN tagowany, inny niż vlan2, i wtedy VLAN ten musiałby być zakończony (zdjęty tag) w urządzeniach
  brmux (czyli do veth0 zawsze dotrze VLAN nietagowany)
```
