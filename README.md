# A2DP_on_pi_3
1. Download the required package
	This project depend on pulseaudio so grab it and installing by typing following
	$ sudo apt-get update && sudo apt-get install bluez pulseaudio-module-bluetooth python-gobject python-gobject-2 bluez-tools udev

2. Edit Configuration and apply them
	First add pi username to the group pulseaudio with,
	$ sudo usermod -a -G lp pi
	
	create new config under /etc/bluetooth/audio.conf using text editor and add the following line,
	$ sudo nano /etc/bluetooth/audio.conf
	> [General]:
	> Enable=Source,Sink,Media,Socket
	
	Open file /etc/bluetooth/main.conf
	$ sudo nano /etc/bluetooth/main.conf

	Set Bluetooth Class, Modify the following line to:
	> Class = 0x00041C
	
	0x000041C means that the rpi bluetooth support A2DP protocol.

	change /etc/pulse/daemon.conf resample-method as below,
	$ sudo nano /etc/pulse/daemon.conf
	> : resample-method = speex-float-3
	> ; resample-method = trivial

	Start pulseaudio service
	$ pulseaudio -D

	we are going to use ragusa87 script to automate the bluetooth source to audio sink.
	First please add new configuration to udev init.d by editing file /etc/udev/rules.d/99-input.rules and add this to the file
	$ sudo nano /etc/udev/rules.d/99-input.rules
	> SUBSYSTEM="input", GROUP="input", MODE="0660"
	> KERNEL=="input[0-9]*", RUN+="/usr/lib/udev/bluetooth"

	add folder udev to /usr/lib by using mkdir
	$ sudo mkdir /usr/lib/udev && cd /usr/lib/udev

	and add this to the file bluetooth (credits ragusa87)
	$ sudo nano bluetooth
#!/bin/bash
# This script is called by udev when you link a bluetooth device with your computer
# It's called to add or remove the device from pulseaudio
#
#

# Output to this file
LOGFILE="/var/log/bluetooth_dev"

# Name of the local sink in this computer
# You can get it by calling : pactl list short sinks
# AUDIOSINK="alsa_output.platform-bcm2835_AUD0.0.analog-stereo"
AUDIOSINK="alsa_output.0.analog-stereo.monitor"
# User used to execute pulseaudio, an active session must be open to avoid errors
USER="pi"

# Audio Output for raspberry-pi
# 0=auto, 1=headphones, 2=hdmi. 
AUDIO_OUTPUT=1

# If on, this computer is not discovearable when an audio device is connected
# 0=off, 1=on
ENABLE_BT_DISCOVER=1

echo "For output see $LOGFILE"

## This function add the pulseaudio loopback interface from source to sink
## The source is set by the bluetooth mac address using XX_XX_XX_XX_XX_XX format.
## param: XX_XX_XX_XX_XX_XX
## return 0 on success
add_from_mac(){
  if [ -z "$1" ] # zero params
    then
        echo "Mac not found" >> $LOGFILE
    else
        mac=$1 # Mac is parameter-1

        # Setting source name
        bluez_dev=bluez_source.$mac
        echo "bluez source: $mac"  >> $LOGFILE

        # This script is called early, we just wait to be sure that pulseaudio discovered the device
        sleep 1
        # Very that the source is present
        CONFIRM=`sudo -u pi pactl list short | grep $bluez_dev`
        if [ ! -z "$CONFIRM" ]
        then
            echo "Adding the loopback interface:  $bluez_dev"  >> $LOGFILE
            echo "sudo -u $USER pactl load-module module-loopback source=$bluez_dev sink=$AUDIOSINK rate=44100 adjust_time=0"  >> $LOGFILE

            # This command route audio from bluetooth source to the local sink..
            # it's the main goal of this script
            sudo -u $USER pactl load-module module-loopback source=$bluez_dev sink=$AUDIOSINK rate=44100 adjust_time=0  >> $LOGFILE
            return $?
        else
            echo "Unable to find a bluetooth device compatible with pulsaudio using the following device: $bluez_dev" >> $LOGFILE
            return -1
        fi
    fi
}

## This function set volume to maximum and choose the right output
## return 0 on success
volume_max(){
    # Set the audio OUTPUT on raspberry pi
    # amixer cset numid=3 <n> 
    # where n is 0=auto, 1=headphones, 2=hdmi. 
    amixer cset numid=3 $AUDIO_OUTPUT  >> $LOGFILE

    # Set volume level to 100 percent
    amixer set Master 100%   >> $LOGFILE
    pacmd set-sink-volume 0 65537   >> $LOGFILE
    return $?
}

