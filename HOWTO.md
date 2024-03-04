

Install weewx v5 via pip on a clean debian(ish) system.


```
echo ".....installing packages......"
sudo apt-get install -y vim python3-pip python3-venv nginx rsyslog

echo ".....installing weewx......"

cd
python3 -m venv weewx-venv
source ~/weewx-venv/bin/activate
pip3 install weewx wheel

echo ".....configuring weewx......"
weectl station create --no-prompt
sed -i -e s:debug\ =\ 0:debug\ =\ 1: ~/weewx-data/weewx.conf

echo ".....configuring rsyslog......"
sudo cp ~/weewx-data/util/rsyslog.d/weewx.conf /etc/rsyslog.d/weewx.conf
sudo systemctl restart rsyslog

echo ".....configuring logrotate...."
sudo cp ~/weewx-data/util/logrotate.d/weewx /etc/logrotate.d/weewx

echo ".....configuring systemd...."
sudo sh ~/weewx-data/scripts/setup-daemon.sh

echo ".....starting weewx...."
sudo systemctl start weewx

echo "....done - see /var/log/weewx/weewx.log....."
```

At this point, you should have weewx running in simulator mode.

```

(weewx-venv) vince@nuc:~$ systemctl status weewx
● weewx.service - WeeWX weather system
     Loaded: loaded (/etc/systemd/system/weewx.service; enabled; preset: enabled)
     Active: active (running) since Sun 2024-03-03 17:44:24 PST; 1min 16s ago
       Docs: https://weewx.com/docs
   Main PID: 15353 (python3)
      Tasks: 1 (limit: 18995)
     Memory: 30.8M (peak: 31.0M)
        CPU: 1.782s
     CGroup: /system.slice/weewx.service
             └─15353 /home/vince/weewx-venv/bin/python3 /home/vince/weewx-venv/lib/python3.12/site-packages/weewxd.py /home/vince/weewx-data/weewx.conf

Mar 03 17:45:14 nuc weewxd[15353]: DEBUG weewx.manager: Daily summary version is 4.0
Mar 03 17:45:15 nuc weewxd[15353]: INFO weewx.cheetahgenerator: Generated 8 files for report SeasonsReport in 0.63 seconds
Mar 03 17:45:15 nuc weewxd[15353]: DEBUG weewx.manager: Daily summary version is 4.0
Mar 03 17:45:16 nuc weewxd[15353]: INFO weewx.imagegenerator: Generated 56 images for report SeasonsReport in 0.81 seconds
Mar 03 17:45:16 nuc weewxd[15353]: INFO weewx.reportengine: Copied 5 files to /home/vince/weewx-data/public_html
Mar 03 17:45:16 nuc weewxd[15353]: DEBUG weewx.reportengine: Report 'SmartphoneReport' not enabled. Skipping.
Mar 03 17:45:16 nuc weewxd[15353]: DEBUG weewx.reportengine: Report 'MobileReport' not enabled. Skipping.
Mar 03 17:45:16 nuc weewxd[15353]: DEBUG weewx.reportengine: Report 'StandardReport' not enabled. Skipping.
Mar 03 17:45:16 nuc weewxd[15353]: DEBUG weewx.reportengine: Report 'FTP' not enabled. Skipping.
Mar 03 17:45:16 nuc weewxd[15353]: DEBUG weewx.reportengine: Report 'RSYNC' not enabled. Skipping.

```

Lets stop it and install the rtl-sdr package and weewx driver via the extension installer.

```
sudo systemctl stop weewx
rm -r ~/weewx-data/archive/* ~/weewx-data/public_html/*

sudo apt-get update
sudo apt-get install -y rtl-sdr rtl0433

source ~/weewx-venv/bin/activate
weectl extension install -y https://github.com/matthewwall/weewx-sdr/archive/master.zip
weectl extension list
```

And verify the driver is installed....

```
(weewx-venv) vince@nuc:~$ weectl extension list
Using configuration file /home/vince/weewx-data/weewx.conf
Extension Name    Version   Description
sdr               0.87      Capture data from rtl_433

```

So lets switch to it.  


```
weewx-venv) vince@nuc:~$ weectl station reconfigure --no-prompt --driver=user.sdr
Using configuration file /home/vince/weewx-data/weewx.conf
Processing configuration file /home/vince/weewx-data/weewx.conf
Saving configuration file /home/vince/weewx-data/weewx.conf
Saved old configuration file as /home/vince/weewx-data/weewx.conf.20240303175452
```

Next edit weewx-data/weewx.conf to add some elements to the SDR stanza.

Ignore the last two items for now.  We will later lightly patch sdr.py to make logging
a little more controllable.   Lets switch to the SDR driver and give it a try.

