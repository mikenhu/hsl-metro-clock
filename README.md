# HSL Metro Clock

I came across [this tweet on X](https://twitter.com/eddible/status/1564917603180617731?s=20&t=dcHyyQINVi-xO-h7mmJiKw) and was inspired to create one for myself.

The project calls to Helsinki Region Transport (HSL) which uses GTFS Realtime feeds. If you are not living in Finland, I believe you can adapt this repo to other GTFS feeds with minor modifications.

If you're interested in controlling this clock with Home Assistant, I include my HA configuration here as well.

Great thanks to Edd Abrahamsen-Mills @eddible for his TFGM Metrolink Clock project <https://github.com/eddible/tfgm-tram-clock>.

## Used hardware

* [Hyperpixel 2.1 Round](https://shop.pimoroni.com/products/hyperpixel-round?variant=39381081882707)
* [Mounting Screws](https://shop.pimoroni.com/products/short-pi-standoffs-for-hyperpixel-round?variant=39384564236371) - Optional
* [Raspberry Pi Zero 2 WH](https://shop.pimoroni.com/products/raspberry-pi-zero-w?variant=39458414297171)
* Micro SD Card - Use a high endurance card, cheap ones didn't last as long in my experience.
* [3D Printed Case](https://cults3d.com/en/3d-model/gadget/sphere-enclosure-w-bump-legs-m3o101-for-pimoroni-hyperpixel-2-1-round-touch-and-raspberry-pi)
* Angled micro USB cable - Optional
* USB power plug.

## Guide

### Flash your Micro SD Card

* Download [Raspberry Pi OS 32-bit Buster Lite](https://downloads.raspberrypi.org/raspios_oldstable_lite_armhf/images/raspios_oldstable_lite_armhf-2023-05-03/)
* Download the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and install it on your computer.
* Launch it with your Micro SD Card plugged into your computer.
* Click `Choose OS` → select the OS file you downloaded earlier.
* Click `Choose Storage` and select your Micro SD card.
* Click `Next` and set following configurations on the next page.
  * `Enable SSH` → `Use password authentication`.
  * `Set username and password` → Leave the username as `pi` (if you must change it, you'll need to update `metro.sh`) but set a password.
  * `Configure wifi` → Enter your Wi-Fi details.
  * `Save` when you're done
* Click `Write` and wait for the OS to be written to your SD Card
* Insert it into your Pi Zero 2 W, connect it to power and give it a few minutes to complete the first boot.  

### Get your stop and route ids

* Get your stop and route ids from here <https://transitfeeds.com/p/helsinki-regional-transport/735/latest/stops>.
* The API endpoints in this repo are taken from here <https://hsldevcom.github.io/gtfs_rt/>

### Installing the software

* SSH into your Pi.
* Install git:

  ```cli
  sudo apt-get update
  sudo apt install git
  ```

* Install the display driver then reboot:

  ```cli
  git clone https://github.com/pimoroni/hyperpixel2r
  cd hyperpixel2r
  sudo ./install.sh
  sudo reboot
  ```

* Download the code and move into the director:  

  ```cli
  git clone https://github.com/mikenhu/hsl-metro-clock
  cd hsl-metro-clock
  ```

* Install required libraries:

  ```cli
  sudo apt install python3-pip -y
  sudo pip3 install -r requirements.txt
  sudo apt-get install libsdl2-mixer-2.0-0 libsdl2-image-2.0-0 libsdl2-ttf-2.0-0 -y
  ```

* You can now configure your config file with your own details. Run these commands:
  * `nano config.ini`
  * Edit the file with your stop ids, route id, and defined direction names as you wish. It should be formatted like this:

    ```ini
    [HSL-CONFIG]
    stop_id_with_names = {"1541602": "West", "1541601": "East"}
    route_id_metro = 31M
    trip_update_url = https://realtime.hsl.fi/realtime/trip-updates/v2/hsl
    service_alerts_url = https://realtime.hsl.fi/realtime/service-alerts/v2/hsl
    ```

  * Once done, press `CTRL+X` → `Y` → `Enter`
* Next we'll set the script to launch at start up. Run these commands:

  ```cli
  sudo cp metro.sh /usr/bin
  sudo chmod +x /usr/bin/metro.sh
  sudo nano /etc/rc.local
  ```
  
* A text editor will open in your terminal window. Use your arrow keys to move to the bottom of the file and create a space above `exit 0` and enter this:
  
  ```bash
  # Fix issue where the Pi restart but sometimes the Hyperpixel does not have screen output
  /usr/bin/hyperpixel2r-init &>/dev/null
  bash metro.sh &>/dev/null
  ```

* To save your changes, press `CTRL+X` → `Y` → `Enter`
* That's it for the software. You can run the metro clock as is. Or...

## Add backlight and Pi controls in Home Assistant

Having the display on all the time is bad, I decided to integrate the metro clock into my home assistant setup. In HA, you can execute commands to control your Pi with command line and shell command integrations.

### Access your Pi from HA

* SSH into your HA host `ssh root@homeassistant.local` or you can use the Terminal add-on on HA.
* Create folder `mkdir /config/.ssh`.
* Create a public key `ssh-keygen`.
* Tell it to store it in `/config/.ssh/id_rsa`. Do not set a password for the key!
* Copy the created key to your Pi `ssh-copy-id -i /config/.ssh/id_rsa pi@[Host]`
* Try to ssh into your Pi from your HA `ssh -i /config/.ssh/id_rsa pi@[Host]` and see if it works without a login.

### Create the switches in HA

Copy my config below and make it match your network setup.

* Command line

```yaml
command_line:
  - switch:
      name: Metro Dashboard Screen
      unique_id: metro_dashboard_screen
      command_off: 'ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i /config/.ssh/id_rsa -o UserKnownHostsFile=/root/.ssh/known_hosts -q pi@[Host] "sudo -E sh -c ''echo 1 > /sys/class/backlight/rpi_backlight/bl_power''"'
      command_on: 'ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i /config/.ssh/id_rsa -o UserKnownHostsFile=/root/.ssh/known_hosts -q pi@[Host] "sudo -E sh -c ''echo 0 > /sys/class/backlight/rpi_backlight/bl_power''"'
      command_state: 'ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i /config/.ssh/id_rsa -o UserKnownHostsFile=/root/.ssh/known_hosts -q pi@[Host] "cat /sys/class/backlight/rpi_backlight/bl_power"'
      value_template: '{{ value == "0" }}'
```

* Shell command

```yaml
shell_command:
  restart_pi: 'ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i /config/.ssh/id_rsa -o UserKnownHostsFile=/root/.ssh/known_hosts -q pi@[Host] "sudo reboot"'
  shutdown_pi: 'ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i /config/.ssh/id_rsa -o UserKnownHostsFile=/root/.ssh/known_hosts -q pi@[Host] "sudo shutdown -h now"'
```

## To do

* Scrolling stop names.
