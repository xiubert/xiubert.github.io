---
permalink: /noiseConsumption/
title: "Noise Consumption"
toc: true
toc_label: "contents:"
toc_icon: "fas fa-broadcast-tower"
last_modified_at: 6/1/2021
---

This here is all about consuming the noise, mostly concerning the how.

## Multi-room audio

When you need the noise in as many places as possible and possibly at the same time. This is not meant to be a difinitive guide, just a place for orienting.

**OBJECTIVE**:  synced multi-room audio via AirPlay or line-in (eg from a bluetooth receiver, pre-amp via turntable, etc).

### **SOFTWARE (free)**

| tool | use |
| ------------------------------------------- | ----------------------------------------------------- |
| [shairport-sync](https://github.com/mikebrady/shairport-sync) | AirPlay audio player |
| [forked-daapd](https://github.com/ejurgensen/forked-daapd) | Media Server for combining AirPlay outputs (via shairport-sync) |
| [cpiped](https://github.com/b-fitzpatrick/cpiped) | Trigger events upon line-in sound detection (eg. send audio from bluetooth receiver plugged into line-in) |

#### Overview 

AirPlay audio source:
1.  Phone sends AirPlay audio to shairport-sync AirPlay sink on server running forked-daapd.
2.  Shairport-sync AirPlay sink outputs to a named FIFO pipe (`airplayPipe`)


Line-in audio source:
1.  Sound starts at audio source plugged into line-in on server running forked-daapd (eg. music is sent to bluetooth receiver plugged into line-in at server).
2.  Cpiped detects audio at line-in and outputs to a sound detection FIFO pipe, this tiggers a script that outputs the detection pipe to a sound output FIFO pipe (`lineInPipe`)

Lastly:  

Forked-daapd automatically detects sound at named FIFO pipe (either `lineInPipe` or `airplayPipe`) and outputs sound to selected shairport-sync (AirPlay) sinks on the home network selected via forked-daapd web interface (http://[server_IP]:3689). [See here.](https://github.com/ejurgensen/forked-daapd#pipes-for-eg-multiroom-with-shairport-sync)


#### Step-by-step
1. Install [shairport-sync](https://github.com/mikebrady/shairport-sync#building-and-installing) both on the server that will run forked-daapd and on the networked AirPlay sinks (eg. Raspberry Pi's with HiFiBerry DAC hats). Below is a likely build configuration (**note**: `--with-pipe` flag only needed on forked-daapd server)
    ```
    ./configure --sysconfdir=/etc --with-alsa --with-avahi --with-ssl=openssl --with-soxr --with-systemd --with-pipe
    ```

    On the main server, you'll likely want multiple AirPlay sinks: one for the multi-room audio going to forked-daapd and another for audio output local to the server. To do this create a shairport-sync conf file and systemd unit for each as such: `/etc/shairport-sync[OutputName].conf`, `shairport-sync@[OutputName].service`.  Note that ports for each must be unique.

    `$ cat /etc/shairport-syncForkedDAAPD.conf`
    ```
    // General Settings
    general = {
      name = "Multi-Room Audio";
      port = 5000; #must be differ from local server audio shairport-sync conf
      output_backend = "pipe";
      #drift_tolerance_in_seconds = 0.020;
      #resync_threshold_in_seconds = 0.100;
    };

    sessioncontrol = {
      allow_session_interruption = "yes";
      session_timeout = 20;
    }

    pipe = {
      name = "/var/media/airplayPipe";
      #audio_backend_latency_offset = 0;
      audio_backend_buffer_desired_length = 44100;
    }
    ```
    Write FIFO pipe (`/var/media/airplayPipe`) and configure permissions on it such that both daapd and shairport-sync have access:
    ```
    # mkfifo /var/media/airplayPipe
    # chown shairport-sync:shairport-sync /var/media/airplayPipe
    # chmod 771 /var/media/airplayPipe
    # gpasswd -a daapd shairport-sync
    ```

    `$ cat /etc/systemd/system/shairport-sync@ForkedDAAPD.service`
    ```
    [Unit]
    Description=Shairport Sync - AirPlay Audio Receiver %I
    After=sound.target
    Requires=avahi-daemon.service
    After=avahi-daemon.service
    Wants=network-online.target
    After=network.target network-online.target

    [Service]
    ExecStart=/usr/local/bin/shairport-sync -c /etc/shairport-sync%I.conf
    User=shairport-sync
    Group=shairport-sync
    SyslogIdentifier=shairport-sync-%I

    [Install]
    WantedBy=multi-user.target
    ```

    `$ cat /etc/shairport-syncServerAudio.conf`
    ```
    // General Settings
    general = {
      name = "Local Server Audio";
      #password = "secret";
      port = 5002; #must differ from that in shairport-syncForkedDAAPD.conf
    };
    alsa = {
      output_device = "default:Audio";
    };
    ```
    ServerAudio systemd service file contents same as above, but file named: `/etc/systemd/system/shairport-sync@ServerAudio.service`

2. Install [forked-daapd](https://github.com/ejurgensen/forked-daapd/blob/master/INSTALL.md) and configure.  These are likely build configs:

    ```
    ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --without-pulseaudio --enable-chromecast
    ```

    `$ cat /etc/forked-daapd.conf`
    ```
    general {
            uid = "daapd"
            logfile = "/var/log/forked-daapd.log"
            loglevel = log
            admin_password = "secret"
            #ipv6 = no
    }

    library {
            name = "Music on %h"
            port = 3689
            directories = { "/var/media" }
    }

    audio {
            nickname = "Server Name"
    }

    #in case you have password protected shairport-sync endpoints/sinks
    airplay "endpointName" {
            password = "secret"
    }
    ```

3. Configure firewall (below is an example for iptables)

    ```
    # iptables -A INPUT -s 192.168.1.0/24 -i eth0 -p udp -m udp --dport 5353 -m comment --comment "avahi local" -j ACCEPT
    # iptables -A INPUT -s 192.168.1.0/24 -i eth0 -p tcp -m tcp --dport 5002 -m comment --comment "shairport local" -j ACCEPT
    # iptables -A INPUT -s 192.168.1.0/24 -i eth0 -p tcp -m tcp --dport 5000 -m comment --comment "shairport local" -j ACCEPT
    # iptables -A INPUT -s 192.168.1.0/24 -i eth0 -p tcp -m tcp --dport 3689 -m comment --comment "forked daapd local" -j ACCEPT
    # iptables -A INPUT -s 192.168.1.0/24 -i eth0 -p tcp -m tcp --dport 3688 -m comment --comment "forked daapd local" -j ACCEPT

    # iptables-save > /etc/iptables/iptables.rules
    # iptables-restore < /etc/iptables/iptables.rules
    # systemctl restart iptables
    # iptables -nvL --line-numbers
    ```

4. Install and configure cpiped for detection of audio at line-in and subsequent script triggering. Cpiped will detect audio at server line-in (eg from a bluetooth receiver) and output it to a named FIFO pipe (`/var/soundDetected`).  Upon detection it will trigger a script that outputs '/var/soundDetected' FIFO pipe to a separate FIFO pipe (`/var/media/lineInPipe`) that forked-daapd will read and send to airplay endpoints.  See [original post here](https://github.com/ejurgensen/forked-daapd/issues/632#issuecomment-454637959) and based on that, a [handy guide here](https://github.com/ejurgensen/forked-daapd/wiki/Making-an-Analog-In-to-Airplay-RPi-Transmitter).

    get and compile:

    ```
    $ git clone "https://github.com/b-fitzpatrick/cpiped.git"
    $ cd cpiped
    $ make cpiped
    # cp cpiped /usr/local/bin/cpiped
    ```

    create scripts to run on sound detection (`soundDetected.sh`) and sound abscence (`soundAbsence.sh`):

    `$ cat /home/daapd/soundDetected.sh`
    ```
    #!/bin/bash

    #in case you want to send forked-daapd playback settings
    #curl -X PUT "http://localhost:3689/api/queue/clear"

    cat /home/daapd/soundDetected > /var/media/lineInPipe &
    echo $! > /tmp/cpipedCatPipe.pid
    ```

    `$ cat /home/daapd/soundAbsence.sh`
    ```
    #!/bin/bash

    #curl -X PUT "http://localhost:3689/api/player/stop"

    kill $(< /tmp/cpipedCatPipe.pid)
    echo " " > /tmp/cpipedCatPipe.pid
    ```

    Create systemd unit for cpiped.  The destination flag parameter for cpiped (-d) may vary for your setup.  Check with `arecord -l`; in `hw:0,0`, first `0` is card number and second is subdevice number.

    `$ cat /etc/systemd/system/cpiped.service`
    ```
    [Unit]
    Description=cpiped
    After=sound.target

    [Service]
    Type=forking
    KillMode=none
    User=daapd
    ExecStart=/usr/local/bin/cpiped -d "hw:0,0" -s /home/daapd/soundDetected.sh -e /home/daapd/soundAbsence.sh -D /home/daapd/soundDetected
    ExecStop=/usr/bin/killall -9 cpiped
    WorkingDirectory=/home/daapd

    [Install]
    WantedBy=multi-user.target
    ```

    Make sure to create corresponding FIFO pipes as shown above for `/home/daapd/soundDetected` and `/home/daapd/lineInPipe`

<!-- TODO:  organize headings for TOC and confirm consistency -->

### **SOFTWARE ($$)**

 - [Roon](https://roonlabs.com/) ([install guide](https://help.roonlabs.com/portal/en/kb/articles/linux-install))

 Roon will do all of the above and it will enable you to play different music located on the main server at different sound endpoints.  This could otherwise be achieved with a sound server such as [airsonic](https://airsonic.github.io/) + shairport-sync endpoints.

 Roon is meant to be run solely on your local network. If you'd like it elsewhere you can remedy this with a VPN server on the main Roon server.




### **HARDWARE**
* Raspberry Pi's paired with [HiFiBerry hats](https://www.hifiberry.com/shop/boards/hifiberry-dacplus-rca-version/) work well as reasonably inexpensive AirPlay/shairport-sync endpoints on the local network
    

## Noise organization

When you have a substantial lot of noise and you need to keep that stuff organized so that it doesn't take you an hour to find that one song by that singer with the hair.

### tools
- free as in beer: [beets](https://beets.readthedocs.io/en/stable/)

- not free as in beer, easy, reliable, robust: [Roon](https://roonlabs.com/features)

__________________________

### Taking charge of your subsonic/airsonic/madsonic database ( via SQL ):

February 17, 2021

This could be of use to migrate from [Subsonic](http://www.subsonic.org/pages/index.jsp) to [Roon](https://roonlabs.com/features).

Rather than navigate Subsonic API docs and query the server, I found it easier to directly access the database.  By default Subsonic uses a HyperSQL database (HSQLDB).  See your trusty Arch docs [here](https://wiki.archlinux.org/index.php/Subsonic#Accessing_the_database).  If I were to start another Subsonic server from scratch, I'd probably opt instead for something more [flexible](http://www.subsonic.org/pages/database.jsp) like PostgreSQL.

See this helpful [reddit post]( https://www.reddit.com/r/subsonic/comments/7c2n6j/database_table_schema/) for a breakdown of the HSQLDB schema.  Also see [airsonic docs](https://airsonic.github.io/docs/database/).

GOAL:  Query playlists by keywords to aggregate file paths to eventually split by top directory so that Roon can scrape them respective of directory.

1. Acquire SqlTool If you're on debian:  `sudo apt install hsqldb-utils`. Otherwise as noted in the [airsonic docs](https://airsonic.github.io/docs/database/) grab and unzip the `hsqldb.jar` file from [here](https://sourceforge.net/projects/hsqldb/files/hsqldb/hsqldb_1_8_1/) and use your JRE to call the sqltool.

2. Run queries interactively. If you desire, output to file with `\o filename.ext` preceding your query.

     **WARNING:  STOP SUBSONIC, MAKE A COPY OF THE `db` FOLDER, and use that for your queries.**
     `cd /var/subsonic/db.copy`

     ```
     hsqldb-sqltool --inlineRc=url=jdbc:hsqldb:file:subsonic,user=sa,password=
     ```

     NOTE: database users aren't mapped from settings, use exact user and pass as above.

     Again, for output to file, precede queries w/ this:
     `\o outputFile.txt`

     You can do this to optimize your queries then save query commands in `.sql` files and redirect output via shell: 
     
3. Run queries stored in `.sql` files.  Here I'm looking for media file paths in playlists that include the word 'house':

     `cat sqlFile.sql`
    ```
    SELECT M.PATH FROM MEDIA_FILE M WHERE M.ID IN (
      SELECT DISTINCT A.MEDIA_FILE_ID FROM PLAYLIST_FILE A WHERE A.PLAYLIST_ID IN (
        SELECT P.ID FROM PLAYLIST P WHERE P.NAME LIKE '%house%'));
    ```

    run this query on your copied db:

    ```
    hsqldb-sqltool --inlineRc=url=jdbc:hsqldb:file:subsonic,user=sa,password= sqlFile.sql > housePlaylists.txt 2>&1
    ```

4.  Process your query outputs!

    `cat splitMediaByTopDir.sh`

    ```
    #!/bin/bash

    echo "splitting $1 into separate playlists"

    paths="random1 random2 /audio/random/ /audio/web/"

    pathLineCount=0
    for path in $paths
    do
      pathLineCount=$(( $pathLineCount + $(grep $path $1 | wc -l) ))
    done
    echo "counted files in path list: $pathLineCount"
    lineCount=$(grep mnt $1 | wc -l)
    echo "all lines: $lineCount"

    if [[ "$pathLineCount" == "$lineCount" ]]; then
      echo "all paths accounted for"
      for path in $paths
      do
        if [[ "$path" == *"audio"* ]]; then
          echo "path has slashes"
          fixpath=${path//\/audio\//}
          fixpath=${fixpath//\//}
        else
          fixpath=$path
        fi
        echo "#EXTM3U" > ${1/.txt/_$fixpath.m3u8}
        grep $path $1 >> ${1/.txt/_$fixpath.m3u8}
      done
    else
      echo "doing nothing, there are paths not accounted for"
    fi
    ```

    Run your script:

    `./splitMediaByTopDir.sh housePlaylists.txt`

    Finally:  drop your playlists in your roon storage directories!



