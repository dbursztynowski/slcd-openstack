# slcd-openstack
Paweł Nikitin  
Karol Grabowski  
Kamil Dudek  

### Aktualny status:  
`kolla-ansible -i all-in-one deploy` wykonało się pomyślnie dosłownie 15 minut temu po rozwiązaniu problemu z openvswitchem.

## WIP. Dokumentacja w wersji w miarę na bieżąco aktualizowanej.  

Użyliśmy jednego raspberry pi 4GB (ost61) jako maszyny do zarządzania klastrem.  
Reszta (ost62, ost63, ost64) robi jako zwykłe hosty w klastrze.  

```
Fizyczna struktura sieciowa klastra
       +-----------------+-----------------+-----------------+-------------------+
       | wlan0           | wlan0           | wlan0           | wlan0             |
  +---------+       +---------+       +---------+       +---------+              |
  |ost61 mgmt|       |  ost62  |       |  ost63  |       |  ost64  |              |
  +---------+       +---------+       +---------+       +---------+              |
       | 192.168.1.61    | 192.168.1.62    | 192.168.1.63    | 192.168.1.64      |
       |                 |                 |                 |                   |
     +---------------------------------------------------------+                 |
     |               switch tp-link 192.168.1.11               |                 |
     +---------------------------------------------------------+                 |
                |                                                                |
                |                                                                |
     +--------------------------------+            wifi FreshTomato09            |
     |   router linksys 192.168.1.1   |------------------------------------------+
     +--------------------------------+
                |
               WAN
```
Interfejsy wifi nie mają na diagramie przypisanych adresów bo kiedy przydzieliliśmy im adresy w dhcp, nie chciały ich przyjąć.  
Chcielibyśmy jeszcze sprawdzić czemu tak jest, na razie nie przeszkodziło nam sprawdzanie na routerze co zostało im przydzielone z dhcp.  

ost61 służy jako maszyna managementowa zamiast vm w virtualbox.  
ost62 dostało w kolla-ansible rolę control i network.  
Wszystkie trzy ost62, ost63, ost64 mają w kolla-ansible rolę compute.  
Chcemy docelowo upchać management, control i network w ost61, jeśli się uda. Na razie, żeby sobie nie generować problemów, jest to rozdzielone między ost61 i ost62.

Komendy konfiguracyjne raspberry pi managementowej i pozostałych są oddzielnie w plikach mgmt.txt i raspi.txt.  
Wszystkie hosty klastra mają port eth0 podłączony do switcha i wirtualizowany jako veth0/1 i połączenie po wifi do routera.  

```
Sieciówka każdego hosta klastra, czyli ost62, ost63, ost64

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

## Napotkane problemy
### 1. Odcięcie komunikacji z 3/4 hostów na interfejsach eth podłączych do switcha.  
Problem pojawiał się zawsze po wgraniu konfiguracji veth.  
Dopiero na drugi dzień zauważyliśmy na switchu, że wszystkie veth mają taki sam adres MAC.  
Okazało się, że wirtualne adresy MAC są seedowane z /etc/machine-id.  
Wszystkie karty sd zostały zapisane z tego samego iso, więc dostały taki sam /etc/machine-id, więc wszystkie interfejsy veth0/1 dostały wygenerowany taki sam adres MAC.  
Naturalnie, switch po zobaczeniu tego samego MAC na nowym porcie nadpisywał sobie stary port i wtedy komunikacja się urywała. Dlatego dostępne przez eth było tylko jedno raspberry pi naraz.  
#### Rozwiązanie: zregenerować /etc/machine-id i /var/lib/dbus/machine-id.  
Dorzucamy to do pliku z komendami dla każdego zwykłego hosta.  
```
sudo rm /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo dbus-uuidgen --ensure=/etc/machine-id
sudo cp /etc/machine-id /var/lib/dbus/machine-id
```
https://askubuntu.com/questions/1126037/netplan-generates-the-same-mac-address-for-bridges-on-two-different-machines  
https://unix.stackexchange.com/questions/402999/is-it-ok-to-change-etc-machine-id  

### 2. Kontenery openvswitch padały z błędem pidfile check failed  
`docker logs openvswitch_db/openvswitch_vswitchd` kończyło się następującymi błędami.  
```
ovsdb-server: /var/run/openvswitch/ovsdb-server.pid: pidfile check failed (No such process), aborting
```
```
ovs-vswitchd: /var/run/openvswitch/ovs-vswitchd.pid: pidfile check failed (No such process), aborting
```
Sami się w to wpakowaliśmy instalując openvswitcha na hostach, podczas szukania rozwiązania innych problemów.  
Podkusił do tego następujący błąd przy `netplan try/apply`.  
```
Cannot call Open vSwitch: ovsdb-server.service is not running.
```
Niby netplan był wgrywany, ale chociażby problem nr 1 powyżej powodował zastanowienie, czy jednak nie brakuje openvswitcha.  
Openvswitch na kontenerach wnioskował na podstawie pidfile w /var/run, że service już działa i wywalał się na sprawdzaniu czy faktycznie już/jeszcze działa.  
Swoją drogą zdziwiło nas trochę, że /var/run jest udostępnione między hostem, a kontenerami openvswitch.  
#### Rozwiązanie: naturalnie `sudo apt remove openvswitch-switch-dpdk --autoremove`, a najlepiej w ogóle nie instalować  
Nie bez powodu w instrukcji nie było instalowania openvswitch na hostach.

### 4. Sieć riviery chyba blokuje port ntp.  
Nie sprawdzaliśmy dokładnie bo to nie było krytyczne, ale problemy z synchronizacją czasu, kiedy klaster tu był, na to wskazywały.  
W sumie to bardziej taka ciekawostka.  
Rozwiązaliśmy go przenosząc klaster z powrotem do mieszkania kolegów z internetem komórkowym.  

## Co nam poszło "lepiej/prościej" niż spodziewane
1. Drobiazg na początku, ale nie mieliśmy potrzeby używać monitora i klawiatury do włączenia ssh, w aktualnym Raspberry Pi Imager przy użyciu wbudowanego obrazu ubuntu server, przed nagraniem karty, proponowana jest zmiana niektórych domyślnych ustawień, w tym włączenie ssh i ustawienie loginu i nazwy użytkownika.


