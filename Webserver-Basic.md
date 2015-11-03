*Upgraden naar CustomBuild 2.0 (CB2)

Om DirectAdmin te upgraden naar custombuild 2.0 (met PHP 5.5 en MySQL 5.6 + Apache 2.4), voer je het volgende uit:

cd /usr/local/directadmin
mv custombuild custombuild_1.x
wget -O custombuild.tar.gz http://files.directadmin.com/services/custombuild/2.0/custombuild.tar.gz
tar xvzf custombuild.tar.gz
cd custombuild
./build

Om MySQL ook te upgraden naar versie 5.6, open je "options.conf" en zet je de waarde "mysql_inst" op "yes". Hier voer je uit:

./build all d
./build rewrite_confs
Het builden kan enige tijd (half uurtje) duren. Als alles klaar is met builden, herstart je apache:

OPEN BASE DIR AANPASSEN

cd /usr/local/directadmin/data/users/ictweb/httpd.conf
cd /usr/local/directadmin/data/templates/custom
virtual_host2.conf aanpassen open base dir weghalen

REDIS INSTALL

Shell

wget http://download.redis.io/releases/redis-3.0.1.tar.gz
tar xzf redis-3.0.1.tar.gz
cd redis-3.0.1
make
make test
make install
cd utils
chmod +x install_server.sh
./install_server.sh

Check Redis status/start/stop

service redis_6379 status
service redis_6379 start

Install CSF:

Installation
============
Installation is quite straightforward:

cd /usr/src
rm -fv csf.tgz
wget https://download.configserver.com/csf.tgz
tar -xzf csf.tgz
cd csf
sh install.sh

INSTALL & ENABLE DKIM > http://help.directadmin.com/item.php?id=569

1) Ensure the exim supports DKIM signging:
[root@es5 ~]# /usr/sbin/exim -bV | grep 'Support for'
Support for: crypteq IPv6 Perl OpenSSL move_frozen_messages Content_Scanning DKIM Old_Demime PRDR OCSP

where we want to see DKIM in the list.
If exim does not support DKIM, then re-compile exim.

2) Add it to exim
cd /etc
wget -O exim.dkim.conf http://files.directadmin.com/services/exim.dkim.conf

Then edit your /etc/exim.conf, and find the code:
remote_smtp:
 driver = smtp
and change it to look like
remote_smtp:
 driver = smtp
.include_if_exists /etc/exim.dkim.conf
and restart exim
/etc/init.d/exim restart



3) Turn in on in DirectAdmin.
cd /usr/local/directadmin
cp -f conf/directadmin.conf conf/directadmin.conf.backup
echo 'dkim=1' >> conf/directadmin.conf

making absolutely sure that you're using two >> characters, else you'll empty your directadmin.conf.
And confirm it's set, and restart DA:
[root@es5 directadmin]# ./directadmin c | grep dkim
dkim=1
[root@es5 directadmin]# /etc/init.d/directadmin restart
Stopping DirectAdmin:                                      [  OK  ]
Starting DirectAdmin:                                      [  OK  ]



4) At this point, any domain created after the change should have the DKIM keys created, and dns zones updated.
For existing domains, you can either enable it individually for each domain, one-by-one:
cd /usr/local/directadmin/scripts
./dkim_create.sh domain.com



5) or you can enable it for all of your domains like this:
echo "action=rewrite&value=dkim" >> /usr/local/directadmin/data/task.queue

but it may be a good idea to test it out manually with 3) first, to ensure it works correctly.

Note: the dkim_create.sh script itself doesn't touch the zone.  It uses a task.queue entry to have the dataskq add the dns entry based on the keys that the dkim_create.sh script created.  As such, it may take up to one minute for the records to be added.

Important: If any of your domains are hosted using an external DNS server that DA does not control, you MUST manually add the TXT records to the remote zones.  You can copy them over as needed.  If the DNS does not have the records, but your emails are signed, this may increase the spam score of those emails, which is the opposite of what we want.


/etc/init.d/httpd restart
Check voor errors het log:

tail -n 500 -f /var/log/httpd/error_log
Mogelijk dat je hier een foutmelding over APC en/of Memcached ziet. Schakel deze dan uit in de php.ini (APC is in PHP 5.5 vervangen door OPcache).

nano /usr/local/lib/php.ini
Onderaan het document vind je de configuratie voor APC en Memcached. Verwijder deze of zet hier een ; voor. Herstart HTTPD hierna nogmaals. De tabellen van MySQL dienen ook nog geupgrade te worden. Om dit te doen voer je "mysql_upgrade" uit. Hiervoor heb je de root gebruiker nodig van MySQL. Het root wachtwoord staat doorgaans in het bestand setup.txt van DirectAdmin. Bekijk deze met:

cat /usr/local/directadmin/scripts/setup.txt
Zoek de regel "mysql=" (vaak regel 3). Hierachter staat het root wachtwoord. Je kunt nu de upgrade van de MySQL-tabellen uitvoeren:

mysql_upgrade -u root -p
Herstart MySQL en check daarna voor de versie + het error log:

/etc/init.d/mysqld restart
mysql -u root -p -e 'SHOW VARIABLES LIKE "%version%"'
tail -100 /var/log/mysql/mysql.error.log
Mocht MySQL niet willen starten, controleer dan goed het mysql.error.log. Waarschijnlijk staat hier een melding over de "table_cache". Deze is niet (goed) compatible met MySQL 5.6. Mocht je deze melding vinden, bewerk dan de config:

nano /etc/my.cnf
en comment de "table_cache" regel door er een # voor te zetten. Doe een laatste check middels "htop" om te kijken of alle processen draaien. Sowieso moeten draaien: MySQL en HTTPD. Om alle versies te controleren, kun je onderstaande uitvoeren (+ voorbeeld hieronder): Apache:

root@alpha03:~$ httpd -v
Server version: Apache/2.4.10 (Unix)
Server built: Dec 10 2014 09:05:03
PHP versie:

root@alpha03:~$ php -v
PHP 5.5.19 (cli) (built: Dec 10 2014 09:20:08) 
Copyright (c) 1997-2014 The PHP Group
Zend Engine v2.5.0, Copyright (c) 1998-2014 Zend Technologies
with the ionCube PHP Loader v4.7.2, Copyright (c) 2002-2014, by ionCube Ltd., and
with Zend OPcache v7.0.4-dev, Copyright (c) 1999-2014, by Zend Technologies
MySQL:

mysql -u root -p -e 'SHOW VARIABLES LIKE "%version%"'
Enter password: 
+-------------------------+------------------------------+
| Variable_name | Value |
+-------------------------+------------------------------+
| innodb_version | 5.6.22 |
| protocol_version | 10 |
| slave_type_conversions | |
| version | 5.6.22 |
| version_comment | MySQL Community Server (GPL) |
| version_compile_machine | x86_64 |
| version_compile_os | Linux |
+-------------------------+------------------------------+