```
[SDR]
    driver = user.sdr
    cmd = rtl_433 -f 915M -F json -M utc

    log_unknown_sensors  = True
    log_unmapped_sensors = True

    log_packets            = True   # log mapped packets if debug > 0
    log_duplicate_readings = True   # suppress logging duplicate packets
```

At this point, we're set to run the driver.  It is going to complain a lot due to having nothing mapped yet.  That's ok.

```
weewxd
```

Let that run for a couple minutes.  It'll look like it's hanging and doing noting, but it is running.
Open another window and do a "tail -f /var/log/weewx/weewx.log -n 200" and you should see lots of output
and the driver complaining about (a) unmapped known sensors and (b) punting trying to decipher unknown sensor models.

```
024-03-03T18:12:25.286568-08:00 nuc weewxd[17775]: INFO weewxd: Starting up weewx version 5.0.2
2024-03-03T18:12:25.286685-08:00 nuc weewxd[17775]: DEBUG weewx.engine: Station does not support reading the time
2024-03-03T18:12:25.286788-08:00 nuc weewxd[17775]: INFO weewx.engine: Using binding 'wx_binding' to database 'weewx.sdb'
2024-03-03T18:12:25.286890-08:00 nuc weewxd[17775]: INFO weewx.manager: Starting backfill of daily summaries
2024-03-03T18:12:25.286978-08:00 nuc weewxd[17775]: INFO weewx.manager: Empty database
2024-03-03T18:12:25.287045-08:00 nuc weewxd[17775]: INFO weewx.engine: Starting main packet loop.
2024-03-03T18:12:39.784865-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:12:39.785449-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:12:36", "model" : "AmbientWeather-WH31B", "id" : 202, "channel" : 5, "battery_ok" : 1, "temperature_C" : 4.200, "humidity" : 78, "data" : "b35c000200", "mic" : "CRC"}#012'
2024-03-03T18:12:39.785619-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:12:39.785820-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:12:36", "model" : "AmbientWeather-WH31B", "id" : 202, "channel" : 5, "battery_ok" : 1, "temperature_C" : 4.200, "humidity" : 78, "data" : "b35c000200", "mic" : "CRC"}#012'
2024-03-03T18:12:39.785971-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:12:39.786161-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:12:36", "model" : "AmbientWeather-WH31B", "id" : 202, "channel" : 5, "battery_ok" : 1, "temperature_C" : 4.200, "humidity" : 78, "data" : "b35c000000", "mic" : "CRC"}#012'
2024-03-03T18:12:39.786316-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:12:39.786533-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:12:36", "model" : "AmbientWeather-WH31B", "id" : 202, "channel" : 5, "battery_ok" : 1, "temperature_C" : 4.200, "humidity" : 78, "data" : "b35c000000", "mic" : "CRC"}#012'
2024-03-03T18:12:52.237444-08:00 nuc weewxd[17775]: INFO user.sdr: unmapped: {'dateTime': 1709518369, 'usUnits': 16, 'soil_moisture_percent.001260.FOWH51Packet': 39.0, 'boost.001260.FOWH51Packet': 0.0, 'soil_moisture_raw.001260.FOWH51Packet': 217.0, 'freq1.001260.FOWH51Packet': None, 'freq2.001260.FOWH51Packet': None, 'battery_ok.001260.FOWH51Packet': 0.778, 'battery_mV.001260.FOWH51Packet': 1400.0, 'snr.001260.FOWH51Packet': None, 'rssi.001260.FOWH51Packet': None, 'noise.001260.FOWH51Packet': None}
2024-03-03T18:12:52.237941-08:00 nuc weewxd[17775]: INFO user.sdr: unmapped: {'dateTime': 1709518369, 'usUnits': 16, 'soil_moisture_percent.001260.FOWH51Packet': 39.0, 'boost.001260.FOWH51Packet': 0.0, 'soil_moisture_raw.001260.FOWH51Packet': 217.0, 'freq1.001260.FOWH51Packet': None, 'freq2.001260.FOWH51Packet': None, 'battery_ok.001260.FOWH51Packet': 0.778, 'battery_mV.001260.FOWH51Packet': 1400.0, 'snr.001260.FOWH51Packet': None, 'rssi.001260.FOWH51Packet': None, 'noise.001260.FOWH51Packet': None}
2024-03-03T18:13:07.050808-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.051523-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:02", "model" : "AmbientWeather-WH31B", "id" : 55, "channel" : 4, "battery_ok" : 1, "temperature_C" : 21.200, "humidity" : 43, "data" : "ea00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.051685-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.051810-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:02", "model" : "AmbientWeather-WH31B", "id" : 55, "channel" : 4, "battery_ok" : 1, "temperature_C" : 21.200, "humidity" : 43, "data" : "ea00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.051931-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.052065-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:02", "model" : "AmbientWeather-WH31B", "id" : 55, "channel" : 4, "battery_ok" : 1, "temperature_C" : 21.200, "humidity" : 43, "data" : "ea00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.052199-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.052308-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:02", "model" : "AmbientWeather-WH31B", "id" : 55, "channel" : 4, "battery_ok" : 1, "temperature_C" : 21.200, "humidity" : 43, "data" : "ea00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.052411-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.052509-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:03", "model" : "AmbientWeather-WH31B", "id" : 153, "channel" : 2, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 41, "data" : "db00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.052624-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.052723-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:03", "model" : "AmbientWeather-WH31B", "id" : 153, "channel" : 2, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 41, "data" : "db00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.052824-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.052920-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:03", "model" : "AmbientWeather-WH31B", "id" : 153, "channel" : 2, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 41, "data" : "db00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:07.053264-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:07.053445-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:03", "model" : "AmbientWeather-WH31B", "id" : 153, "channel" : 2, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 41, "data" : "db00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:20.812871-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:20.813252-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:17", "model" : "AmbientWeather-WH31B", "id" : 145, "channel" : 1, "battery_ok" : 1, "temperature_C" : 20.400, "humidity" : 44, "data" : "4e00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:20.813498-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:20.813712-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:17", "model" : "AmbientWeather-WH31B", "id" : 145, "channel" : 1, "battery_ok" : 1, "temperature_C" : 20.400, "humidity" : 44, "data" : "4e00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:20.813933-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:20.814140-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:17", "model" : "AmbientWeather-WH31B", "id" : 145, "channel" : 1, "battery_ok" : 1, "temperature_C" : 20.400, "humidity" : 44, "data" : "4e00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:20.814347-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:20.814546-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:17", "model" : "AmbientWeather-WH31B", "id" : 145, "channel" : 1, "battery_ok" : 1, "temperature_C" : 20.400, "humidity" : 44, "data" : "4e00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:25.791797-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:25.792388-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:22", "model" : "AmbientWeather-WH31B", "id" : 196, "channel" : 3, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 42, "data" : "5a00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:25.792622-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:25.792838-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:22", "model" : "AmbientWeather-WH31B", "id" : 196, "channel" : 3, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 42, "data" : "5a00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:25.793064-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:25.793218-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:22", "model" : "AmbientWeather-WH31B", "id" : 196, "channel" : 3, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 42, "data" : "5a00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:25.793417-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:25.793532-08:00 nuc weewxd[17775]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 02:13:22", "model" : "AmbientWeather-WH31B", "id" : 196, "channel" : 3, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 42, "data" : "5a00000000", "mic" : "CRC"}#012'
2024-03-03T18:13:44.800527-08:00 nuc weewxd[17775]: DEBUG user.sdr: parse_json: unknown model AmbientWeather-WH31B
2024-03-03T18:13:44.810708-08:00 nuc weewxd[17775]: INFO weewx.engine: Main loop exiting. Shutting engine down.
2024-03-03T18:13:44.810967-08:00 nuc weewxd[17775]: INFO user.sdr: shutdown process rtl_433 -f 915M -F json -M utc
2024-03-03T18:13:44.811223-08:00 nuc weewxd[17775]: DEBUG user.sdr: close stdout
2024-03-03T18:13:44.812744-08:00 nuc weewxd[17775]: DEBUG user.sdr: close stderr
2024-03-03T18:13:44.813898-08:00 nuc weewxd[17775]: DEBUG user.sdr: shutdown stdout-thread
2024-03-03T18:13:44.814495-08:00 nuc weewxd[17775]: DEBUG user.sdr: shutdown stderr-thread
2024-03-03T18:13:44.815295-08:00 nuc weewxd[17775]: INFO user.sdr: shutdown complete
2024-03-03T18:13:44.815664-08:00 nuc weewxd[17775]: CRITICAL weewxd: Keyboard interrupt.
```

