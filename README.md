# Time Pi -Neil's version

[![CI](https://github.com/njohnsn/time-pi/actions/workflows/ci.yml/badge.svg)](https://github.com/njohnsn/time-pi/actions/workflows/ci.yml)

![PTP server using a Raspberry Pi 5](/resources/PTP_NMJ.jpeg)

A Raspberry Pi stratum 1 time server.

Takes in GPS (or potentially other stratum 0 time sources), spits out NTP, PTP, etc.

## Credit to [Jeff Geerling](https://www.jeffgeerling.com)

## Setup - Grandmaster

### Hardware

To run a decent time server (with high accuracy), you need a few things:

- A computer running Linux (all the nice tooling for network time is available here)
- A high quality time source (GPS is most commonly used these days)
- A network adapter capable of hardware timestamping (Intel and ASIX make some good NICs for time-related applications)

There are many options for each of the above items—for example, many use [Adafruit's Ultimate GPS HAT](https://www.adafruit.com/product/2324) or its [USB equivalent](https://www.adafruit.com/product/4279) for GPS acquisition, and for PTP, you can use a Compute Module 4 or 5's built-in NIC (with [PPS in or out](https://www.jeffgeerling.com/blog/2022/ptp-and-ieee-1588-hardware-timestamping-on-raspberry-pi-cm4)), or add on a compatible NIC on the Pi 5 with something like the [uPCIty Lite](https://amzn.to/4iUn9ke).

My own hardware configuration—which is the basis for the code in this repository, consists of:

- Computer: Raspberry Pi 5 model B
- Time source: u-blox M8Q (installed on a [Uptronics GPS/RTC HAT](https://store.uputronics.com/products/raspberry-pi-gps-rtc-expansion-board))
- NIC: Intel i226-LM (installed on [GeeekPi P02 PCIe Slot for Raspberry Pi 5](https://a.co/d/0cpXWuVd))

A precise GPS signal for nanosecond-accurate time requires a decent antenna with as clear a view of the sky as possible. Some GPS receivers are better than others, but even USB receivers will do better than NTP!

I use a small [active Multiband GPS antenna](https://www.sparkfun.com/gnss-l1-l5-multi-band-high-precision-antenna-5m-sma.html) for testing, but for permanent installation it is best to mount a [higher-quality GPS antenna](https://www.sparkfun.com/gnss-multi-band-l1-l2-l5-surveying-antenna-tnc-spk6618h.html) outside, clear of obstructions.

### Software

Make sure you have Ansible installed. Copy the following example files and customize them according to your setup:

- `example.hosts.ini` to `hosts.ini`
- `example.config.yml` to `config.yml`

Now run the Ansible playbook:

```bash
ansible-playbook main.yml
```

This playbook will configure:

- [GPSd](https://gpsd.gitlab.io/gpsd/gpsd.html): interface with the u-blox GPS and provide GPS data to other applications
- [Chrony](https://chrony-project.org): NTP server, which also sync GPS time (with Internet NTP server backup) to the system clock
- [Linux PTP](https://linuxptp.nwtime.org): synchronize the system clock to the NIC PHC (Physical Hardware Clock), and set up the Pi as a PTP grandmaster clock

## Setup - Client Pis

For client Pis, make sure you have them listed under a `[clients]` heading in your `hosts.ini` file, then run the Ansible playbook:

```bash
ansible-playbook ptp-clients.yml
```

This should configure the clients to acquire PTP time from the grandmaster Pi.

Using u-blox GPS modules, you may encounter a baud rate mismatch. Many newer u-blox modules default to `38400` baud, while older modules default to `9600` baud. This project recommends `115200` baud for slightly faster timing updates.

```bash
# Get the protocol version ('PROTVER')
ubxtool -p MON-VER
```

```text
...
UBX-MON-VER:
  swVersion EXT CORE 4.04 (7f89f7)
  hwVersion 00190000
  extension ROM BASE 0x118B2060
  extension FWVER=SPG 4.04
  extension PROTVER=32.01
...
```

```bash
# Set the version in ubxtool options
export UBXOPTS="-P 32.01"

# Set the baud rate to 115200
ubxtool -S 115200

# Persist the setting
ubxtool -p SAVE -P 32.01
ubxtool -p COLDBOOT -P 32.01

# If running GPSd, update the rate gpsd's config and restart gpsd
sudo nano /etc/default/gpsd
sudo systemctl restart gpsd
```

`ubxtool` is installed as part of the `gpsd-clients` package, which is automatically installed by this playbook.

For more on how to set the baud rate (or tweak other GPS module parameters), see [millerjs.org's ubxtool page](https://wiki.millerjs.org/ubxtool) and the [ubxtool examples](https://gpsd.io/ubxtool-examples.html) page. You can also configure most options via `pygpsclient` using a GUI.

## Debugging

Some handy commands:

```bash
# GPS-related debugging
sudo systemctl status gpsd  # check gpsd status
gpsmon -n  # monitors gpsd output
cgps -s  # also monitors gps output

# PTP timestamping debugging
ethtool -T eth0  # or eth1, lists hardware clock info

# Chrony debugging
chronyc sources -v  # shows sources with documentation of fields
chronyc tracking  # shows detailed timing data

# Check on NTP service from another computer
ntpdate -q [ip of grandmaster]

# Check on PTP clocks and offsets (assuming ptp4l is running)
wget https://tsn.readthedocs.io/_downloads/f329e8dec804247b1dbb5835bd949e6f/check_clocks.c
gcc -o check_clocks check_clocks.c
sudo ./check_clocks -d eth0  # or eth1 (the interface you're using for PTP)

# Check offset between NIC PHY and system clock
sudo phc_ctl eth1 cmp  # should be nearly -37000000000ns

# Use Wireshark's CLI to monitor PTP timestamps
tshark -i en0 -f "udp port 319 or 320" -V -Y "ptp" | grep "OriginTimestamp"

# Use ptp4l to listen for PTP messages
sudo ptp4l -i eth0 -m -s -A
```

Much of the work that went into this project was based on [this thread on the TimeHat v2](https://github.com/geerlingguy/raspberry-pi-pcie-devices/issues/674).

### PTP Debugging

To see how PTP timing is functioning, specifically, you can run the following tools on another computer on the network:

- [ptp-watch.py](resources/ptp_watch.py): Python script to monitor PTP timestamps on the network from any other Linux computer (run with `sudo python3 ptp_watch.py --iface eth0`)
- [Wireshark](https://www.wireshark.org/): Run with filter "ptp", and it will show Announce/Sync/Follow_Up messages, allowing you to inspect the entire structured PTP message.
  - You can also monitor using the built-in CLI, `tshark` (see example command above)
- [PTP Trackhound](https://www.ptptrackhound.com/#/home): A free tool (requires email subscription to Meinberg) that focuses exclusively on PTP traffic, giving more timing-related analytics.

### Debugging with an Oscilloscope for ns-level precision

Once you have PTP set up and multiple computers are in sync, it's useful to see how close the synchronized timing is with an oscilloscope (completely removed from the Linux OS layer).

Assuming all your computers/NICs have PPS outputs, you can connect different oscilloscope channels to those PPS outputs.

Then check if the rising edge of the timing pulse is close—on my LAN, I expect to see well under 100 ns of variance, with 20-30 ns more common. In some scenarios, you can get this down to single-digit ns.

## Slave / Client Setup

For PTP, you need to install and configure PTP for Linux on slave/client machines, and synchronize them to the master/server node as well.

An example configuration for a slave/client node is set up in `ptp-client-node.yml`, and further examples may be provided in the future.

## Other Pi Time projects in this repository

- [Raspberry Pi Pico Mini Rack GPS Clock](./pico-clock-mini-rack/README.md)

## Other Hobbyist Time Servers and Pi Time Builds

- [Austin's Nerdy Things: Nanosecond accurate PTP Pi server](https://austinsnerdythings.com/2025/02/18/nanosecond-accurate-ptp-server-grandmaster-and-client-tutorial-for-raspberry-pi/)
- [Austin's Nerdy Things: Microsecond Accurate NTP for Raspberry Pi with GPS PPS](https://austinsnerdythings.com/2025/02/14/revisiting-microsecond-accurate-ntp-for-raspberry-pi-with-gps-pps-in-2025/)
- [Andreas Spiess: NTP Server from GPS Satellites](https://www.youtube.com/watch?v=RKRN4p0gobk)
- [Time-Card-Flex CM4 and CM5 with OCP-TAP Time Card](https://github.com/regymm/Time-Card-Flex/blob/0637ed49693192d11c7df9ac883c444ce62d2b25/DOC/Setup-and-Usage.md)
- [Conor Robinson's Raspberry Pi NTP Server](https://conorrobinson.ie/raspberry-pi-ntp-server-part-6/)
- [Jeff Geerling: Time Card mini for GPS and OXCO on Pi](https://www.jeffgeerling.com/blog/2023/time-card-mini-adds-pi-gps-and-oxco-your-pc)
- [Raspberry Pi 5 Stratum 1 NTP Server with Uptronics GPS HAT](https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/)
- [BeagleBone Black PRU PPS (IEP hardware timestamping)](https://github.com/dniminenn/bbs-pps-pru)

## Other Time Resources I found Interesting

- [LeapSecond.com](http://www.leapsecond.com) (great resources for timing nerds)
- [Time-Nuts Mailing List](http://www.leapsecond.com/time-nuts.htm) (for amateurs who are interested in precise Time & Frequency)
- [Where does my computer get the time from?](https://dotat.at/@/2023-05-26-whence-time.html) (good overview of the sources of modern NTP + GPS time, with the history of each source)
- [Satpulse time server architecture](https://satpulse.net/2025/05/21/time-server-architecture.html) (System clock vs PHC vs NIC, and how time is transferred internally and through PTP and NTP)
- [pixie Pi 5 GPS Time Server](https://github.com/josh-blake/pixie/wiki) (Guide for improving accuracy with realtime kernel, Ethernet NIC options, etc.)

## License

GPLv3 or Later

## Orignal Author

[Jeff Geerling](https://www.jeffgeerling.com), with assistance from Ahmad Byagowi and Oleg Obleukhov from Meta.

## Forked to document use of Intel i226 and PCIe adapter

[Neil Johnosn](https://erudicon.com)
