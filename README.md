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

## airtable.env
```
AIRTABLE_API_KEY=****
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