So what do we have here ?

* we have entries for a known Ecowitt WH51 soil moisture sensor
* we have entries for five unknown sensors (channels 1-5) claiming to be Ambient Weather WH31B temperature+humidity sensors
* and we have the relevant data returned for each sensor
* and it looks like the sensors report duplicate packets.  We'll deal with that later.

First lets work the soil moisture sensor. Consider the output from the logs above...

```
#  unmapped: {'dateTime': 1709518369, 'usUnits': 16, 'soil_moisture_percent.001260.FOWH51Packet': 39.0, 'boost.001260.FOWH51Packet': 0.0, 'soil_moisture_raw.001260.FOWH51Packet': 217.0, 'freq1.001260.FOWH51Packet': None, 'freq2.001260.FOWH51Packet': None, 'battery_ok.001260.FOWH51Packet': 0.778, 'battery_mV.001260.FOWH51Packet': 1400.0, 'snr.001260.FOWH51Packet': None, 'rssi.001260.FOWH51Packet': None, 'noise.001260.FOWH51Packet': None}
```

What's that mean ?   It means that we know its data elements and values and we have enough info to map it to the weewx db.  Lets map it to the soilMoist1 element.

```
    [[sensor_map]]
        soilMoist1 = soil_moisture_percent.001260.FOWH51Packet
```

