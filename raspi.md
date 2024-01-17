
warto stworzyć swap bo malinki 4GB potrafią zapchać ram podczas instalacji kolla-ansible.  
```bash
sudo fallocate -l 10G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

ważna sprawa. żeby veth miały różne adresy MAC trzeba im wygenerować unikalne seedy.  
```bash
sudo rm /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo dbus-uuidgen --ensure=/etc/machine-id
sudo cp /etc/machine-id /var/lib/dbus/machine-id
```

warto wyłączyć pytanie o hasło do sudo, żeby ciągle nie wpisywać. Możliwe nawet, że ansible czy coś innego tego wymaga.  
w pliku /etc/sudoers zmienić linię z %sudo na  
```
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

ustawienie hostname.
```bash
hostnamectl set-hostname ost61
```
w /etc/hosts trzeba wpisać hostname
#127.0.0.1	ost61

dodać kilka folderów z binarkami do path
```bash
sudo tee -a ~/.bashrc << EOT
export PATH=$PATH:/usr/local/sbin:/usr/sbin:/sbin
EOT
```

konfiguracja eth0 dla networkd
```bash
sudo tee /etc/systemd/network/20-wired.network << EOT
[Match]
Name=eth0

[Network]
DHCP=yes
EOT
```

instalowanie aktualizacji i kilku rzeczy które się potem przydają
```bash
sudo apt remove unattended-upgrades -y
sudo apt update && sudo apt upgrade -y
sudo apt install sshpass lm-sensors net-tools qemu-kvm -y
```

zmodyfikować /etc/sysctl.conf
```
net.ipv4.ip_forward=1
```
i zastosować zmiany
```bash
sudo sysctl -p
```

stworzenie veth0 i veth1
```bash 
sudo tee /etc/systemd/network/veth-openstack-net-itf-veth0.netdev << EOT
#network_interface w globals kolla-ansible
[NetDev]
Name=veth0
Kind=veth
[Peer]
Name=veth0br
EOT
```
```bash
sudo tee /etc/systemd/network/veth-openstack-neu-ext-veth1.netdev << EOT
#neutron_external_interface w globals kolla-ansible
[NetDev]
Name=veth1
Kind=veth
[Peer]
Name=veth1br
EOT
```

w tym momencie wrzucić plik /etc/netplan/50-cloud-init.yaml na raspberry pi dopasowując go do poszczególnej raspberry w następujący sposób
ustawić adres veth0 raspberry pi
```
    veth0:
      addresses:
        - 192.168.2.61/24   # dopasowac adres do raspberry
```

zastosowanie netplana
```bash
sudo systemctl enable systemd-networkd
sudo netplan generate
sudo netplan try
```
jeśli po `sudo netplan try` po pewnej chwili zacznie się odliczanie, to jest dobrze i można wcisnąć enter żeby zachować zmiany  
to zabezpieczenie jest jak przy zmienianiu ustawień monitora, jeśli byśmy się odcięli, to enter nie zostanie wciśnięty i komputer wróci do starych ustawień
