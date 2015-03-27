FreeNAS
=======

I started using FreeNAS in August 2013. It is fantastic piece of software and I have been really impressed by the upgrades just in the few months I've been using it. It looks like they recently went to a plugin system as of version 9 to make installing software easier for end users. I've ran into several issues related to plugins and user + group permissions so I decided to just use the available FreeBSD port system. After fiddling for a few days (now turned into months) I believe I have created something helpful for the community and anyone interested in picking up the port system. The sandbox nature of FreeNAS's jail system is especially helpful for playing around without having any consequence on your core system.

Here are straight-forward instructions to setting up a bunch of different software on FreeNAS. If you make a terrible error, just throw up another plugin sandbox and repeat.

ToC
--------

+ [Dependencies](#dependencies)
+ [Couch Potato](#couch_potato)
+ [Sick Beard](#sick_beard)
+ [Sabnzbd](#sabnzbd)
+ [Subsonic](#subsonic)
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

Additionally, **do not copy/paste entire chunks of commands**. I often skip over different yes/no options for brevvity in this guide. Read what the prompt says and feel free to drop a comment if answers seem ambiguous.


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
```

**Ports and dependencies<a name="#dependencies">**
```
ssh root@192.168.1.2
jls
jexec 17 tcsh
passwd
portsnap fetch update
portsnap extract
echo 'sshd_enable="YES"' >> /etc/rc.conf
echo 'ftpd_enable="YES"' >> /etc/rc.conf
cd /usr/ports/devel/git && make config-recursive && make install clean
cd /usr/ports/devel/py-cheetah && make install clean
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
cd /usr/ports/news/sabnzbdplus && make config-recursive && make install clean
cd /usr/local && mkdir sabnzbd
chown -R media:media sabnzbd
echo 'sabnzbd_enable="YES"' >> /etc/rc.conf
echo 'sabnzbd_user="media"' >> /etc/rc.conf
echo 'sabnzbd_group="media"' >> /etc/rc.conf
```

<a name="sick_beard"></a>
**Sick Beard**
```
cd /usr/local && git clone git://github.com/SiCKRAGETV/SickRage.git sickbeard
chown -R media:media sickbeard
cp /usr/local/sickrage/init.freebsd /usr/local/etc/rc.d/sickrage
echo 'sickbeard_enable="YES"' >> /etc/rc.conf
echo 'sickbeard_user="media"' >> /etc/rc.conf
```

<a name="couch_potato"></a>
**Couch Potato**
```
cd /usr/local && git clone git://github.com/RuudBurger/CouchPotatoServer.git
chown -R media:media CouchPotatoServer
cp /usr/local/CouchPotatoServer/init/freebsd /usr/local/etc/rc.d/couchpotato
chmod +x /usr/local/etc/rc.d/couchpotato
echo 'couchpotato_enable="YES"' >> /etc/rc.conf
echo 'couchpotato_user="media"' >> /etc/rc.conf
```

<a name="headphones"></a>
**Headphones**
```
cd /usr/local && git clone git://github.com/rembo10/headphones.git
chown -R media headphones && chgrp -R media headphones
cp /usr/local/headphones/init-alt.freebsd /usr/local/etc/rc.d/headphones
chmod +x /usr/local/etc/rc.d/headphones
echo 'headphones_enable="YES"' >> /etc/rc.conf
echo 'headphones_user="media"' >> /etc/rc.conf
```

<a name="plex"></a>
**Plex**
```
cd /usr/ports/multimedia/plexmediaserver && make install clean
cd /usr/local && mkdir plexdata
chown -R media:media plexdata
echo 'plexmediaserver_enable="YES"' >> /etc/rc.conf
echo 'plexmediaserver_user="media"' >> /etc/rc.conf
echo 'plexmediaserver_group="media"' >> /etc/rc.conf
```

<a name="subsonic"></a>
**Subsonic**
```
cd /usr/ports/java/openjdk6-jre && make config-recursive && make install
mkdir -p /usr/local/subsonic/standalone && cd /usr/local/subsonic/standalone
wget -Osubsonic.tar.gz http://downloads.sourceforge.net/project/subsonic/subsonic/4.9/subsonic-4.9-standalone.tar.gz
tar -zxvf subsonic.tar.gz
rm subsonic.tar.gz
chown -R media:media /usr/local/subsonic

vi /usr/local/etc/rc.d/subsonic
# Use the file "subsonic" attached below
chmod a+x /usr/local/etc/rc.d/subsonic

echo 'subsonic_enable="YES"' >> /etc/rc.conf
echo 'subsonic_user="media"' >> /etc/rc.conf
echo 'subsonic_bin="/usr/local/subsonic/standalone/subsonic.sh"' >> /etc/rc.conf
echo 'subsonic_podcast_folder="/mnt/media/music/podcasts"' >> /etc/rc.conf
echo 'subsonic_playlist_folder"/mnt/media/music/playlists"' >> /etc/rc.conf
```

Here is the [subsonic launch script](#file-subsonic) to use.

<a name="deluge"></a>
**Deluge**
Uncheck option for  GTK during configuration.
```
cd /usr/ports/net-p2p/deluge && make WITHOUT_X11=yes config-recursive && make install clean
mkdir -p /usr/local/deluge
chown -R media:media /usr/local/deluge
echo 'deluged_enable="YES"' >> /etc/rc.conf
echo 'deluged_user="media"' >> /etc/rc.conf
echo 'deluged_confdir="/usr/local/deluge"' >> /etc/rc.conf
echo 'deluge_web_enable="YES"' >> /etc/rc.conf
echo 'deluge_web_user="media"' >> /etc/rc.conf
echo 'deluge_web_confdir="/usr/local/deluge"' >> /etc/rc.conf
```

<a name="calibre"></a>
**Calibre**

Version is a bit old in the repo. Might consider trying to install from source in the future.
```
cd /usr/ports/deskutils/calibre && make config-recursive && make install clean
echo 'calibre_enable="YES"' >> /etc/rc.conf
echo 'calibre_port="8082"' >> /etc/rc.conf
echo 'calibre_user="media"' >> /etc/rc.conf
echo 'calibre_library="/mnt/media/books"' >> /etc/rc.conf
```

<a name="owncloud"></a>
**OwnCloud**
First install the php packages below from webserver. Then build owncloud.
```
cd /usr/ports/textproc/libxml2/ && make config-recursive && make install clean
cd /usr/ports/textproc/php5-xsl/ && make config-recursive && make install clean
cd /usr/ports/www/owncloud && make config-recursive && make install clean
```

<a name="webserver"></a>
**Nginx webserver + PHP + MYSQL**

For PHP5-extensions, include: bz2 curl fileinfo gd mbstring mcrypt mysqli openssl pdo_mysql pdo_pgsql pgsql xsl zip

For PHP5, include: FPM
```
cd /usr/ports/www/nginx && make config-recursive && make install clean
cd /usr/ports/lang/php5-extensions && make config-recursive && make install clean
cd /usr/ports/databases/mysql56-server && make config-recursive && make install clean
cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
cd /usr/local/etc/nginx && cp nginx.conf-dist nginx.conf && cp mim.types-dist mime.types
echo 'nginx_enable="YES"' >> /etc/rc.conf
echo 'php_fpm_enable="YES"' >> /etc/rc.conf
echo 'mysql_enable="YES"' >> /etc/rc.conf
/usr/local/etc/rc.d/mysql-server start
mysql_secure_installation
```
Modify nginx.conf similar to the attached [nginx.conf](#file-nginx-conf) file. Make sure to pay close attention to where "root" is and "location ~ \.php$". You can overwrite the file with ":wq!"

Create a similar [index.html](#file-index-html) as below in folder /usr/local/www.

<a name="tt-rss"></a>
**Tiny Tiny RSS**

Helpful link: http://tt-rss.org/forum/viewtopic.php?f=16&t=911
```
cd /usr/ports/www/tt-rss && make config-recursive && make install clean
echo 'ttrssd_enable="YES"' >> /etc/rc.conf
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
/usr/local/etc/rc.d/couchpotato start
/usr/local/etc/rc.d/sickrage start
/usr/local/etc/rc.d/headphones start
/usr/local/etc/rc.d/sabnzbd start
/usr/local/etc/rc.d/plexmediaserver start
/usr/local/etc/rc.d/deluge start
/usr/local/etc/rc.d/nginx start
/usr/local/etc/rc.d/php-fpm start
/usr/local/etc/rc.d/calibre start
/usr/local/etc/rc.d/subsonic start
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
echo 'sshd_enable="YES"' >> /etc/rc.conf
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
echo 'crashplan_enable="YES"' >> /etc/rc.conf
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
```
portsnap fetch update
cd /usrports/ports-mgmt/pkg && make install clean
cd /usr/ports/ports-mgmt/portmaster && make install clean
portmaster -Raf
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