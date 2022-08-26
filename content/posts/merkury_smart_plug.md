---
title: "Voiding the Warranty - Merkury Smart Plug"
date: 2022-08-23T21:30:03-05:00
draft: true
tags: ["tryharder", "iot", "firmware", "devices"]
---

### The Cause

Studying infosec and IoT device security has given me a desire to create my own gadgets, and what better way than destroying
some cheap off-the-shelf devices that can be found at your local store?  

"But,", some may ask, "why would I do that when there are options such as Arduino, ESP32, and ESP8266 that are available?"

- **Availability**  
    Where I am located, if I want an Arduino board or similar, I have to order it online. After I pay
    for the device, and additional shipping, I patiently wait for my device to arrive in the hopes that the order is correct and device is functional.
    By utilizing devices I can purchase locally I am able to cut out shipping, waiting, and crossing my fingers. Also, these devices are usually sold
    in packs of 2-4 at a cost comparable or cheaper than the cost of a single Arduino (not including shipping cost).

- **Peripherals**  
    Although these devices usually function as simple hardware such as a light bulb or power plug; they also contain a small ARM core processor,
    as well as WiFi and Bluetooth chipsets to provide internet connectivity to enable remote control and timer programming capability.
    Included with this device is a WiFi adapter (capable of acting as access point *and* station simultaneously), Bluetooth capability, GPIO, and I2C!  
    To add such functionality to some Arduino, ESP32, or ESP8266 would require additional parts, which would in turn incur additional cost per project.

- **Versatility**  
    If I can figure out how to flash my own firmware to these devices, I would then have the ability to write my own code, utilize additional libraries,
    and perform functions with the device not intended by the end consumer.

I don't see a reason *not* to at least give it a try.

So, without further adieu, let us break something.

---

### The Victim

The first device I chose to violate is a Merkury Smart Plug. These are currently sold near me for around $13 a pair (yay for spare).  
{{< imagelink src=/img/merkury_smart_plug/box.jpg link=/img/merkury_smart_plug/box.jpg position=center caption="https://www.walmart.com/ip/Merkury-Innovations-Indoor-Smart-Plug-2-Pack/758274409" >}}
  
{{< comment >}} Before teardown photos {{< /comment >}}
|Plug Front| Plug Back|
|:---:|:---:|
|{{< imagelink src=/img/merkury_smart_plug/plug1.jpg link=/img/merkury_smart_plug/plug1.jpg caption="Simple front with LED" >}}|{{< imagelink src=/img/merkury_smart_plug/plug2.jpg link=/img/merkury_smart_plug/plug2.jpg caption="Model:MI-WW218-199WW FCCID: 2AJ3WEBECZW03" >}}|

