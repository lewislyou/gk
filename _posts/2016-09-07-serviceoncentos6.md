---
title: Centos6 service
---

# How to respawn a script in RHEL/Centos6 when /etc/inittab has been deprecated
     cd /etc/init
     vi scriptFileName.conf

And add this content

     start on stopped rc RUNLEVEL=[12345]
     stop on runlevel [!12345]
     respawn
     exec /you/respawned/script.sh -your -parameters

Save file and then launch this command (without .conf of file)

     start scriptFileName

for example service zebra
/etc/init/zebra.conf

     start on stopped [12345]
     stop on runlevel [!12345]
     post-stop script
          pid=$(cat /var/run/quagga/zebra.pid)
          if $(ps -ef |grep 'zebra'|grep -q $pid) ;then
              kill $pid
          fi
     end script
     respawn
     exec /usr/sbin/zebra.sh start

    start zebra

<http://serverfault.com/questions/640788/how-to-respawn-a-script-in-rhel-centos6-when-etc-inittab-has-been-deprecated>

otherwise use supervisord

{{ page.date|date_to_string }}
