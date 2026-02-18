# Linux CLI Cheat Sheet -- Ubuntu vs RedHat

## 1️⃣ Gestion des paquets

  ----------------------------------------------------------------------------
  Action         Ubuntu (APT)                   RedHat (DNF/YUM)
  -------------- ------------------------------ ------------------------------
  Installer      `sudo apt install <package>`   `sudo dnf install <package>`
  package                                       

  Supprimer      `sudo apt remove <package>`    `sudo dnf remove <package>`
  package                                       

  Mettre à jour  `sudo apt update`              `sudo dnf check-update`
  liste                                         

  Mettre à jour  `sudo apt upgrade`             `sudo dnf upgrade`
  système                                       

  Installer      `sudo dpkg -i <file>.deb`      `sudo rpm -i <file>.rpm`
  fichier local                                 
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

## 2️⃣ Services (systemd)

``` bash
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl status <service>
sudo systemctl enable <service>
```

------------------------------------------------------------------------

## 3️⃣ Gestion réseau

### Ubuntu (Netplan)

``` bash
/etc/netplan/*.yaml
sudo netplan apply
```

### RedHat (NetworkManager)

``` bash
nmcli connection show
nmcli connection up <connection_name>
nmtui
```

------------------------------------------------------------------------

## 4️⃣ Firewall

### Ubuntu -- ufw

``` bash
sudo ufw status
sudo ufw enable
sudo ufw allow 22
sudo ufw deny 80
```

### RedHat -- firewalld

``` bash
sudo firewall-cmd --state
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --reload
```

------------------------------------------------------------------------

## 5️⃣ Sécurité

  Ubuntu     RedHat
  ---------- ---------------------------
  AppArmor   SELinux activé par défaut

``` bash
getenforce
sudo setenforce 0
sudo setenforce 1
```

------------------------------------------------------------------------

## 6️⃣ Utilisateurs & sudo

``` bash
sudo adduser <username>
sudo useradd <username>
passwd <username>
sudo usermod -aG sudo <username>
sudo usermod -aG wheel <username>
```

------------------------------------------------------------------------

## 7️⃣ Chemins importants

  Service       Ubuntu                   RedHat
  ------------- ------------------------ ------------------------
  Apache        `/etc/apache2/`          `/etc/httpd/`
  Logs Apache   `/var/log/apache2/`      `/var/log/httpd/`
  SSH           `/etc/ssh/sshd_config`   `/etc/ssh/sshd_config`

------------------------------------------------------------------------

## 8️⃣ Commandes utiles Linux

``` bash
uname -a
hostnamectl
df -h
free -m
ps aux
top
htop
ls -l
tree
find / -name <file>
grep "text" <file>
```

------------------------------------------------------------------------

## Notes rapides entretien

-   Ubuntu → APT / dpkg, AppArmor, ufw
-   RedHat → DNF/YUM, SELinux, firewalld
-   systemctl identique sur les deux
-   Répertoires peuvent changer (/etc/apache2 vs /etc/httpd)
-   SELinux et firewall sont causes fréquentes d'erreurs