## This function will detect the bluetooth mac address from input device and configure it.
## Lots of devices are seen as input devices. But Mac OS X is not detected as input
## return 0 on success
detect_mac_from_input(){
    ERRORCODE=-1

    echo "Detecting mac from input devices" >> $LOGFILE
    for dev in $(find /sys/devices/virtual/input/ -name input*)
    do
        if [ -f "$dev/name" ]
        then
            mac=$(cat "$dev/name" | sed 's/:/_/g')
            add_from_mac $mac

            # Endfor if the command is successfull
            ERRORCODE=$?
            if [ $ERRORCODE -eq 0]; then
                return 0
            fi
        fi
    done
    # Error
    return $ERRORCODE
}
## This function will detect the bt mac address from dev-path and configure it.
## Devpath is set by udev on device link
## return 0 on success
detect_mac_from_devpath(){
    ERRORCODE=-1
    if [ ! -z "$DEVPATH" ]; then
        echo "Detecting mac from DEVPATH"  >> $LOGFILE
        for dev in $(find /sys$DEVPATH -name address)
        do
            mac=$(cat "$dev" | sed 's/:/_/g')
            add_from_mac $mac

            # Endfor if the command is successfull
            ERRORCODE=$?
            if [ $ERRORCODE -eq 0]; then
                return 0
            fi

        done
        return $ERRORCODE;
    else
        echo "DEVPATH not set, wrong bluetooth device? " >> $LOGFILE
        return -2
    fi
    return $ERRORCODE
}


## Detecting if an action is set
if [ -z "$ACTION" ]; then
    echo "The script must be called from udev." >> $LOGFILE
    exit -1;
fi
## Getting the action
ACTION=$(expr "$ACTION" : "\([a-zA-Z]\+\).*")

# Switch case
case "$ACTION" in
"add")

    # Turn off bluetooth discovery before connecting existing BT device to audio
    if [ $ENABLE_BT_DISCOVER -eq 1]; then
        echo "Stet computer as hidden" >> $LOGFILE
        hciconfig hci0 noscan
    fi

    # Turn volume to max
    volume_max

    # Detect BT Mac Address from input devices
    detect_mac_from_input
    OK=$?

    # Detect BT Mac address from device path on a bluetooth event
    if [ $OK != 0 ]; then
        if [ "$SUBSYSTEM" == "bluetooth" ]; then
            detect_mac_from_devpath
            OK=$?
        fi
    fi

    # Check if the add was successfull, otherwise display all available sources
    if [ $OK != 0 ]; then
        echo "Your bluetooth device is not detected !" >> $LOGFILE
        echo "Available sources are:" >> $LOGFILE
        sudo -u $USER pactl list short sources >> $LOGFILE
    else
        echo "Device successfully added " >> $LOGFILE
    fi
    ;;

"remove")
    # Turn on bluetooth discovery if device disconnects
    if [ $ENABLE_BT_DISCOVER -eq 1]; then
        echo "Set computer as visible" >> $LOGFILE
        sudo hciconfig hci0 piscan
    fi
    echo "Removed" >> $LOGFILE
    ;;

#   
*)
    echo "Unsuported action $action" >> $LOGFILE
    ;;
esac
echo "--" >> $LOGFILE
	
	PLEASE NOTE that your AUDIOSINK might different from mine, check it before using 
	$ pactl list short sinks
	
	make the script executable by inputting this code
	$ chmod 777 bluetooth 

	plug in headset to test whether the audio jack working and test with
	$ aplay /usr/share/sounds/alsa/Front_Center.wav
	
	or you can set the default audio routing with
	$ sudo amixer cset numid=3 n
	where n could be: 0 = auto 1 = jack 2 = hdmi

3. Pair and Connect the audio
	go to terminal and type 
	$ bluetoothctl
	
	First activate bluetooth with
	# power on
	# agent on

	set the default agent that you've been editing before with 
	# default-agent
	
	and then set discoverable mode and pair mode on with 
	# discoverable on
	# pairable on

	You should see raspberrypi bluetooth on your phone or laptop and you can pair it on the phone by clicking it and touch pair
	On the terminal you type y
	you connect to the phone by typing
	# connect xx:xx:xx:xx:xx:xx # where xx:xx:xx:xx:xx:xx is you phone bluetooth mac address. 
	and don't forget to trust with 
	# trust xx:xx:xx:xx:xx:xx #where xx:xx:xx:xx:xx:xx