At this point we can restart weewx, wait a little, stop weewx, and we should see items in the db for soilMost1.  Lets try it.

```
sudo systemctl stop weewx
sudo systemctl start weewx
sleep 400
sudo systemctl stop weewx
echo "select * from archive;" | sqlite3 ~/weewx-data/archive/weewx.sdb

```

Watch your logfile for it saving data to the db.

```
2024-03-03T18:40:52.127578-08:00 nuc weewxd[18970]: DEBUG user.sdr: packet={'soilMoist1': 39.0, 'dateTime': 1709520049, 'usUnits': 16}
2024-03-03T18:40:52.139086-08:00 nuc weewxd[18970]: INFO weewx.manager: Added record 2024-03-03 18:40:00 PST (1709520000) to database 'weewx.sdb'
2024-03-03T18:40:52.145991-08:00 nuc weewxd[18970]: INFO weewx.manager: Added record 2024-03-03 18:40:00 PST (1709520000) to daily summary in 'weewx.sdb'
2024-03-03T18:40:52.174341-08:00 nuc weewxd[18970]: DEBUG weewx.reportengine: Running reports for latest time in the database.
```

Which means you should see data in the db for that mapped sensor if we query for it now

```
echo "select datetime(dateTime,'unixepoch','localtime'),dateTime,soilMoist1 from archive;" | sqlite3 ~/weewx-data/archive/weewx.sdb

2024-03-03 18:40:00|1709520000|39.0

```

Cool.  It added a record to the db at 18:40:00 matching the syslog message saying what it claimed to do.  One sensor working.

Lets do the rest.  This is a little harder as the two temp+hum sensor types are not known to weewx so we'll have to edit sdr.py to add them.

First find something that looks close to the data you have.  We have Ecowitt sensors which we know are also known as AmbientWeather sometimes.

The data will look something like:
```
{"time" : "2024-03-04 02:41:59", "model" : "AmbientWeather-WH31B", "id" : 153, "channel" : 2, "battery_ok" : 1, "temperature_C" : 21.000, "humidity" : 41, "data" : "db00000000", "mic" : "CRC"
```

First lets copy the original sdr.py just....in....case.... we might mess it up editing below.  
```
cp ~/weewx-data/bin/user/sdr.py ~/weewx-data/bin/user/sdr.py.keepme
```

Next edit sdr.py and find the line beginning with `class AmbientWH31EPacket(Packet)` which should be at about line 1118 in the file.  To get there in vi, you would `vi +1118 sdr.py` to go right there.   Copy the whole Class block there and put another copy right below there.  There should be 28 lines copied.  In those 28 lines edit WH31E everywhere to say WH31B.  This creates an added class for this previously unknown sensor.  Save you file and start weewx again.  Wait (im)patiently and watch your logs.  You should see it stop complaining about unknown sensors and just complain that the newly added sensor type has some unmapped (yet) entries.

