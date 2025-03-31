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
  |  ost61  |       |  ost62  |       |  ost63  |       |  ost64  |              |
  +---------+       +---------+       +---------+       +---------+              |
       | 192.168.1.61    | 192.168.1.62    | 192.168.1.63    | 192.168.1.64      | <- adresy IP są przypisane interfejsom fizycznym RPi  
       | vlan2, vlan1    |vlan2, vlan1     |vlan2, vlan1     |vlan2 vlan1        |    przed koniguracją malin dla OpenStack; po konfiguracji 
     +---------------------------------------------------------+                 |    adresy te będą przypisane wirtualnym urządzeniom
     |      vlan1 (tagowany)   switch tp-link                  |                 |    w poszczególonych malinach - por. rys. poniżej
     +----------┴----------------------------------------------+                 |
                | VLAN 0 (nietagowany)                                           |    vlan1 - VLAN tagowany od switcha TP link do 
                |                                                                |
     +--------------------------------+            wifi FreshTomato09            |
     |   router linksys 192.168.1.1   |------------------------------------------+
     +----------┬---------------------+            10.0.1.0/24 dhcp
                |
               WAN
- vlan2 (numer przykładowy) to VLAN tagowany, obejmuje switch tp-link i wyszystkie interfejesy/bidge na drodze aż po
urządzenia veth1 w poszczególnych RPi (ale na łączu veth0br-veth0 vlan2 nie ma, por.rys. poniżej)
- VLAN to nietagowany (vlan1) jest obecny wszędzie począwszy od urządzenia linksys
```

Wszystkie hosty (Raspberry Pi) klastra mają interfejs fizyczny podłączony do switcha TP-Link, a port eth0 jest dowiązany do wirtualnego bridge wewnątrz Raspberry Pi w celu uzyskania dwóch interfejsów wirtualnych, tutaj veth0 i veth1 (jeden z tych interfejsów przejmie adres IP z sieci lokalnej - to adresy 192.168.1.6x na rysunku powyżej). Wynika to z tego, że używany w projekcie instalator Kolla-Ansible wymaga, aby każdy host w klastrze OpenStack miał co najmniej dwa interfejsy. Modelowo, interfejsy veth0 i veth1 pełnią więc rolę dwóch interfejsów "fizycznego" serwera OpenStack.

Ponadto każdy host Raspberry Pi jest podłączony przez WiFi do routera jako połączenie zapasowe na wypadek "odcięcia" się przy konfigurowaniu głównej sieci naszego DC. Ruter ten separuje też infrastrukturę mini-datacenter od reszty sieci (domowej/akademika, etc.). Szczegóły te przedstawiono graficznie na rysunku poniżej, a dokładne opisy konfiguracyjne są zawarte w dokumencie poradnik.md.

```
Zgodnie z powyższym, sieciówka każdego hosta klastra w aspekcie OpenStack wygląda następująco:

     192.168.1.6x/24    bez adresu IP <- tego chce Kolla-Ansible jako stan początkowy przed instalacją
          vlan1          vlan2, vlan1
vlan1  +----┴----+       +----┴----+     vlan2 (tagowany) tylko na veth1; nietagowany ruch z vlan1 na veth0 i veth1 (sieć flat OpenStack)
       |  veth0  |       |  veth1  |  <- interfejsy wirtualne; w modelu OpenStack odpowiadają fizycznym interfejsom serwera (dla veth0 
       +---------+       +---------+     skonfigurujemy w netplan adres IP z sieci lokalnej, ale już podczas instalacji OpenStack adres ten 
vlan1       |   veth  pairs   |          zostanie przeniesiony na dodatkowo utworzony OpenStack'owy bridge br-ext)
       +---------+       +---------+
vlan1  | veth0br |       | veth1br | vlan2, vlan1
       +---------+       +---------+
          +-┴-----------------┴-+
          |        brmux        | # ten bridge docelowo nie musi mieć nadanego adresu IP
          +----------┬----------+   vlan2, vlan1
 vlan1 (nietag) +---------+ 
 vlan2 tag      |  eth0   |  fizyczny iterface RbPi, docelowo bez adresu IP
=============== +----┬----+ ================
                     |       fizyczna sieć zewnętrzna

- vlan2 (numer przykładowy) to VLAN tagowany dla sieci providerskiej OpenStack, obejmuje switch tp-link i wszystkie
  interfejesy/bidge na drodze aż po veth1 (ale na łączu brmux-veth0br vlan2 nie ma); vlan2 zostanie zadeklarowany w pliku
  konfiguracyjnym OpenStack dla Neutrona, ml2_conf.ini, w celu zrealizowania tagowanej sieci providerskiej.
- vlan1 to czysty Ethernet (VLAN 1 - nietagowany), jest obecny wszędzie, począwszy od urządzenia linksys. Słuzy do realizacji
  sieci providerskiej typu flat (obejmującej wszystkie maszyny OpenStack). Dla podniesienia stopnia izolacji ruchu
  mógłby to też być VLAN tagowany, inny niż vlan2. Wtedy vlan1 ten albo mógłby być zakończony (zdjęty tag) w którymś z
  urządzeń brmux, veth0br, veth0 (czyli na wyjściu z veth0 w górę zawsze dotrze VLAN nietagowany - sieć providerska flat);
  albo mógłby być wyprowadzony w górę z veth0 jako VLAN tagowany i wtedy służyć za sieć providerską tagowaną (inną niż vlan2).
```
