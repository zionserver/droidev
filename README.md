# droidev

How to change the IMEI on Android devices

Disclaimer The following information is intended for research and development purposes.

Do NOT change the IMEI of a mobile device that has been stolen. There are other ways of tracking such devices, and changing the IMEI does not make their use any safer, but can negatively influence the stability of mobile phone networks.
Changing the IMEI as a temporary measure is a tool for development (e.g. testing of Android apps that use the IMEI as an identifier, fuzzing such use, or provoking error cases in software). Before using the phone normally again, the IMEI should be changed back to the original. Therefore, copy the original IMEI before changing it.
Any experiments with modified IMEI are best done with disabled GSM/UMTS/LTE support (i.e. "airplane mode"), although the use of WLAN or other network connections is not restricted. Mobile network operators monitor their networks and you may be held accountable for creating network problems with modified IMEIs.
WARNING! If something went wrong it could potentially damage your phone! We are not responsibility for any damage.

Prerequisites
Android device with fastboot (Samsung devices typically do not implement this mode in the standard bootloader, but most other manufacturers support it)
rooted device
bootloader unlocked
fastboot utility installed on host computer
adb installed on host computer (not needed, but recommended)
Step 1
Access fastboot

This can be done either via button combination which differs from device to device or the easy method over adb command

adb reboot fastboot
 

Step 2
Set new IMEI

fastboot oem writeimei 123456789012347
 

Step 3
Check IMEI

Verify that the IMEI has changed successfully run:

fastboot getvar imei
which should show the actual IMEI of the device.

 

Step 4
Reboot

fastboot reboot
Reboot device and enter *#06# into the dialer. Be aware, if the entered IMEI is not valid, it will not be shown on the device. Here is a short overview about the IMEI structure:

AA:	Type Allocation Code (TAC), first two digits are the reporting body identifier
BBBBBB:	Remainder of the TAC (FAC), manufacturer & phone type
CCCCCC:	Serial number of model (SNR)
D:	Luhn check digit (CD)
 

I also wrote a short BASH script which allows Linux user to quickly change the IMEI or generate a random valid one (valid in sense of: phone will accept the generated number, provider usually not). This may be useful for quickly testing the behavior of apps that rely on the IMEI as a unique, stable identifier. Here is the code:

#!/bin/bash


#check if correct parameter passed to the script
if [[ $# -ne 1 ]];then
  echo "usage: change_imei [imei|'rand']"
  exit 1
fi

imei=$1

if [[ $imei == 'rand' ]];then
  #generate a random imei number which is valid for the device
  imei="35"
  range=10;
  for i in {0..11}; do
    r=$RANDOM;
    let "r %= $range";
    imei="$imei""$r";
  done;

  #generate luhn check digit
  a=$((${imei:0:1} + ${imei:2:1} + ${imei:4:1} + ${imei:6:1} + ${imei:8:1} + ${imei:10:1} + ${imei:12:1}))
  b="$((${imei:1:1}*2))$((${imei:3:1}*2))$((${imei:5:1}*2))$((${imei:7:1}*2))$((${imei:9:1}*2))$((${imei:11:1}*2))$((${imei:13:1}*2))"
  c=0

  for (( i=0; i<${#b}; i++ )); do
    c=$(($c+${b:$i:1}))
  done

  d=$(($a + $c))
  luhn=$((10-$(($d%10))))
  if [[ "$luhn" -eq 10 ]]; then luhn=0; fi
  
  #set imei with luhn digit
  imei="$imei$luhn"

else
  #check if length of imei is ok
  if [[ ${#1} -ne 15 ]];then
    echo "length of imei not correct"
    exit 1
  fi
fi

#reboot into bootloader
adb reboot bootloader &>/dev/null

#check if we are in fastboot already
sudo sh -c "fastboot getvar imei" &>/dev/null
sleep 1

#get the old imei
old_imei=$(sudo sh -c "fastboot getvar imei 2>&1" | sed -n 1p | awk '{print $2}')

#write the new one
sudo sh -c "fastboot oem writeimei $imei" &>/dev/null

#get the new set imei
new_imei=$(sudo sh -c "fastboot getvar imei 2>&1" | sed -n 1p | awk '{print $2}')

#reboot device 
sudo sh -c "fastboot reboot" &>/dev/null

#check if the new imei matches the imei on the phone
if [[ $imei == $new_imei ]]; then
  echo -e "old imei: $old_imei\nnew imei: $imei"
else
  echo -e "something went wrong\nactual imei: $new_imei"
fi

This code has been tested on HTC M7 device on an Linux machine with adb and fastboot installed on. I do not give any warranty of functionality of this code - maybe you need to adapt some of the code to work correct under other devices as well. If you want to support a 17 digit long IMEI you can drop the luhn check digit and add two additional digits which stands for software version.