```
2024-03-03T18:56:47.238000-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521004, 'usUnits': 17, 'temperature.55.AmbientWH31BPacket': 21.2, 'humidity.55.AmbientWH31BPacket': 43.0, 'battery.55.AmbientWH31BPacket': 1, 'channel.55.AmbientWH31BPacket': 4, 'rssi.55.AmbientWH31BPacket': None, 'snr.55.AmbientWH31BPacket': None, 'noise.55.AmbientWH31BPacket': None}
2024-03-03T18:56:47.238409-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521004, 'usUnits': 17, 'temperature.55.AmbientWH31BPacket': 21.2, 'humidity.55.AmbientWH31BPacket': 43.0, 'battery.55.AmbientWH31BPacket': 1, 'channel.55.AmbientWH31BPacket': 4, 'rssi.55.AmbientWH31BPacket': None, 'snr.55.AmbientWH31BPacket': None, 'noise.55.AmbientWH31BPacket': None}]
2024-03-03T18:57:04.803231-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521020, 'usUnits': 17, 'temperature.145.AmbientWH31BPacket': 20.3, 'humidity.145.AmbientWH31BPacket': 43.0, 'battery.145.AmbientWH31BPacket': 1, 'channel.145.AmbientWH31BPacket': 1, 'rssi.145.AmbientWH31BPacket': None, 'snr.145.AmbientWH31BPacket': None, 'noise.145.AmbientWH31BPacket': None}
2024-03-03T18:57:04.804158-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521020, 'usUnits': 17, 'temperature.145.AmbientWH31BPacket': 20.3, 'humidity.145.AmbientWH31BPacket': 43.0, 'battery.145.AmbientWH31BPacket': 1, 'channel.145.AmbientWH31BPacket': 1, 'rssi.145.AmbientWH31BPacket': None, 'snr.145.AmbientWH31BPacket': None, 'noise.145.AmbientWH31BPacket': None}]
2024-03-03T18:57:04.804488-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521021, 'usUnits': 17, 'temperature.202.AmbientWH31BPacket': 3.9, 'humidity.202.AmbientWH31BPacket': 79.0, 'battery.202.AmbientWH31BPacket': 1, 'channel.202.AmbientWH31BPacket': 5, 'rssi.202.AmbientWH31BPacket': None, 'snr.202.AmbientWH31BPacket': None, 'noise.202.AmbientWH31BPacket': None}
2024-03-03T18:57:04.805766-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521021, 'usUnits': 17, 'temperature.202.AmbientWH31BPacket': 3.9, 'humidity.202.AmbientWH31BPacket': 79.0, 'battery.202.AmbientWH31BPacket': 1, 'channel.202.AmbientWH31BPacket': 5, 'rssi.202.AmbientWH31BPacket': None, 'snr.202.AmbientWH31BPacket': None, 'noise.202.AmbientWH31BPacket': None}]
2024-03-03T18:57:12.143936-08:00 nuc weewxd[20120]: DEBUG user.sdr: packet={'soilMoist1': 39.0, 'dateTime': 1709521029, 'usUnits': 16}
2024-03-03T18:57:12.147865-08:00 nuc weewxd[20120]: DEBUG user.sdr: ignoring duplicate packet {'soilMoist1': 39.0, 'dateTime': 1709521029, 'usUnits': 16}
2024-03-03T18:57:33.113212-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521048, 'usUnits': 17, 'temperature.196.AmbientWH31BPacket': 21.0, 'humidity.196.AmbientWH31BPacket': 42.0, 'battery.196.AmbientWH31BPacket': 1, 'channel.196.AmbientWH31BPacket': 3, 'rssi.196.AmbientWH31BPacket': None, 'snr.196.AmbientWH31BPacket': None, 'noise.196.AmbientWH31BPacket': None}
2024-03-03T18:57:33.113890-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521048, 'usUnits': 17, 'temperature.196.AmbientWH31BPacket': 21.0, 'humidity.196.AmbientWH31BPacket': 42.0, 'battery.196.AmbientWH31BPacket': 1, 'channel.196.AmbientWH31BPacket': 3, 'rssi.196.AmbientWH31BPacket': None, 'snr.196.AmbientWH31BPacket': None, 'noise.196.AmbientWH31BPacket': None}]
2024-03-03T18:57:33.114193-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521049, 'usUnits': 17, 'temperature.153.AmbientWH31BPacket': 21.0, 'humidity.153.AmbientWH31BPacket': 41.0, 'battery.153.AmbientWH31BPacket': 1, 'channel.153.AmbientWH31BPacket': 2, 'rssi.153.AmbientWH31BPacket': None, 'snr.153.AmbientWH31BPacket': None, 'noise.153.AmbientWH31BPacket': None}
2024-03-03T18:57:33.115214-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521049, 'usUnits': 17, 'temperature.153.AmbientWH31BPacket': 21.0, 'humidity.153.AmbientWH31BPacket': 41.0, 'battery.153.AmbientWH31BPacket': 1, 'channel.153.AmbientWH31BPacket': 2, 'rssi.153.AmbientWH31BPacket': None, 'snr.153.AmbientWH31BPacket': None, 'noise.153.AmbientWH31BPacket': None}]
2024-03-03T18:57:56.182914-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521073, 'usUnits': 17, 'temperature.55.AmbientWH31BPacket': 21.2, 'humidity.55.AmbientWH31BPacket': 43.0, 'battery.55.AmbientWH31BPacket': 1, 'channel.55.AmbientWH31BPacket': 4, 'rssi.55.AmbientWH31BPacket': None, 'snr.55.AmbientWH31BPacket': None, 'noise.55.AmbientWH31BPacket': None}
2024-03-03T18:57:56.184639-08:00 nuc weewxd[20120]: message repeated 3 times: [ INFO user.sdr: unmapped: {'dateTime': 1709521073, 'usUnits': 17, 'temperature.55.AmbientWH31BPacket': 21.2, 'humidity.55.AmbientWH31BPacket': 43.0, 'battery.55.AmbientWH31BPacket': 1, 'channel.55.AmbientWH31BPacket': 4, 'rssi.55.AmbientWH31BPacket': None, 'snr.55.AmbientWH31BPacket': None, 'noise.55.AmbientWH31BPacket': None}]
2024-03-03T18:58:04.833572-08:00 nuc weewxd[20120]: INFO user.sdr: unmapped: {'dateTime': 1709521081, 'usUnits': 17, 'temperature.145.AmbientWH31BPacket': 20.3, 'humidity.145.AmbientWH31BPacket': 43.0, 'battery.145.AmbientWH31BPacket': 1, 'channel.145.AmbientWH31BPacket': 1, 'rssi.145.AmbientWH31BPacket': None, 'snr.145.AmbientWH31BPacket': None, 'noise.145.AmbientWH31BPacket': None}
```