If we search for the [information and documentation for the given FCCID](https://apps.fcc.gov/oetcf/eas/reports/ViewExhibitReport.cfm?mode=Exhibits&RequestTimeout=500&calledFromFrame=N&application_id=yl2hr7pvnYtSU5TBbzIdXQ%3D%3D&fcc_id=2AJ3WEBECZW03)
we will find disassembled photos showing that the cover is held on by glue around the edge. We can also see the part number for the chipset, but its hard to make out for certain.  
Using two small screwdrivers I was able to pry around the edge until the cover finally popped off. It doesn't look pretty and new, but we're going for
function over form here.  

|Under The Hood|
|:---:|
|{{< imagelink src=/img/merkury_smart_plug/plug_cut1.jpg link=/img/merkury_smart_plug/plug_cut1.jpg >}}|

We can see on the back of this chipset a few interesting labeled contacts such as two pair of `RX/TX`, a `VBAT`, as well as `GND`. These are what we
will be utilizing for communication and flashing of the controller.  

On the front of the chipset we see it is labeld as `Model: WB2S`. A quick Google search will lead us to [the WB2S datasheet provided by Tuya](https://developer.tuya.com/en/docs/iot/wb2s-module-datasheet?id=K9ghecl7kc479).

{{< imagelink src=/img/merkury_smart_plug/chipset_front.jpg link=/img/merkury_smart_plug/chipset_front.jpg position=center caption="Front of controller with Model: W2BS" >}}

A basic rundown of what we have from the datasheet:  
>Overview  
WB2S contains a low-power Arm Cortex-M4 microcontroller unit (MCU), 1T1R WLAN module, 256 KB static random-access memory (SRAM), 2 MB flash memory, and extensive peripherals.  
WB2S is an RTOS platform that integrates all function libraries of the Wi-Fi MAC and TCP/IP protocols. You can develop embedded Wi-Fi products as required.  

The specific features of the device:  
>Features  
Embedded low-power 32-bit CPU, which can also function as an application processor  
Clock rate: 120 MHz  
Working voltage: 3.0 V to 3.6 V  
Peripherals: nine GPIOs, one universal asynchronous receiver/transmitter (UART), and one analog-to-digital converter (ADC)  
Wi-Fi connectivity  
802.11b/g/n  
Channels 1 to 14 at 2.4 GHz  
Support WEP, WPA/WPA2, WPA/WPA2 PSK (AES) ,WPA3 security modes  
Up to +16 dBm output power in 802.11b mode  
EZ net pairing mode for Android and iOS devices  
On-board PCB antenna with a gain of -1.0dBi  
Working temperature: -40°C to +85°C  
Bluetooth LE  
    Support Bluetooth (V4.2)  
    Maximum output power + 6dBm  
    Onboard PCB antenna  

So, it says we have a 120 MHz ARM Cortex-M4 mpu, 256KB RAM, 2MB flash, 2.4Ghz BK7231T wireless chipset, Bluetooth LE, GPIO, UART, and ADC.
I don't think thats a bad deal for about $6.50 total.  

Even though the connections are thankfully labeled on the chip, the datasheet also includes a diagram as well. Thanks Tuya!  

|WB2S Diagram|
|:---:|
|{{< imagelink src=/img/merkury_smart_plug/contact_diagram.png position=center link=/img/merkury_smart_plug/contact_diagram.png caption="Source: https://developer.tuya.com/en/docs/iot/wb2s-module-datasheet?id=K9ghecl7kc479" >}}|

---

### Making Connections  

Since this is our first time (or *mine* at least), to do any work with firmware and devices, I took to Google again. Surely someone else
has tried to work with these devices, right? After reading through a handful reddit and forum posts asking about the Tuya WB2S, I stumbled onto
a forum post at [Elektroda](https://www.elektroda.com):  

- [WB2S/BK7231 Tutorial - writing custom firmware - UDP/TCP/HTTP/MQTT @ elektroda](https://www.elektroda.com/rtvforum/topic3850712.html)

In the discussion the author describes exactly how he was able to connect the device, read the log output, and flash the device with custom firmware.
This sounds to me to be pretty spot-on for what we're needing.  

Tuya has also made the SDK for the chipset open source availble [here](https://github.com/tuya/tuya-iotos-embeded-sdk-wifi-ble-bk7231t.git).  

While searching for more information on the SDK and BK7231T chipset, I found:  
- [Light jailbreaking: exploiting Tuya IoT devices for fun and profit](https://rb9.nl/posts/2022-03-29-light-jailbreaking-exploiting-tuya-iot-devices/)  

In the write-up, the researchers describes how they went about finding a vulerability with the Tuya SDK allowing them to jailbreak the device and install
custom firmware with OTA update. I was later able to find [the video](https://youtu.be/uZXSMUp2bgU) from their presentation at [MCH2022](https://mch2022.org/)
regarding the vulnerability and after their disclosure to Tuya which I think is worth the watch.

This thing is going to be fun!

Now, getting back to business, we need to be able to handle the plug without risk of getting bit by the AC portion of the device while also having access for wiring,
so I removed a section of the case for access. After modifying the case to accomodate wiring, we need to attach leads to the following connections on the chip: 

- VBAT 3.3v power  
- GND ground  
- RX1/TX1 uart programming  
- RX2/TX2 uart logging
- CEN reset

The contact pads are very delicate, so it is recommended to make sure the wiring is extra stable before moving. If one of those pads pulled even slightly it can
separate from the PCB and it may take a bit of finesse to make a new lead on the board. I experienced that with another chipset from a bulb, so I decided to reinforce
the connection with hot-glue for a little extra insurance.

Modified Case Installed|Ugly But Works|
|:---:|:---:|
|{{< imagelink src=/img/merkury_smart_plug/plug_cut2.jpg link=/img/merkury_smart_plug/plug_cut2.jpg >}}|{{< imagelink src=/img/merkury_smart_plug/ugly_but_works.jpg link=/img/merkury_smart_plug/ugly_but_works.jpg >}}

Now we need a way to connect this device to our computer. What I used were 2x [FT232RL FTDI Usb to TTL Serial Adapter](http://www.hiletgo.com/ProductDetail/2152064.html)
I ordered from Amazon for around $6/each.  

For this use since I only needed the VCC/GND and RX/TX from each, I was able to use the pins without doing any more soldering. We will need one FTDI adapter
for the logging console, and the other to do the programming. The CEN lead will be left open for now as it touched to GND to reset the chip.

|FT232RL Front|FT232RL Back|
|:---:|:---:|
|{{< imagelink src=/img/merkury_smart_plug/ft232rl_front.jpg link=/img/merkury_smart_plug/ft232rl_front.jpg >}}|{{< imagelink src=/img/merkury_smart_plug/ft232rl_back.jpg link=/img/merkury_smart_plug/ft232rl_back.jpg >}}|

|Device to Logging Serial Adapter|Device to Programming Serial Adapter|
|:---:|:---:|
|              |VBAT    ->  VCC|
|              |GND     ->  GND|
|2RX    ->   TX|1RX     ->   TX|
|2TX    ->   RX|1TX     ->   RX|

I scavenged a reset switch from an old Dell desktop case to make life easier with resets, but it is not required.

|Wired Up|Reset Switch|
|:---:|:---:|
|{{< imagelink src=/img/merkury_smart_plug/connect1.jpg link=/img/merkury_smart_plug/connect1.jpg caption="After connecting the logging serial adapter." >}}|{{< imagelink src=/img/merkury_smart_plug/connect2.jpg link=/img/merkury_smart_plug/connect2.jpg caption="My Final setup including a reset switch scavanged from an older Dell desktop case.">}}|

---

### Device Log Setup  

Now it is time for us to test our connections. First, we will connect the FTDI that is connected to 2RX/2TX to the computer. For each of my serial adapters I used a cable to connect
instead of plugging directly into the USB port. Once connected, we need to figure out what serial port it is using.  
 

{{< comment >}} Dumping firmware {{< /comment >}}
{{< imagelink src=/img/merkury_smart_plug/putty_config.png link=/img/merkury_smart_plug/putty_config.png position=center caption="COM3 to FTDI connected to UART1" >}}

{{< imagelink src=/img/merkury_smart_plug/putty_boot.png link=/img/merkury_smart_plug/putty_boot.png position=center caption="Merkury smart plug boot log" >}}

---

### Firmware Dump Setup  

```shell
git clone https://github.com/OpenBekenIOT/hid_download_py.git
```

{{< imagelink src=/img/merkury_smart_plug/hid_download_py.png link=/img/merkury_smart_plug/hid_download_py.png position=center caption="Clone https://github.com/OpenBekenIOT/hid_download_py.git repository" >}}

```shell
python -m venv venv
source venv/Scripts/activate
```

{{< imagelink src=/img/merkury_smart_plug/python_venv_create.png link=/img/merkury_smart_plug/python_venv_create.png position=center caption="Setup python virtual environment" >}}

{{< comment >}} Additional python modules install {{< /comment >}}
The `uartprogram` script requires a few additional modules that we don't yet have setup in our virtual environment, so we're likely to get errors if we attempt to run it as is.
We can utilize the Python pip command to install the missing modules:  

```shell
pip install pyserial
pip install tqdm
```

---

<< TESTING READ >>
{{< imagelink src=/img/merkury_smart_plug/uartprogram_usage.png link=/img/merkury_smart_plug/uartprogram_usage.png position=center caption="`uartprogram` program usage" >}}

```shell
python ./uartprogram -d COM6 -b 115200 -r ./firmware.bin
```

{{< imagelink src=/img/merkury_smart_plug/firmware_read.png link=/img/merkury_smart_plug/firmware_read.png position=center caption="Reading the device application firmware" >}}

{{< imagelink src=/img/merkury_smart_plug/firmware_read_success.png link=/img/merkury_smart_plug/firmware_read_success.png position=center caption="Completed firmware read test" >}}