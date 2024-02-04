
## Problemy które mogą się przytrafić
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

### 5. Systemd-networkd nie odświeża całej konfiguracji przy `systemctl restart systemd-networkd`
Okazuje się że konfiguracja bridge nie aktualizuje się po samym restarcie, trzeba go wcześniej usunąć:
```bash
ip link set down brmux
ip link del dev brmux
systemctl restart systemd-networkd
```