What's this mean - we have 'known' entries we need to map in the sensor_map.  It even tells you what to add !

Lets map them to extraTemp1-5 for sensor id 1-5 respectively.

```
    (add to the [sensor_map] stanza)

        extraTemp1     = temperature.145.AmbientWH31BPacket
        extraTemp2     = temperature.153.AmbientWH31BPacket
        extraTemp3     = temperature.196.AmbientWH31BPacket
        extraTemp4     = temperature.55.AmbientWH31BPacket
        extraTemp5     = temperature.202.AmbientWH31BPacket
```

Lets fire weewx up again and watch the logs.  Hopefully everything is mapped and logs are quiet

```
2024-03-03T19:04:52.780682-08:00 nuc weewxd[20353]: DEBUG user.sdr: ignoring duplicate packet {'extraTemp3': 21.0, 'dateTime': 1709521489, 'usUnits': 17}
2024-03-03T19:04:52.781207-08:00 nuc weewxd[20353]: message repeated 2 times: [ DEBUG user.sdr: ignoring duplicate packet {'extraTemp3': 21.0, 'dateTime': 1709521489, 'usUnits': 17}]
2024-03-03T19:05:11.910979-08:00 nuc weewxd[20353]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709521508, 'usUnits': 17}
2024-03-03T19:05:11.913128-08:00 nuc weewxd[20353]: DEBUG user.sdr: ignoring duplicate packet {'extraTemp1': 20.3, 'dateTime': 1709521508, 'usUnits': 17}
2024-03-03T19:05:11.914072-08:00 nuc weewxd[20353]: message repeated 2 times: [ DEBUG user.sdr: ignoring duplicate packet {'extraTemp1': 20.3, 'dateTime': 1709521508, 'usUnits': 17}]
2024-03-03T19:05:22.133345-08:00 nuc weewxd[20353]: DEBUG user.sdr: packet={'soilMoist1': 39.0, 'dateTime': 1709521519, 'usUnits': 16}
2024-03-03T19:05:22.147548-08:00 nuc weewxd[20353]: INFO weewx.manager: Added record 2024-03-03 19:05:00 PST (1709521500) to database 'weewx.sdb'
2024-03-03T19:05:22.153696-08:00 nuc weewxd[20353]: INFO weewx.manager: Added record 2024-03-03 19:05:00 PST (1709521500) to daily summary in 'weewx.sdb'
2024-03-03T19:05:22.179177-08:00 nuc weewxd[20353]: DEBUG weewx.reportengine: Running reports for latest time in the database.
```

