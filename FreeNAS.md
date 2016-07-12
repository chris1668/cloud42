FreeNAS
=======

I started using FreeNAS in August 2013. It is fantastic piece of software and I have been really impressed by the upgrades just in the few months I've been using it. It looks like they recently went to a plugin system as of version 9 to make installing software easier for end users. I've ran into several issues related to plugins and user + group permissions so I decided to just use the available FreeBSD port system. After fiddling for a few days (now turned into months) I believe I have created something helpful for the community and anyone interested in picking up the port system. The sandbox nature of FreeNAS's jail system is especially helpful for playing around without having any consequence on your core system.

Here are straight-forward instructions to setting up a bunch of different software on FreeNAS. If you make a terrible error, just throw up another plugin sandbox and repeat.

ToC
--------

+ [Dependencies](#dependencies)
+ [Couch Potato](#couch_potato)
+ [SickRage](#sickrage)
+ [Sabnzbd](#sabnzbd)
+ [Madsonic](#madsonic)
+ [Headphones](#headphones)
+ [Plex](#plex)
+ [Deluge](#deluge)
+ [Nginx Webserver](#webserver)
+ [Calibre](#calibre)
+ [TT-RSS](#tt-rss)
+ [OwnCloud](#owncloud)


Notes
-----
Gist is awesome, and currently this write-up is written in Markdown. Unfortunately that means you'll have to replace a bunch of references to:

+ **tetra**: My zpool
+ **192.168.1.2**: My static FreeNAS ip
+ **192.168.1.3**: My static media jail ip
+ **192.168.1.4**: My static backup jail ip

Along with that be sure to keep paths consistent between your builds. It is easy to forget if you SSH'd into FreeNAS or into the jail.

Additionally, **do not copy/paste entire chunks of commands**. I often skip over different yes/no options for brevity in this guide. Read what the prompt says and feel free to drop a comment if answers seem ambiguous.


Setting up the jail
-------------------
**Create a jail using the FreeNAS web UI**
```
Jail name: media_jail
IPv4 address: 192.168.1.3/24
autostart: checked
type: pluginjail
VIMAGE: unchecked
vanilla: checked
sysctls: allow.raw_sockets=true,allow.sysvipc=true
```

**Ports and dependencies<a name="#dependencies">**
```
ssh root@192.168.1.2
jls
jexec 17 tcsh
passwd
portsnap fetch extract
portsnap fetch update
sysrc sshd_enable=YES
sysrc ftpd_enable=YES
cd /usr/ports/ports-mgmt/pkg/ && make deinstall
cd /usr/ports/ports-mgmt/pkg/ && make install clean
pkg2ng
cd /usr/ports/ports-mgmt/portmaster && make config-recursive install clean
cd /usr/ports/devel/git && make config-recursive install clean
cd /usr/ports/devel/py-cheetah && make config-recursive install clean
```

**Create a 'media' user and create media directory**
```
mkdir /mnt/media
adduser
Username   : media
Password   : <blank>
Full Name  : Media
Uid        : 1001
Class      : 
Groups     : media 
Home       : /home/media
Home Mode  : 
Shell      : /bin/tcsh
Locked     : no
id media
```

Create this user in your FreeNAS with the same uid and gid (typically 1001 if you haven't made a custom account yet).

Add mounts for media + crashplan backups inside the jail (/mnt/<ZFS>/media to /mnt/media).

Software
--------
<a name="sabnzbd"></a>
**Sabnzbd**
```
cd /usr/ports/news/sabnzbdplus && make config-recursive install clean
chown -R media:media sabnzbd
sysrc sabnzbd_enable=YES
sysrc sabnzbd_user=media
sysrc sabnzbd_group=media
```

<a name="sickrage"></a>
**SickRage**
```
cd /usr/local && git clone git://github.com/SiCKRAGETV/SickRage.git sickrage
chown -R media:media sickrage
cp /usr/local/sickrage/runscripts/init.freebsd /usr/local/etc/rc.d/sickrage
sysrc sickrage_enable=YES
sysrc sickrage_user=media
sysrc sickrage_group=media
```

<a name="couch_potato"></a>
**Couch Potato**
```
cd /usr/local && git clone git://github.com/RuudBurger/CouchPotatoServer.git couchpotato
chown -R media:media couchpotato
cp /usr/local/couchpotato/init/freebsd /usr/local/etc/rc.d/couchpotato
chmod +x /usr/local/etc/rc.d/couchpotato
sysrc couchpotato_enable=YES
sysrc couchpotato_user=media
sysrc couchpotat_dir=/usr/local/couchpotato
```

<a name="headphones"></a>
**Headphones**
```
cd /usr/local && git clone git://github.com/rembo10/headphones.git
chown -R media:media headphones
cp /usr/local/headphones/init-scripts/init.freebsd /usr/local/etc/rc.d/headphones
chmod +x /usr/local/etc/rc.d/headphones
sysrc headphones_enable=YES
sysrc headphones_user=media
```

<a name="plex"></a>
**Plex**
```
cd /usr/ports/multimedia/plexmediaserver && make config-recursive install clean
cd /usr/local && mkdir plexdata
chown -R media:media plexdata
sysrc plexmediaserver_enable=YES
sysrc plexmediaserver_user=media
sysrc plexmediaserver_group=media
```

<a name="madsonic"></a>
**Madsonic**

Discovered madsonic only recently (quite a surprise!). Looks like a great fork of subsonic that includes a number of features for free compared to subsonic. The port depends on jetty server and other stuff. I prefer to use openjdk and a standalone install as done below. Use the madsonic start script also attached to the gist.
```
cd /usr/ports/java/openjdk8-jre && make config-recursive install clean
mkdir -p /usr/local/madsonic && cd /usr/local/madsonic
wget -Omadsonic.tar.gz http://madsonic.org/download/6.0/20160122_madsonic-6.0.7960-standalone.zip
tar -zxvf madsonic.tar.gz
rm madsonic.tar.gz
chown -R media:media /usr/local/madsonic

vi /usr/local/etc/rc.d/madsonic
# Use the file "madsonic" attached below
chmod a+x /usr/local/etc/rc.d/madsonic

sysrc madsonic_enable=YES
sysrc madsonic_user=media
sysrc madsonic_bin=/usr/local/madsonic/madsonic.sh
sysrc madsonic_podcast_folder=/mnt/media/music/podcasts
sysrc madsonic_playlist_folder=/mnt/media/music/playlists
```

Here is the [madsonic launch script](#file-madsonic) to use.

<a name="deluge"></a>
**Deluge**
Uncheck option for  GTK during configuration.
```
cd /usr/ports/net-p2p/deluge && make WITHOUT_X11=yes config-recursive install clean
mkdir -p /usr/local/deluge
chown -R media:media /usr/local/deluge
sysrc deluged_enable=YES
sysrc deluged_user=media
sysrc deluged_confdir=/usr/local/deluge
sysrc deluge_web_enable=YES
sysrc deluge_web_user=media
sysrc deluge_web_confdir=/usr/local/deluge
```

<a name="calibre"></a>
**Calibre**

```
cd /usr/ports/deskutils/calibre && make config-recursive && make install clean
sysrc calibre_enable=YES
sysrc calibre_port=8082
sysrc calibre_user=media
sysrc calibre_library=/mnt/media/books
```

<a name="webserver"></a>
**Nginx webserver + PHP + MYSQL**

For PHP5-extensions, include: bz2 ctype curl ftp dom exif fileinfo gd gmp iconv json ldap mbstring mcrypt mysql mysqli openssl pdo_mysql pdo_pgsql pdo_sqlite pgsql xsl zip zlib

For PHP5, include: FPM
```
cd /usr/ports/www/nginx && make config-recursive install clean
cd /usr/ports/databases/mysql56-server && make config-recursive install clean
cd /usr/ports/databases/postgresql94-server && make config-recursive install clean
cd /usr/ports/lang/php56-extensions && make config-recursive install clean
cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
cd /usr/local/etc/nginx && cp nginx.conf-dist nginx.conf && cp mim.types-dist mime.types
sysrc nginx_enable=YES
sysrc php_fpm_enable=YES
sysrc mysql_enable=YES
sysrc postgresql_enable=YES
service postgresql initdb
service postgresql start
service mysql-server start
mysql_secure_installation
su pgsql
passwd
```
Modify nginx.conf similar to the attached [nginx.conf](#file-nginx-conf) file. Make sure to pay close attention to where "root" is and "location ~ \.php$". You can overwrite the file with ":wq!"

Create a similar [index.html](#file-index-html) as below in folder /usr/local/www.
<a name="owncloud"></a>
**OwnCloud**
```
cd /usr/local/www/owncloud && make config-recursive install clean
su pgsql
createdb ocdb
psql -s ocdb
create user <user> password <password>
GRAND ALL PRIVELEGES ON DATABASE ocdb TO <user>
```

<a name="tt-rss"></a>
**Tiny Tiny RSS**

Helpful link: http://tt-rss.org/forum/viewtopic.php?f=16&t=911
```
cd /usr/ports/www/tt-rss && make config-recursive && make install clean
sysrc ttrssd_enable=YES
mysql -u root -p
CREATE DATABASE ttrssdb;
GRANT ALL ON ttrssdb.* TO ttrssuser IDENTIFIED BY "pick some random long password with lots of words BLAMMY";
quit;
chown -R www:www /usr/local/www/tt-rss
rm /usr/local/www/tt-rss/config.php
```

Testing
-------
```
service sshd start
service couchpotato start
service sickrage start
service headphones start
service sabnzbd start
service plexmediaserver start
service deluge start
service nginx start
service php-fpm start
service postgres start
service calibre start
service madsonic start
```

Now check to make sure everything is running fine (<a href="http://192.168.1.3">192.168.1.3</a>). Then shut down the plugin server and start it up again. Everything should still be working fine.


Backups
-------

**Important files**
Sickbeard replacement: sickbeard.db and config.ini
Sabnzbd replacement: sabnzbd.ini
Couch potato replacement:
Plex: not worth it

**Moving over databases and config files from plugins**
Log into FreeNAS
```
cp /mnt/tetra/plugins_1/usr/pbi/sickbeard-amd64/data/config.ini /mnt/tetra/media_jail/usr/local/sickbeard
cp /mnt/tetra/plugins_1/usr/pbi/sickbeard-amd64/data/sickbeard.db /mnt/tetra/media_jail/usr/local/sickbeard
cp /mnt/tetra/plugins_1/usr/pbi/sabnzbdplus-amd64/sabnzbd/sabnzbd.ini /mnt/tetra/media_jail/usr/local/sabnzbd
cp /mnt/tetra/plugins_1/usr/pbi/headphones-amd64/data/config.ini /mnt/tetra/media_jail/usr/local/headphones/
cp /mnt/tetra/plugins_1/usr/pbi/headphones-amd64/data/headphones.db /mnt/tetra/media_jail/usr/local/headphones/
cp /mnt/tetra/plugins_1/usr/local/CouchPotatoServer/data/settings.conf /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data
cp /mnt/tetra/plugins_1/usr/local/CouchPotatoServer/data/couchpotato.db /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data
```

Make sure your settings move across the boundary. Daemons might not start up if ip's, filepaths, etc. are different.

**Crashplan**

*Create a jail using the FreeNAS web UI*
```
Jail name: backup_jail
IPv4 address: 192.168.1./24
autostart: checked
type: portjail
VIMAGE: unchecked
vanilla: checked
```

Now add the directories you want to backup and where the backups should go.

**Ports and dependencies**
```
ssh root@192.168.1.2
jls
jexec 5 tcsh
passwd
portsnap fetch && portsnap extract && portsnap update
sysrc sshd_enable=YES
vi /etc/ssh/sshd_config
# add the following to the end of the file
Match User backup
    AllowTcpForwarding yes

adduser # backup with Uid 1002
```
Download the java runtime in order to please the CrashPlan overlords (or alternatively, modify the crashplan port scripts):
http://www.oracle.com/technetwork/java/javase/downloads/index.html
Scroll down to Java SE 7u71/72 and click JRE Download (third button). Accept the license and download jre-7u71-linux-x64.tar.gz. Then copy it over:

```
scp ~/Downloads/jre-7u71-linux-i586.tar.gz root@192.168.1.2:/mnt/tetra/backup_jail/usr/ports/distfiles/
```
This is just to appease the makefile, and instead we will use OpenJDK's JRE anyways. The following takes quite a while to install.

Install java and crashplan
```
cd /usr/ports/java/openjdk8-jre/ && make config-recursive && make install clean
cd /usr/ports/sysutils/linux-crashplan/ && make config-recursive && make install clean
sysrc crashplan_enable=YES
```
Now change the default Java binary path:

```
vi /usr/local/crashplan/install.vars
  JAVACOMMON=/usr/local/bin/java
```

Restart the jail. Now follow [this guide](http://support.code42.com/CrashPlan/Latest/Configuring/Configuring_A_Headless_Client) to modify your current crashplan install to work on the remote machine. You will need to create a port bridge over ssh, which you can do with the following command (before starting up crashplan locally):
```
ssh -L 4200:localhost:4243 backup@192.168.1.4
```

Helpful codes
-------------
Mounting USB drive:
```
kldload fuse
mkdir /mnt/usb
ntfs-3g /dev/da1s1 /mnt/usb
ntfs-3g -o permissions /dev/da1s1 /mnt/usb
```

Upgrading
=========
Upgrading can be a royal pain... but fear not. Typically you can just run a portmaster -ad, and if it says "conflict... blah blah" just run "pkg delete -f <stupid old package>" then re-run the portmaster command. Eventually everything should be updated!
```
less /usr/ports/UPDATING
portsnap fetch update
cd /usrports/ports-mgmt/pkg && make install clean
cd /usr/ports/ports-mgmt/portmaster && make install clean
portmaster -Rafd
```

Rsync files
```
&rsync --progress --stats --recursive --times --perms --links  --dry-run /mnt/tetra /mnt/usb/tetra
nohup foo &

rsync -az -H --delete --numeric-ids --stats --progress -e ssh root@192.168.1.2:/mnt/tetra/family /media/jacob/usb/tetra
rsync -az -H --delete --numeric-ids --stats --progress -e ssh root@192.168.1.2:/mnt/tetra/media_jail/usr/local/sickbeard/data/config.ini /media/jacob/usb/tetra/backup

cp 
```

Copy server and daemon config files and databases
```
mkdir /mnt/tetra/backup/server_configs
cd /mnt/tetra/backup/server_configs
rsync -aqz /mnt/tetra/media_jail/usr/local/sickbeard/config.ini /mnt/tetra/media_jail/usr/local/sickbeard/sickbeard.db sickbeard/
rsync -aqz /mnt/tetra/media_jail/usr/local/sabnzbd/sabnzbd.ini sabnzbd/
rsync -aqz /mnt/tetra/media_jail/usr/local/headphones/config.ini /mnt/tetra/media_jail/usr/local/headphones/headphones.db headphones/
rsync -aqz /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/settings.conf /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/couchpotato.db couchpotato/
rsync -aqz /mnt/tetra/media_jail/usr/local/etc/nginx/nginx.conf /mnt/tetra/media_jail/usr/local/www/home/index.html nginx/
```

```
cd /mnt/tetra/backup/server_configs
rsync -aqz sabnzbd/sabnzbd.ini /mnt/tetra/media_jail/usr/local/sabnzbd/
rsync -aqz sickbeard/config.ini sickbeard/sickbeard.db /mnt/tetra/media_jail/usr/local/sickbeard/
rsync -aqz headphones/config.ini headphones/headphones.db /mnt/tetra/media_jail/usr/local/headphones/
rsync -aqz couchpotato/settings.conf couchpotato/couchpotato.db /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/

cd /usr/local && chmod -R media:media sabnzbd sickbeard headphones CouchPotatoServer
```
