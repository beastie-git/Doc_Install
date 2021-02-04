# Intstallation de ruTorrent

## Prérequis :

apache 2.4.x\
rtorrent 0.9.7\
libtorrent 0.13.7\
ruTorrent 3.10

### Installation de rtorrent :

Créer un home directory :

```
mkdir /data/rtorrent
cd /data/rtorrent
```

configurer rtorrent avec le script du développeur : <https://github.com/rakshasa/rtorrent/wiki/CONFIG-Template>

```
curl -Ls "https://raw.githubusercontent.com/wiki/rakshasa/rtorrent/CONFIG-Template.md" \
    | sed -ne "/^######/,/^### END/p" \
    | sed -re "s:/home/USERNAME:$HOME:" > ./.rtorrent.rc
```

Éditer la config :

```
## On modifie les chemins
method.insert = cfg.basedir,  private|const|string, (cat,"/data/rtorrent/")
method.insert = cfg.watch,    private|const|string, (cat,(cfg.basedir),"torrent/")
...
## Commenter les lignes suivantes
#execute.throw = sh, -c, (cat,\
#    "mkdir -p \"",(cfg.download),"\" ",\
#    "\"",(cfg.logs),"\" ",\
#    "\"",(cfg.session),"\" ",\
#    "\"",(cfg.watch),"/load\" ",\
#    "\"",(cfg.watch),"/start\" ")
...
## Maxer l'utilisation de la ram
pieces.memory.max.set = 1000M
...
## Loguer les connexions RPC
log.xmlrpc = (cat, (cfg.logs), "xmlrpc.log")
...
## Umask pour du 755/644
system.umask.set = 0022
...
## On démarre le téléchargement pour tout les torrents déposés dans le dossier
## Add torrent
#schedule2 = watch_load, 11, 10, ((load.verbose, (cat, (cfg.watch), "*.torrent")))
## Add & download straight away
schedule2 = watch_start, 10, 10, ((load.start_verbose, (cat, (cfg.watch), "*.torrent")))
...
## On lance le socket unix pour la communication avec ruTorrent, en 777 pour qu apache puisse s'y connecter
#system.daemon.set = true
network.scgi.open_local = (cat,(session.path),rpc.socket)
execute.nothrow = chmod,777,(cat,(session.path),rpc.socket)
```

Créer les répertoires :

```
mkdir {download,log,.session,torrent}
```

Ajouter un utilisateur rtorrent :

```
adduser --disabled-login --disabled-password --home /data/rtorrent/ --system rtorrent
```

Affecter les bons droits :

```
chown -R rtorrent:www-data /data/rtorrent/
```

Lancer rtorrent pour tester la config

```
sudo -u rtorrent rtorrent
```

On refait un chown pour les fichiers qui viennent d'être créés dans .session

```
chown -R rtorrent:www-data /data/rtorrent/
```

### Configuration d'apache :

On créer le vhost /etc/apache2/sites-available/rutorrent.conf

```
<VirtualHost 192.168.1.100:8083>

  ServerAdmin webmaster@localhost
  DocumentRoot /var/www/html

  <Directory /var/www/html/>
    Options Indexes Includes FollowSymLinks MultiViews
    AllowOverride all
    Require all granted
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

On ouvre le port dans /etc/apache2/ports.conf

```
Listen 8083
```

on charge la configuration et on redémarre apache

```
a2ensite rutorrent.conf
systemctl restart apache2
```

### Installation de ruTorrent :

on trouve la dernière version stable ici : <https://github.com/Novik/ruTorrent/releases>

```
cd /var/wwww
wget https://github.com/Novik/ruTorrent/archive/v3.10.zip
unzip v3.10.zip
mv ruTorrent-3.10/ html
rm v3.10.zip
chown -R www-data:www-data html
```

On modifie la configuration de rutorrent : conf/config.php

```
// On commente le chemin socket ip, et on file le soket unix
  //$scgi_port = 5000;
  //$scgi_host = "127.0.0.1";

  // For web->rtorrent link through unix domain socket 
  // (scgi_local in rtorrent conf file), change variables 
  // above to something like this:
  //
  $scgi_port = 0;
  $scgi_host = "unix:///data/rtorrnet/.session/rpc.socket";
```

### Améliorations :

Si on veut mettre des limites de connection, on rajoute au .rtorrent.rc

```
# Global upload and download rate in KiB.
# "0" for unlimited
throttle.global_down.max_rate.set_kb = 1000
throttle.global_up.max_rate.set_kb = 1000
```

Si on veut que rtorrent arrête le téléchargement en cas d'espace disque faible :

```
# Close torrents when disk-space is low. 
schedule2 = low_diskspace,5,60,close_low_diskspace=5000M
```

Si on veut que le plugin mediainfo fonctionne :

```
aptitude install mediainfo
```

Si on veut que le plugin spectrogram fonctionne :

```
aptitude install sox
```

Si on veut que le pluggin unpack marche :

```
aptitude install unrar
```

Pour optimiser les downloads avec peu de seed :

```
## Tracker-less torrent and UDP tracker support
## (conservative settings for 'private' trackers, change for 'public')
dht.mode.set = on
# port for DTH (default 6881)
#dht.port.set = 6881
# Use peer exchange
protocol.pex.set = yes
# DTH use udp by default
trackers.use_udp.set = yes
```
Pour avoir le pays des peers, il faut installer GeoIP :

```
aptitude install php-geoip geoip-database
systemctl restart apache2.service
systemctl restart php7.3-fpm.service
```

## Iptables :

```
iptables -A INPUT -i enp1s0 -p tcp --dport 50000 -j ACCEPT
ip6tables -A INPUT -i enp1s0 -p tcp --dport 50000 -j ACCEPT
ip6tables -A INPUT -i enp1s0 -p udp --dport 6881 -j ACCEPT
iptables -A INPUT -i enp1s0 -p udp --dport 6881 -j ACCEPT
```
