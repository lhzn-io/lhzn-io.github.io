# bosn: An Instrument Hub for a Small Research Boat

Our boat already knows a great deal about where it is and what the water is doing. Its chartplotters, heading sensor, sounder and radio share position, speed, heading, depth and water temperature with one another many times a second, and then the numbers are gone. `bosn` (short for boatswain, the crew member responsible for the boat's equipment) is a small computer that sits on that network, records everything, and makes it available to software ashore and aboard. This post describes it from the ground up: the network it listens to, the hardware in the box, what it does with the data, and the power design we are moving to after two months of use.

![The open bosn enclosure aboard, a yellow Nanuk case on an HDPE plate: the NMEA 2000 bulkhead and micro-C tee at top right with the lever-connector blocks below it; the Mean Well converter in the center right; the Raspberry Pi 5 with the LoRa bonnet and its OLED at lower left beside the 2 TB SSD; the CAN adapter's green terminal and the USB serial cable above the Pi; the WIBL logger's board along the bottom edge; the LoRa antenna through the top wall](/assets/images/bosn/bosn-enclosure-v1.jpg)

*The enclosure aboard, October 22. This photo shows the v1 power supply, the silver Mean Well converter at center right; the wiring diagram below is the v2 design that replaces it.*

## The network on a boat: NMEA 2000

Most modern boat electronics talk over [NMEA 2000](https://www.nmea.org/nmea-2000.html), a CAN bus running at 250 kbit/s. Each device puts messages on the bus identified by a parameter group number (PGN): position, speed and course over ground, heading, depth, water temperature, engine data, AIS targets. A single backbone cable with a terminator at each end runs through the boat, devices tee off it with short drops on round five-pin "micro-C" connectors, and the backbone also carries 12 V to power the smaller devices.

On our boat the bus carries two Garmin chartplotters, a heading sensor, a VHF radio with AIS, a Wi-Fi gateway and the [WIBL](https://github.com/CCOMJHC/WIBL) depth logger from the University of New Hampshire. Together they put about 150 messages a second on the bus underway, about 4 million over a day of nearly eight hours.

## What bosn is

`bosn` is a Raspberry Pi 5 (8 GB) with three jobs:

1. **Listen.** A USB CAN adapter (DSD TECH SH-C31A with candleLight firmware, which Linux sees as a standard SocketCAN interface, `can0`) connects the Pi to the bus. It only listens and does not terminate the bus; the backbone already has its two terminators.
2. **Translate and record.** [Signal K](https://signalk.org/), an open-source marine data server, decodes the NMEA 2000 messages into named, timestamped values (`navigation.speedOverGround`, `environment.depth.belowTransducer` and so on) and serves them over a WebSocket and REST interface. A raw log of every frame goes to a 2 TB SSD, not the SD card, so nothing is lost to decoding choices made today.
3. **Relay.** An Adafruit LoRa Radio Bonnet (RFM95W, in the US 902 to 928 MHz band) receives telemetry from our solar-powered sensor buoy, [lazy-buoy](https://github.com/lhzn-io/lazy-buoy), and publishes it into Signal K as a second source. The bonnet's small OLED shows the hub's state at a glance, and holding one of its buttons shuts the Pi down cleanly.

`bosn` also takes the WIBL's serial console through a 3.3 V USB serial cable (ground, transmit and receive only, so the logger still runs from bus power alone). That lets it read the logger's status and, since October 31, its motion sensor.

## The box

Everything lives in a Nanuk 909 case shared with the WIBL logger. One micro-C bulkhead brings the bus in through the case wall to a tee: one branch feeds the WIBL, the other a short pigtail cut from a male-to-male cable, which breaks out the bus's four conductors inside the case.

- **Power** from the bus's 12 V passes a 2 A blade fuse, with a TVS diode (P6KE18A) across it to clamp surges, into lever-connector blocks that serve as the positive and common ground nodes.
- **CAN-H and CAN-L** go to the adapter's screw terminal as a twisted pair; the shield's drain wire is insulated and left unconnected, because the bus grounds it at its power tap.
- **The parts** mount on a 6.35 mm HDPE plate cut to the case floor, on standoffs, so nothing is screwed into the case itself. A second bulkhead carries the LoRa antenna lead out to an external antenna.

![bosn wiring schematic, Rev C: the NMEA 2000 drop through the bulkhead to a micro-C tee feeding the WIBL logger and a pigtail; the pigtail's +12 V through a 2 A fuse and TVS diode into the supercapacitor UPS, which powers the Raspberry Pi 5 over USB-C and signals it through pogo pins; CAN-H and CAN-L to the SH-C31A adapter, which connects to the Pi over USB](/assets/images/bosn/bosn-wiring-rev-c.png)

*Wiring, Rev C (the v2 power design described below). Everything ties to one common ground node, which is the CAN reference and the Pi's ground.*

## What the data is for

Each night the SSD's logs sync ashore, where a pipeline splits them into sessions (trips, dock days, bench tests) and builds a page for each. For trips with phone video from the helm, a ride page plays the video beside a map that follows the boat over the seafloor relief, with the instruments alongside and the forecast wind and wave crests for that hour. The sounder's depth is compared with a 10 m seafloor elevation model at the observed tide, which checks both the sounder and the model: on November 1 past Cold Spring Harbor Light they agreed to a median of 0.1 m. The [field note on measuring the ride](https://longhorizon.eco/field-notes/measuring-the-ride) describes what this has shown so far.

## Power, version 1, and what it taught us

The first design powered the Pi from the bus through an isolated 12 V to 5 V converter (Mean Well RSD-30G-5, rated for railway use, 9 to 36 V in). The hub draws 4.0 W at 12.5 V idle, about 8 Ah a day. Two months aboard exposed three problems.

1. **Low 5 V.** The converter has no output trim, and it holds the Pi's 5 V rail at 4.92 V idle. The Pi flags under-voltage (`throttled=0x50000`) on some boots.
2. **Engine cranking.** On November 1 the voltage dip as the engine cranked crashed the hub with a kernel panic; that ride's data was recovered from the WIBL's own log ([the ride](https://www.youtube.com/watch?v=HS7JQ8Rz5V0)).
3. **Hard power cuts at the dock.** The hub runs from the house batteries, which are switched off when we leave the boat. Over the 12 trips logged between September 30 and November 3, the logs end between 7.7 and 31 minutes after the boat stops at the dock, each time with the Pi still writing.

The switch-off is also information: it means the trip is over. A hub that dies at that moment cannot report it.

## Power, version 2: the battery switch as a signal

The goal: when 12 V drops, the hub gets about a minute of hold-up, learns that input power is gone, sends one message, shuts down cleanly, and boots by itself the next time the batteries come on. The same hold-up should carry it through an engine crank.

**Options we ruled out.**

- *A USB power bank between the converter and the Pi.* Many banks drop their output for a fraction of a second while they renegotiate USB power delivery as the input comes or goes, which resets a Pi. Whether a bank that has run flat resumes output on its own when charging power returns varies by model and is rarely documented. And a Pi 5 that halts with 5 V still applied stays halted until its power button is pressed or power is removed and restored ([Raspberry Pi documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)), so a bank that keeps supplying 5 V after a clean shutdown means the Pi never sees power return.
- *Lithium UPS boards.* Several Pi 5 UPS boards hold 18650 or 21700 cells, detect power loss and restart the Pi automatically. The hub sits in a sealed case in a boat that can reach 50 to 60 C in summer sun, above the charging range of most lithium cells, and none of the boards we found publish an operating temperature.

**The design.** We chose a supercapacitor UPS built for the Pi 5, the Geekworm UPS_Scap_5V5A (100 F version). Supercapacitors tolerate heat far better than lithium cells, and a minute of hold-up is all the job needs.

- **Power path.** The board takes 9 to 24 V directly from the bus, so it replaces the converter, and it supplies 5 V at 5 A with the Pi 5's USB-PD handshake. The Pi then allows its USB ports their full current, which matters with the CAN adapter and the SSD on USB.
- **Mounting.** It has the Pi's 85 x 56 mm footprint and mounts under it. Its signal contacts are spring-loaded pogo pins on its top face that press on the underside of the Pi: header pins 3 and 5 (I2C, for an INA219 current monitor at address 0x40), pin 37 (GPIO 26, high while input power is present), and the Pi 5's power button pads. The 40-pin header stays free, so the LoRa bonnet stacks on top as before.
- **Hold-up.** At the hub's draw, about 0.7 A at 5 V, the vendor's figures (88 to 101 s at 1 A) suggest about two minutes.

![Side view of the Rev C stack: HDPE plate, 18 mm standoffs, the UPS board with its supercapacitors below it, 9 mm standoffs, the Raspberry Pi 5 with the active cooler, and the LoRa bonnet with its OLED on the 40-pin header; pogo pins reach the Pi's underside at the I2C pins, GPIO 26 and the J2 power button pads](/assets/images/bosn/bosn-stack-rev-c.png)

*The v2 stack, side view. Heights are estimates from the vendor's assembly photos; the top of the stack sits about 55 mm above the case floor in a 93 mm interior.*

**The shutdown service.** Pogo pins against solder joints can chatter on a boat running at 26 knots through chop, and a chattering power-loss line must not shut the hub down underway. So the service acts only when two independent signals agree: GPIO 26 stays low for 2 s, and the INA219 shows the input gone. Then it sends its message, sets an RTC wake alarm, and halts. The UPS drains and cuts its output, and the Pi boots when the batteries come back on.

The wake alarm covers a known gap: the vendor documents a 1 to 2 s window during shutdown in which restored power can leave the Pi off until the next power cycle. A Pi 5 boots from halt on an RTC alarm with `POWER_OFF_ON_HALT=1` and `WAKE_ON_GPIO=0` in its bootloader configuration and the RTC backup battery fitted, so an alarm set a few minutes ahead boots it either way. `PSU_MAX_CURRENT=5000` stays set.

## What's next

The v2 design is not built yet. Before it goes aboard, the bench test covers the hold-up time at idle and under load, how quickly the power loss is detected, ten power cycles (one restored during the shutdown window), an engine-crank dip reproduced on the bench supply, a tap-and-shake test of the pogo contacts, and the Pi's 5 V rail against v1's 4.92 V. We will report the results in a follow-up post.

With a clean signal at the end of every trip, the hub can close out its own records: finish the session log, sync it ashore, and mark the trip complete. The WIBL's motion records, now logged on every trip, will let us score each ride against the forecast for the same water.