Not quite but better. We have duplicate packets heard (we'll deal with it shortly) but it did save to the db.  Lets see if all the mapped items made it.

```
echo "select datetime(dateTime,'unixepoch','localtime'),dateTime,soilMoist1,extraTemp1,extraTemp2,extraTemp3,extraTemp4,extraTemp5,extraTemp6 from archive;" | sqlite3 ~/weewx-data/archive/weewx.sdb
2024-03-03 19:05:00|1709521500|70.0|68.54|69.62|69.8|70.16|39.02|
```

Ok - lets handle the duplicate packet logs.  They're a little annoying.  What's happening is the driver is logging those if debug > 0 in weewx.conf which is a little chatty perhaps.  Lets make it more discretely controllable by lightly patching sdr.py a bit more by editing around line 3339 in sdr.py.

change:
```
                            else:
                                logdbg("ignoring duplicate packet %s" % pkt)
```
to be as follows.  Be sure to indent the lines correctly.
```
                            else:
                                if self._log_dups:
                                    logdbg("ignoring duplicate packet %s" % pkt)
```
and we need to add a line around line 3304
```
        self._log_unknown = tobool(stn_dict.get('log_unknown_sensors', False))
        self._log_unmapped = tobool(stn_dict.get('log_unmapped_sensors', False))
        self._log_dups     = tobool(stn_dict.get('log_duplicate_readings', True))   # add me
```

The result here is that if log_duplicate_readings = False in weewx.conf's SDR stanza, this noise won't be logged.  Lets try it.  The SDR stanza at this point should look like:

```
[SDR]
    # This section is for the software-defined radio driver.

    # The driver to use
    driver = user.sdr
    cmd = rtl_433 -f 915M -F json -M utc

    log_unknown_sensors  = True
    log_unmapped_sensors = True

    log_duplicate_readings = False  # suppress logging duplicate packets

    [[sensor_map]]
    soilMoist1 = soil_moisture_percent.001260.FOWH51Packet
    extraTemp1 = temperature.145.AmbientWH31BPacket
    extraTemp2 = temperature.153.AmbientWH31BPacket
    extraTemp3 = temperature.196.AmbientWH31BPacket
    extraTemp4 = temperature.55.AmbientWH31BPacket
    extraTemp5 = temperature.202.AmbientWH31BPacket
```

Save the file, start weewx up again, and watch your log.  The duplicates messages should be suppressed now.  Logging the packets is still True so we expect to see those.

```
2024-03-03T19:40:20.680760-08:00 nuc weewxd[22536]: DEBUG user.sdr: parse_json: unknown model Fineoffset-WH32
2024-03-03T19:40:20.681629-08:00 nuc weewxd[22536]: DEBUG user.sdr: punt unrecognized line '{"time" : "2024-03-04 03:40:17", "model" : "Fineoffset-WH32", "id" : 35, "battery_ok" : 1, "temperature_C" : 3.200, "humidity" : 91, "mic" : "CRC"}#012'

```

So we're not 'quite' there.  The WH32 isn't known to the driver either because the model 'Fineoffset-WH32' is unknown.   We need to similarly edit sdr.py to add a stanza for this model sensor.  Add a stanza below 'class AmbientWH31BPacket(Packet)' with the same content but change references to WH31B in the new class to say WH32 in that class. The WH32 doesn't have elements for rssi, signal, noise so delete those lines.  Now fire weewx up yet again.

```
2024-03-03T19:45:21.138529-08:00 nuc weewxd[22832]: INFO weewx.engine: Starting main packet loop.


2024-03-03T19:45:51.905151-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp3': 21.0, 'dateTime': 1709523946, 'usUnits': 17}
2024-03-03T19:45:51.909688-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp5': 3.3, 'dateTime': 1709523946, 'usUnits': 17}
2024-03-03T19:45:51.910787-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709523948, 'usUnits': 17}
2024-03-03T19:46:07.221681-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp2': 20.9, 'dateTime': 1709523964, 'usUnits': 17}
2024-03-03T19:46:14.180798-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp4': 21.4, 'dateTime': 1709523971, 'usUnits': 17}
2024-03-03T19:46:54.937224-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp3': 20.9, 'dateTime': 1709524009, 'usUnits': 17}
2024-03-03T19:46:54.940410-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709524009, 'usUnits': 17}
2024-03-03T19:46:54.942488-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp5': 3.2, 'dateTime': 1709524011, 'usUnits': 17}
2024-03-03T19:47:09.227229-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp2': 20.9, 'dateTime': 1709524026, 'usUnits': 17}
2024-03-03T19:47:23.251646-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'soilMoist1': 39.0, 'dateTime': 1709524039, 'usUnits': 16}
2024-03-03T19:47:23.254187-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp4': 21.4, 'dateTime': 1709524040, 'usUnits': 17}
2024-03-03T19:47:55.886991-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709524070, 'usUnits': 17}
2024-03-03T19:47:55.890016-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp3': 21.0, 'dateTime': 1709524072, 'usUnits': 17}
2024-03-03T19:47:59.948806-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp5': 3.2, 'dateTime': 1709524076, 'usUnits': 17}
2024-03-03T19:48:11.216947-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp2': 20.9, 'dateTime': 1709524088, 'usUnits': 17}
2024-03-03T19:48:54.999936-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709524131, 'usUnits': 17}
2024-03-03T19:48:58.805419-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp3': 21.0, 'dateTime': 1709524135, 'usUnits': 17}
2024-03-03T19:49:04.959500-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp5': 3.2, 'dateTime': 1709524141, 'usUnits': 17}
2024-03-03T19:49:13.219531-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp2': 20.9, 'dateTime': 1709524150, 'usUnits': 17}
2024-03-03T19:49:41.270297-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp4': 21.3, 'dateTime': 1709524178, 'usUnits': 17}
2024-03-03T19:49:55.950325-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp1': 20.3, 'dateTime': 1709524192, 'usUnits': 17}
2024-03-03T19:50:01.849141-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp3': 20.9, 'dateTime': 1709524198, 'usUnits': 17}
2024-03-03T19:50:09.847488-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp5': 3.2, 'dateTime': 1709524206, 'usUnits': 17}
2024-03-03T19:50:15.217733-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp2': 20.9, 'dateTime': 1709524212, 'usUnits': 17}
2024-03-03T19:50:50.218722-08:00 nuc weewxd[22832]: DEBUG user.sdr: packet={'extraTemp4': 21.3, 'dateTime': 1709524247, 'usUnits': 17}
2024-03-03T19:50:50.231614-08:00 nuc weewxd[22832]: INFO weewx.manager: Added record 2024-03-03 19:50:00 PST (1709524200) to database 'weewx.sdb'
2024-03-03T19:50:50.239427-08:00 nuc weewxd[22832]: INFO weewx.manager: Added record 2024-03-03 19:50:00 PST (1709524200) to daily summary in 'weewx.sdb'
2024-03-03T19:50:50.269145-08:00 nuc weewxd[22832]: DEBUG weewx.reportengine: Running reports for latest time in the database.
2024-03-03T19:50:50.269298-08:00 nuc weewxd[22832]: DEBUG weewx.reportengine: Running report 'SeasonsReport'
```

Looks good.  No unknown models.  No duplicate packets.  Now set log_packets = False to quiet 'that' down.   We also need to add one more small patch to quiet down debug=1 logs further.   First edit weewx.conf to add one more line there to the SDR stanza.

```
[SDR]
    # This section is for the software-defined radio driver.

    # The driver to use
    driver = user.sdr
    cmd = rtl_433 -f 915M -F json -M utc

    log_unknown_sensors  = True
    log_unmapped_sensors = True

    log_packets            = False  # log mapped packets if debug > 0          <=== add this
    log_duplicate_readings = False  # suppress logging duplicate packets

    [[sensor_map]]
    soilMoist1 = soil_moisture_percent.001260.FOWH51Packet
    extraTemp1 = temperature.145.AmbientWH31BPacket
    extraTemp2 = temperature.153.AmbientWH31BPacket
    extraTemp3 = temperature.196.AmbientWH31BPacket
    extraTemp4 = temperature.55.AmbientWH31BPacket
    extraTemp5 = temperature.202.AmbientWH31BPacket
```

Now edit sdr.py similar to the edits we did before when we made logging duplicates more configurable.  Again this is around line 3332 or so.  Make sure the two lines marked with 'add me' are added to your sdr.py file.

```
  def __init__(self, **stn_dict):
        loginf('driver version is %s' % DRIVER_VERSION)
        self._model = stn_dict.get('model', 'SDR')
        loginf('model is %s' % self._model)
        self._log_lines = tobool(stn_dict.get('log_lines', False))
        self._log_unknown = tobool(stn_dict.get('log_unknown_sensors', False))
        self._log_unmapped = tobool(stn_dict.get('log_unmapped_sensors', False))
        self._log_packets  = tobool(stn_dict.get('log_packets', True))              # add me
        self._log_dups     = tobool(stn_dict.get('log_duplicate_readings', True))   # add me
        self._sensor_map = stn_dict.get('sensor_map', {})
```

and a little futher down indent the last line above and put an if statement directly above it.
```

    def genLoopPackets(self):
        while self._mgr.running():
            for lines in self._mgr.get_stdout():
                if self._log_lines:
                    loginf("lines: %s" % lines)
                for packet in PacketFactory.create(lines):
                    if packet:
                        pkt = self.map_to_fields(packet, self._sensor_map)
                        if pkt:
                            if not self._packets_match(pkt, self._last_pkt):
                                if self._log_packets:
                                    logdbg("packet=%s" % pkt)
```

At this point your debug=1 logging is much more configurable.  You can suppress logging duplicates or the resulting packets by setting the weewx.conf settings to False.  


```
2024-03-03T19:57:05.955825-08:00 nuc weewxd[23505]: INFO __main__: Starting up weewx version 5.0.2
2024-03-03T19:57:05.955885-08:00 nuc weewxd[23505]: DEBUG weewx.engine: Station does not support reading the time
2024-03-03T19:57:05.955943-08:00 nuc weewxd[23505]: INFO weewx.engine: Using binding 'wx_binding' to database 'weewx.sdb'
2024-03-03T19:57:05.956003-08:00 nuc weewxd[23505]: INFO weewx.manager: Starting backfill of daily summaries
2024-03-03T19:57:05.956151-08:00 nuc weewxd[23505]: INFO weewx.manager: Daily summaries up to date
2024-03-03T19:57:05.956231-08:00 nuc weewxd[23505]: INFO weewx.engine: Starting main packet loop.
2024-03-03T20:00:31.837493-08:00 nuc weewxd[23505]: INFO weewx.manager: Added record 2024-03-03 20:00:00 PST (1709524800) to database 'weewx.sdb'
2024-03-03T20:00:31.852588-08:00 nuc weewxd[23505]: INFO weewx.manager: Added record 2024-03-03 20:00:00 PST (1709524800) to daily summary in 'weewx.sdb'
2024-03-03T20:00:31.878884-08:00 nuc weewxd[23505]: DEBUG weewx.reportengine: Running reports for latest time in the database.
2024-03-03T20:00:31.879006-08:00 nuc weewxd[23505]: DEBUG weewx.reportengine: Running report 'SeasonsReport'
```

Nice and clean.  No unmapped sensors.  No duplicate packet noise in the logs.  No unknown sensor models.  Call it done and release the victory beer(s) !!!!

If you want to cut to the chase, the modified sdr.py has been submitted to upstream as https://github.com/matthewwall/weewx-sdr/pull/184 and the modified sdr.py and matching weewx.conf snippet are located here in this repo.
