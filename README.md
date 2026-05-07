t 🎵 Owlie: A DIY RFID Jukebox

<p align="center">
  <img src="img/owlbox.png" alt="Jukebox" width="320">
</p>

Owlie is a simple, kid-proof jukebox powered by a **Raspberry Pi 4**, a PN532 RFID reader, and a pair of speakers. This project uses **Raspeberry Pi OS Bookworm 64-bit Lite** for the device firmware, **Phoeniebox** for playback, orchestration and a user-friendly interface.

##  Features

-   **Tap-to-Play:** Present an RFID tag to start playing a playlist.
-   **Auto-Pause:** Remove the tag, and the music pauses.
-   **Smart Resume:** Present the same tag to resume, or a new tag to play new media.
-   **Tactile Controls:** A rotary encoder controls volume, and buttons handle previous/next track.
-   **Wi-Fi Speaker:** Can also be used as a standard Wi-Fi speaker.

## Hardware & Assembly

For detailed hardware, assembly, and 3D printing instructions, please see the **[Hardware Manual](manual.md)**.

The core components are:
-   Raspberry Pi 4/5 or 2W
-   PN532 RFID reader (SPI recommended)
-   [Waveshare WM8960 audio HAT](https://www.waveshare.com/wiki/WM8960_Audio_HAT?srsltid=AfmBOoqW70ZW3h23JUg6jWNIV9WuVtEOpoivRx90iPtG8A7PR_MYpmdh) or any other Audio DAC/Amp HAT
-   Rotary encoder :  KY-040
-   2 × momentary buttons
-   Passive speakers + 5V/3A power supply
-   [3D-printed enclosure](https://makerworld.com/en/models/1914879)

---

## Software Setup Guide

### Prerequisites

Owlie uses [Phoeniebox](https://github.com/miczflor/rpi-jukebox-rfid) as the main software. The Waveshare audio HAT requires waveshare drivers to be installed also. If you use any other Audio DAC/Amp HAT, follow the specific instructions for the drivers install.

### Step 1: Flash Raspberry Pi OS

Flash onto the SD card `Raspberry Pi OS Bookworm 64-bit lite`.
Note that Phoeniebox is not compatible yet with Trixie, and the Waveshare HAT requires 64bit firmware.

### Step 2: Install the audio HAT

1. Plug the WM8960 HAT
2. Install the drivers, following [waveshare instructions](https://www.waveshare.com/wiki/WM8960_Audio_HAT?srsltid=AfmBOoqW70ZW3h23JUg6jWNIV9WuVtEOpoivRx90iPtG8A7PR_MYpmdh) :
  - Install the prerequisites
``` bash
sudo apt -y update
sudo apt -y upgrade
sudo apt install git
```
  - Install the drivers
``` bash
git clone https://github.com/waveshare/WM8960-Audio-HAT
cd WM8960-Audio-HAT
sudo chmod +x install.sh
sudo ./install.sh 
sudo reboot
```
  - Check if the driver is installed
  ```bash
pi@raspberrypi:~ $ sudo dkms status 
wm8960-soundcard, 1.0, 4.19.58-v7l+, armv7l: installed
  ```
  - Check the soundcard : 
  ```bash
pi@raspberrypi:~ $ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 1: wm8960soundcard [wm8960-soundcard], device 0: bcm2835-i2s-wm8960-hifi wm8960-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
  ```
  - test the sound :
```bash
aplay /usr/share/sounds/alsa/*
```
  - adjust the volume
```bash
sudo alsamixer
```
If WM8960 is not the default sound card, you should press F6 to choose an audio device.

### Step 3: Install Phoenixbox

1.  This project uses Phoenix 2. It should work on Phoeniebox 3 when its full version is released
2.  Follow these [instructions](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/INSTALL) for installation
```bash
cd; rm install-jukebox.sh; wget https://raw.githubusercontent.com/MiczFlor/RPi-Jukebox-RFID/master/scripts/installscripts/install-jukebox.sh; chmod +x install-jukebox.sh; ./install-jukebox.sh
```
3. Follow on-screen instructions for full install
  - select the PN532 I2C integration when prompted
  - activate the gpio control when prompted

### Step 4: adjust the soundcard configuration (notably for volume control)

Following these issues [#1217](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/1217) and [#549](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/549).
To get the sound card WM8960 correctly working :
1. Adjust the `mpd.conf`configuration 
```bash
sudo nano /etc/mpd.conf
```
2. Look for the àudio_output entry that is not commented out. Comment it out et replace it by :
```bash
audio_output {
        type            "alsa"
        name            "WM8960 HiFi HAT"
        device          "hw:CARD=wm8960soundcard,DEV=0"
        mixer_type      "hardware"
        mixer_device    "hw:CARD=wm8960soundcard"
        mixer_control   "Speaker"
}
```
3. Check alsa config :
Run `amixer -c 1 scontrols` (the WM8960 Hi-Fi HAT is card number 1, use `aplay -l` to get the number). You should end up with a long list of 51 simple mixer controls, including "Playback". If you get this long list, your ALSA setting should be fine.

4. Might need to run `sudo systemctl enable mpd.service` if you get `An error occorured: Execution failed Command: /home/pi/RPi-Jukebox-RFID/scripts/playout_controls.sh -c=volumeup Output: RC: .1`

5. Adjust iFace name : `nano RPi-Jukebox-RFID/settings/Audio_iFace_Name` and change `PCM` to `Speaker`

6. Use `alsamixer` to adjust sound. Switch card using `F6`. If you want to keep settings between reboot :

``` bash
sudo alsactl store 1 #1 being the card number
```

### Step 5: Setup PN532 RFID reader

These instructions are for the following RFID reader:

<https://shop.pimoroni.com/products/adafruit-pn532-nfc-rfid-controller-shield-for-arduino-extras>

Similar shields/breakout boards, based on the same chip might work, but have not been tested.  

It has been tested with the I2C interface. Using SPI might work as well, but it has not been tested.

1. Connect the PN532 RFID reader to the GPIO pins

    | PN532 | Raspberry Pi | Raspi Pins |
    | ----- | ------------ | ---------- |
    | 5V    | 5V           |     4      |
    | GND   | GND          |     6      |
    | SDA   | GPIO 2 (SDA) |     3      |
    | SCL   | GPIO 3 (SCL) |     5      |

2. Activate the I2C interface of the Raspberry Pi
    - `sudo raspi-config`
    - Select "5 Interfacing Options" -> I2C -> yes
    - or instead of using the UI, here is the CLI command:
        `sudo raspi-config nonint do_i2c 0`

3. Install I2C tools
    - `sudo apt-get install i2c-tools`

4. Check that the reader is found trough I2C
    - check `sudo i2cdetect -y 1`
    - output should look like this:

                 0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
            00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
            10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
            20: -- -- -- -- 24 -- -- -- -- -- -- -- -- -- -- -- 
            30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
            40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
            50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
            60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
            70: -- -- -- -- -- -- -- -- 

    - if the table is empty, try switching I2C off and on again in raspi-config or reboot

5. Configure experimental reader
   - `cd scripts`
   - `cp Reader.py.experimental Reader.py`
   - Run `python3 RegisterDevice.py`
   - Select 2 (PN532)

6. Restart the phoniebox-rfid-reader service
   - `sudo systemctl restart phoniebox-rfid-reader.service`

### Step 6: Configure GPIO actions

1. Create and run the service
  - The service can be activated during installation with the installscript.
  - If the service was not activated during installation, you can alternatively use `sudo install.sh` in `components/gpio_control`.
2. Configure in the file `~/RPi-Jukebox-RFID/settings/gpio_settings.ini`. Refer to [Phoeniebox documentation](https://github.com/MiczFlor/RPi-Jukebox-RFID/blob/develop/components/gpio_control/README.md#rotaryencoder)
3. Use `sudo systemctl restart phoniebox-gpio-control` to activate new settings

Please note that pins are GPIO numbers

#### For the Rotary encoder

```ini
[RotaryVolumeControl]
enabled: True
Type: RotaryEncoder
Pin1: 22
Pin2: 23
timeBase: 0.1
functionCall1: functionCallVolU
functionCall2: functionCallVolD
```
* **enabled**: This needs to be `True` for the rotary encoder to work.
* **Pin1**: GPIO number corresponding to rotary direction "clockwise" ('CLK')
* **Pin2**: GPIO number corresponding to rotary direction "counter clockwise" ('DT')
* **functionCall1: functionCallVolU**: function called for every rotation step corresponding to rotary direction "clockwise" (custom steps as optional argument).
* **functionCall2: functionCallVolD**: function called for every rotation step corresponding to rotary direction "counter clockwise" (custom steps as optional argument).
* **timeBase**: Factor used for calculating the rotation value base on rotation speed, defaults to `0.1`. Use `0` for deactivating rotation speed influence.

and its button

```ini
[PlayPause]
enabled: True
Type: Button
Pin: 27
pull_up_down: pull_up
functionCall: functionCallPlayerPause
```

The corresponding pinout :
```text
  .---------------.                      .---------------.
  |               |                      |               |
  |           CLK |----------------------| GPIO 22       |
  |               |                      |               |
  |           DT  |----------------------| GPIO 23       |
  |               |                      |               |
  |           SW  |----------------------| GPIO 27       |
  |               |                      |               |
  |           +   |----------------------| 3.3V          |
  |               |                      |               |
  |           GND |----------------------| GND           |
  |               |                      |               |
  '---------------'                      '---------------'
       KY-040                                Raspberry
```

#### For the button next and prev song

```ini
[NextSong]
enabled: True
Type:  Button
Pin: 26
pull_up_down: pull_up
functionCall: functionCallPlayerNext

[PrevSong]
enabled: True
Type:  Button
Pin: 6
pull_up_down: pull_up
functionCall: functionCallPlayerPrev
```

Be careful of overlap with the WM8960 HAT if adding other GPIOs. Here is it's full pinout


|Functional Pins	|Raspberry Pi Pins (BCM)|	Description|
|5V|	5V|	Power positive (5V power input)|
|GND|	GND|	Power Ground|
|SDA|	P2/SDA|	I2C data input|
|SCL|	P3/SCL| I2C clock Input|
|CLK|	P18|	I2S bit clock input|
|LRCLK|	P19|	I2S frame clock input|
|DAC|	P21|	I2S serial data output|
|ADC|	P20|	I2S serial data input|
|BUTTON|P17|	Custom buttons|


### Step 7: configure RFID chips, playlists, music through web-interface

1. Connect to the web-interface : `http://owlie.local` // `http://<ip>`
2. you can add music through the web interface or in `~/RPi-Jukebox-RFID/shared/audiofolders`

### Step 8 : assemble the hardware [Hardware Manual](manual.md)

---


## 🙏 Acknowledgements

-   [Phoniebox](https://phoniebox.de/index-en.html)
-   [Owlbox](https://github.com/XtracT/rfid_jukebox)
