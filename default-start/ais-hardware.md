---
description: How to deploy your own Automatic Identification System (AIS) receiver.
cover: ../.gitbook/assets/IMG_3297.jpg
coverY: 0
layout:
  width: default
  cover:
    visible: true
    size: full
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
---

# 📡 AIS Hardware

In addition to utilizing [AIS data provided by Spire](https://spire.com/maritime/?utm_term=spire%20ais%20data\&utm_campaign=Maritime+-+Search+-+Exact\&utm_source=adwords\&utm_medium=ppc\&hsa_acc=4934961383\&hsa_cam=20888450362\&hsa_grp=155753134974\&hsa_ad=685453945378\&hsa_src=g\&hsa_tgt=kwd-922295538895\&hsa_kw=spire%20ais%20data\&hsa_mt=e\&hsa_net=adwords\&hsa_ver=3\&gad_source=1\&gclid=CjwKCAjw74e1BhBnEiwAbqOAjHJkT1RTcEZGEybUWVkwfv9MYvm6ZO-6JOT7diQdXleleG9dHmal1BoCKzcQAvD_BwE) for the Canadian coasts, you can install AIS receiver hardware to capture AIS data directly. The received data can be processed and stored in databases, which can then be used with AISdb. This approach offers additional data sources and allows users to collect and process their data (as illustrated in the pipeline below). Doing so allows you to customize your data collection efforts to meet specific needs and seamlessly integrate the data with AISdb for enhanced analysis and application. At the same time, you can share the data you collect with others.



<figure><img src="https://lh7-rt.googleusercontent.com/slidesz/AGV_vUc6fT3v7BQAiIdFK5InU7VePIbcbiTnRe8xdhnaEyUeJLVPa3eYSrhZH4haXSSalom--oIA8nhXaJLWlgE2pihRVQPzJkyr-4caNB5AiIH5t-6_vMYvtBnd3FthBkpz5JPuuP5sPCMAI5B8eg9Yg5yVNbqfWDiQpH-AOKTHGR44HXIE-Y3H1Ss=s2048?key=y5AqRm_P3gRqgeTbBf2ndA" alt="" width="563"><figcaption><p>Pipeline for capturing and sharing your own AIS data with a VHF Antenna and AISdb.</p></figcaption></figure>

## Requirements

* <mark style="background-color:yellow;">Raspberry Pi or other computers</mark> with internet working capability

<div align="center" data-full-width="true"><figure><img src="../.gitbook/assets/image (27).png" alt="" width="300"><figcaption><p>Raspberry Pi (Image source: <a href="https://www.raspberrypi.com/products/raspberry-pi-3-model-b/">https://www.raspberrypi.com/products/raspberry-pi-3-model-b/</a>)</p></figcaption></figure></div>

* <mark style="background-color:yellow;">162MHz receiver</mark>, such as the [Wegmatt dAISy 2 Channel Receiver](https://shop.wegmatt.com/collections/frontpage/products/daisy-2-dual-channel-ais-receiver-with-nmea-0183?variant=7103563628580)
* <mark style="background-color:yellow;">An antenna in the VHF frequency band (30MHz - 300MHz)</mark> _e.g._ Shakespeare QC-4 VHF Antenna
* Optionally, you may want
  * Antenna mount
  * A filtered preamp, such as [this one sold by Uputronics](https://store.uputronics.com/index.php?route=product/product\&path=59\&product_id=93), to improve signal range and quality

An additional option includes <mark style="background-color:yellow;">**free AIS receivers**</mark> <mark style="background-color:yellow;"></mark><mark style="background-color:yellow;">from</mark> [<mark style="background-color:yellow;">MarrineTraffic</mark>](https://www.marinetraffic.com/en/p/apply-for-free-ais-receiver)<mark style="background-color:yellow;">.</mark> This option may require you to share the data with the organization to help expand its AIS-receiving network.

## Hardware Setup

* When setting up your antenna, place it as high as possible and far away from obstructions and other equipment as is practical.
* Connect the antenna to the receiver. If using a preamp filter, connect it between the antenna and the receiver.
* Connect the receiver to your Linux device via a USB cable. If using a preamp filter, power it with a USB cable.
*   Validate the hardware configuration

    * When connected via USB, the AIS receiver is typically found under `/dev/` with a name beginning with `ttyACM`, for example `/dev/ttyACM0`. Ensure the device is listed in this directory.
    * To test the receiver, use the command `sudo cat /dev/ttyACM0` to display its output. If all works as intended, you will see streams of bytes appearing on the screen.

    <pre class="language-bash" data-line-numbers><code class="lang-bash">$ sudo cat /dev/ttyACM0
    !AIVDM,1,1,,A,B4eIh>@0&#x3C;voAFw6HKAi7swf1lH@s,0*61
    !AIVDM,1,1,,A,14eH4HwvP0sLsMFISQQ@09Vr2&#x3C;0f,0*7B
    !AIVDM,1,1,,A,14eGGT0301sM630IS2hUUavt2HAI,0*4A
    !AIVDM,1,1,,B,14eGdb0001sM5sjIS3C5:qpt0L0G,0*0C
    !AIVDM,1,1,,A,14eI3ihP14sM1PHIS0a&#x3C;d?vt2L0R,0*4D
    !AIVDM,1,1,,B,14eI@F@000sLtgjISe&#x3C;W9S4p0D0f,0*24
    !AIVDM,1,1,,B,B4eHt=@0:voCah6HRP1;?wg5oP06,0*7B
    !AIVDM,1,1,,A,B4eHWD009>oAeDVHIfm87wh7kP06,0*20
    </code></pre>

A visual example of the antenna hardware setup that MERIDIAN has available is as follows:

<figure><img src="../.gitbook/assets/image (21).png" alt="" width="563"><figcaption><p>MERIDIAN AIS hardware setup working at Sandy Cove in Halifax, NS - Canada.</p></figcaption></figure>

## Software Setup

### Quick Start

Connect the receiver to the Raspberry Pi via a USB port, and then run the `configure_rpi.sh` script. This will install the Rust toolchain, AISdb dispatcher, and AISdb system service (described below), allowing the receiver to start at boot.

{% code lineNumbers="true" %}
```bash
curl --proto '=https' --tlsv1.2 https://git-dev.cs.dal.ca/meridian/aisdb/-/raw/master/configure_rpi.sh | bash
```
{% endcode %}

### Custom Install

1. **Install Raspberry Pi OS with SSH enabled:** Visit [https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/) to download and install the Raspberry Pi OS. If using the RPi imager, please ensure you run it as an administrator.
2. **Connect the receiver:** Attach the receiver to the Raspberry Pi using a USB cable. Then log in to the Raspberry Pi and update the system with the following command: `sudo apt-get update`
3. **Install the Rust toolchain**: Install the Rust toolchain on the Raspberry Pi using the following command:  `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`  Afterward, log out and log back in to add Rust and Cargo to the system path.
4. **Install the network client and dispatcher:** (a) From [crates.io](https://crates.io/), using `cargo install mproxy-client`(b) To install from the source, use the local path instead, _e.g._ `cargo install --path ./dispatcher/client`
5. **Install systemd services**: Set up new systemd services to run the AIS receiver and dispatcher. First, create a new text file `./ais_rcv.service` with contents in the block below, replace `User=ais` and `/home/ais` with the username and home directory chosen in step 1.

{% code title="./ais_rcv.service" overflow="wrap" lineNumbers="true" %}
```
[Unit]
Description="AISDB Receiver"
After=network-online.target
Documentation=https://aisdb.meridian.cs.dal.ca/doc/receiver.html

[Service]
Type=simple
User=ais # Replace with your username
ExecStart=/home/ais/.cargo/bin/mproxy-client --path /dev/ttyACM0 --server-addr 'aisdb.meridian.cs.dal.ca:9921' # Replace home directory
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
```
{% endcode %}

This service will broadcast receiver input downstream to _<mark style="background-color:yellow;">**aisdb.meridian.cs.dal.ca**</mark>_ via UDP. You can add additional endpoints at this stage; for more information, see `mproxy-client --help.` Additional AIS networking tools, such as `mproxy-forward`, `mproxy-server`, and `mproxy-reverse`, are available in the `./dispatcher` source directory.

Next, link and enable the service on the Raspberry Pi to ensure the receiver starts at boot:

{% code lineNumbers="true" %}
```bash
sudo systemctl enable systemd-networkd-wait-online.service
sudo systemctl link ./ais_rcv.service
sudo systemctl daemon-reload
sudo systemctl enable ais_rcv
sudo systemctl start ais_rcv
```
{% endcode %}

See more examples in `docker-compose.yml`

## 💡 Common Issues

For some Raspberry hardware (such as the author's Raspberry Pi 4 Model B Rev 1.5), when connecting dAISy AIS Receivers, the device file in Linux used to represent a serial communication interface is not always "/dev/ttyACM0", as used in our `./ais_rcv.service`.

You can check the actual device file in use by running:

```
ls -l /dev
```

For example, the author found that `serial0` was linked to `ttyS0` (i.e., `ttyS0`).

Simply changing `/dev/ttyACM0` to `/dev/ttyS0` may result in receiving garbled AIS signals. This is because the default baud rate settings are different. You can modify the default baud rate for `ttyS0` using the following command:

```
stty -F /dev/ttyS0 38400 cs8 -cstopb -parenb
```
