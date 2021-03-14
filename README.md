# Thermometre de Piscine

## Raspberry PI model 4, 2GB 
Linux raspberrypi 4.19.118-v7l+ #1311 SMP Mon Apr 27 14:26:42 BST 2020 armv7l GNU/Linux

## Raspberry PI 0W, 512MB
Linux temp-piscine 5.4.83+ #1379 Mon Dec 14 13:06:05 GMT 2020 armv6l GNU/Linux

Applications à installer
```
wget -O- http://www.piduino.org/piduino-key.asc | sudo apt-key add -
echo 'deb http://raspbian.piduino.org stretch piduino' | sudo tee /etc/apt/sources.list.d/piduino.list
sudo apt update
sudo apt install -y mbpoll
sudo apt-get install -y jq
sudo apt-get install -y smbclient
sudo apt-get install -y samba samba-common-bin
sudo timedatectl set-timezone America/Toronto
mkdir /home/pi/Shared_Piscine
```

## /boot/config.txt
```
# Bunch of commented lines omitted

dtparam=i2c_arm=off
dtparam=audio=on
[pi4]
# Enable DRM VC4 V3D driver on top of the dispmanx display stack
dtoverlay=vc4-fkms-v3d
max_framebuffers=2

[all]
#dtoverlay=vc4-fkms-v3d
enable_uart=1

#Using the PL011 UART por
dtoverlay=pi3-miniuart-bt
```

## /etc/samba/smb.conf
```
[Piscine]
    path = /home/pi/Shared_Piscine
    read only = no
    public = yes
    writable = yes

```
Restart samba after updating file
```
sudo systemctl restart smbd
```

## recordTemperature.sh
```
TEMPERATURE_LINE=$(mbpoll  -m rtu  -b 9600 -d 8 -s 1 -P none -a 1   -t 4 /dev/ttyAMA0 -1|grep '^\[1\]'|cut -f 2)
TEMPERATURE=$(echo ${TEMPERATURE_LINE} | cut -f 2)
if [[ ${TEMPERATURE} =~ .*\(.*\) ]]; then
        TEMPERATURE=$(echo ${TEMPERATURE_LINE} | cut -d '(' -f 2|cut -d ')' -f 1)
fi

source ~/airtable.env
if [ -z "${AIRTABLE_API_KEY}" ]; then
        echo "Temperature: ${TEMPERATURE}"
        echo "Airtable key not defined"
else

        tmpfile=$(mktemp airtable.XXXXXX)
        exec 3>"$tmpfile"
        curl -v -X POST https://api.airtable.com/v0/appLbCkfKBUPSXaQ8/Data   -H "Authorization: Bearer ${AIRTABLE_API_KEY}"   -H "Content-Type: application/json"   --data '{
  "records": [
    {
      "fields": {"Id":"Piscine Kender","Temperature":'$TEMPERATURE'}
    }  ]
}' >&3
        cat "$tmpfile"
        ID=$(cat "$tmpfile"|tail -n 1|jq -M '.records[0].fields.Id')
        TIMESTAMP=$(cat "$tmpfile"|tail -n 1|jq -M '.records[0].fields.Timestamp')
        rm "$tmpfile"
        echo $ID, $TIMESTAMP, $TEMPERATURE >> piscine.csv
        cp piscine.csv Shared_Piscine
fi
```

## airtable.env
```
AIRTABLE_API_KEY=****
```

## crontab
```
SHELL=/bin/bash
* * * * * source ~/airtable.env; ~/recordTemperature.sh >recordTemperature.log 2>&1; ~/deleteOldRecord.sh >deleteOldRecord.log 2>&1
```

## To collect stats with prometheus
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-armv7.tar.gz
mkdir ~/node_exporter
cd ~/node_exporter
tar -xzvf ~/node_exporter-1.1.2.linux-armv7.tar.gz
scp pi@10.1.1.XXX:/etc/systemd/system/node_exporter.service .
# vi node_exporter.service
# Update the version number as necessary
sudo mv node_exporter.service /etc/systemd/system/node_exporter.service
sudo systemctl enable node_exporter.service
sudo systemctl start node_exporter.service
sudo systemctl status node_exporter.service
```

## On the prometheus server:
### /home/pi/prometheus/prometheus.yml
```
    - targets: ['10.1.1.XXX:9100']
```
