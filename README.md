# Thermometre de Piscine

Linux raspberrypi 4.19.118-v7l+ #1311 SMP Mon Apr 27 14:26:42 BST 2020 armv7l GNU/Linux

```
wget -O- http://www.piduino.org/piduino-key.asc | sudo apt-key add -
echo 'deb http://raspbian.piduino.org stretch piduino' | sudo tee /etc/apt/sources.list.d/piduino.list
sudo apt update
sudo apt install mbpoll
```

## crontab
```
SHELL=/bin/bash
* * * * * source ~/airtable.env; ~/recordTemperature.sh >recordTemperature.log 2>&1; ~/deleteOldRecord.sh >deleteOldRecord.log 2>&1
```
