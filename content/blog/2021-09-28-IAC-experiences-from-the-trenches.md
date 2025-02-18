---
title: "Indy Autonomous Challenge (IAC): Experiences from the Trenches"
date: 2021-09-28
menu: "blog"
weight: 20210928
description: "28 September 2021 -- Paris."
draft: false
---

The [Indy Autonomous Challenge](https://www.indyautonomouschallenge.com/) is a competition of autonomous racecars between teams of university students. Even if fully autonomous, each car needs to communicate with its team’s base station to report telemetry, status and to receive commands, such as emergency stop. The communication infrastructure between the cars and the base stations leverages [CISCO Ultra-Reliable Wireless Backhaul (CURWB)](https://www.youtube.com/watch?v=EFvkFx1lwoY&ab_channel=Cisco). As all cars share the same infrastructure, some limitations have been imposed on teams in terms of packet rates and bandwidth usage.

In this blog we present the very first lessons learnt from IAC with regard to ROS2 communication over wireless transports.

All the cars’ software is based on [Autoware](https://www.autoware.org/) + [ROS2](https://www.ros.org/) with [Eclipse CycloneDDS](https://github.com/eclipse-cyclonedds/cyclonedds) which is the default middleware for the latest ROS2 release: [Galactic Geostone](https://docs.ros.org/en/galactic/).

------
## Lesson 1: Avoid DDS traffic over constrained transport
As seen in a [previous blog](../2021-03-23-discovery), the amount of discovery traffic generated by DDS in a ROS2 context with a lot of nodes/publishers/subscribers can be extremely problematic on wireless transports such as WiFi or CURWB. In these deployments, you really want to constrain the DDS traffic to be bound to wired networks or even better not to leave the host. Then, you can transparently rely on **Eclipse zenoh** for communication over the wireless network through the [zenoh/DDS bridge](https://github.com/eclipse-zenoh/zenoh-plugin-dds).

In IAC’s race cars, ROS2 nodes are deployed on a single host -- either in the car or in the base station. Thus, we can instruct CycloneDDS to only use the loopback interface by adding to the configuration file the two lines marked below.

```XML
<CycloneDDS xmlns="https://cdds.io/config" 
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
            xsi:schemaLocation="https://cdds.io/config https://raw.githubusercontent.com/eclipse-cyclonedds/cyclonedds/master/etc/cyclonedds.xsd">
    <Domain id="any">
        <General>
            <NetworkInterfaceAddress>lo</NetworkInterfaceAddress>  <!-- This line -->
            <AllowMulticast>default</AllowMulticast>
            <MaxMessageSize>65500B</MaxMessageSize>
            <FragmentSize>4000B</FragmentSize>
        </General>
        <Internal>
            <Watermarks>
                <WhcHigh>500kB</WhcHigh>
            </Watermarks>
            <AssumeMulticastCapable>lo</AssumeMulticastCapable>    <!-- And this line -->
        </Internal>
    </Domain>
</CycloneDDS>
```

The [`<AssumeMulticastCapable>`](https://github.com/eclipse-cyclonedds/cyclonedds/blob/master/docs/manual/options.md#cycloneddsdomaininternalassumemulticastcapable) option is necessary to workaround some OS (typically Linux) erroneously declaring the loopback interface as not capable of doing UDP multicast.

You can also explicitly activate multicast on loopback on Unix systems doing:
```bash
sudo route add -net 224.0.0.0 netmask 240.0.0.0 dev lo
sudo ifconfig lo multicast
````

------
## Lesson 2: Use the Right Transport for your Data

Most of the data shared between the car and the base station is periodic telemetry. This kind of data is far more important to arrive in time than to be reliably delivered.  It is well known that wireless networks have much higher packet loss rates than wired networks. Wireless networks of quickly moving things, i.e. race cars,  just make that even worse. As a consequence leveraging TCP/IP as an underlying transport for communication between the car and the base station is not a good idea as that will lead to inevitable [congestion control](https://en.wikipedia.org/wiki/TCP_congestion_control) and consequently delay fresh data to repair losses of older telemetry data.  Even worse, in these scenarios TCP/IP is known to suffer from congestion issues related to the acknowledgements and retransmissions packets (that might be lost also). Therefore, on lossy networks and even with an elaborate congestion control algorithm, the usage of TCP/IP can cause undesired spikes of bandwidth consumption and extra-latencies caused by the acknowledgement/retransmission of packets.

In light of these considerations, it should be clear that when leveraging zenoh for communication between the car and the base station the transport protocol of choice should be UDP/IP. 

The configuration the zenoh/DDS bridge to use UDP/IP instead of TCP/IP is quite simple:

 - In the racecar run  
   **`zenoh-bridge-dds --no-multicast-scouting -l udp/0.0.0.0:7447`**

 - In the base station (assuming racecar’s IP is `192.168.1.2`) run  
   **`zenoh-bridge-dds --no-multicast-scouting -l udp/0.0.0.0:7447 -e udp/192.168.1.2:7447`**

Specifically, the configuration options used here are the following:

 - **`--no-multicast-scouting`** : by default the bridge periodically sends messages on UDP/IP multicast to discover other zenoh bridges or applications. As the racecar’s IP is fixed, there is no need for this scouting mechanism, and we deactivate it to avoid useless traffic on the wireless transport.
 - **`-l udp/0.0.0.0:7447`** : this makes the bridge to open a UDP/IP socket instead of a TCP/IP socket for incoming zenoh protocol
 - **`-e udp/192.168.1.2:7447`** : this makes the base station’s bridge to establish a zenoh session with the bridge within the racecar.

Few notes on this:

 1. **No need to start the bridge in the racecar before the bridge in base station**: 
    this one will periodically try to establish the zenoh/UDP session with the provided IP+port until it succeeds.
 2. **No need to configure the bridge in the racecar with the IP of the base station**:
    At session establishment, the bridge in the base station will send its IP+port to the bridge in the racecar. And this one will send messages to this IP+port over UDP/IP.
 3. Those 2 features work not only for the zenoh/DDS bridge but for any Eclipse zenoh application :wink:

-------
## Lesson 3: Pace the traffic !

The traffic between ROS2 nodes can be intense and not compatible with the bandwidth or the message rate limitations imposed by some transports. 

First, not all ROS2/DDS topics need be routed by the zenoh/DDS bridge. You can control the list of topics that are allowed to be routed over zenoh using the bridge’s **`--allow`** option. This option takes a regular expression that should match the topic names to be routed.  
Here is an example of usage to allow a list of (fictitious) topics:  
  - **`--allow “position|speed|levels|race_flags|emergency_stop”`**


Still, you might want to keep high frequency for intra-host publications, but downsampling those publications when routing them over a wireless transport.

For this purpose, we recently introduced a new option in the zenoh/DDS bridge:  
 - **`--max-frequency <String>...`** : specifies a maximum frequency of data routing over zenoh per-topic.  
   The string must have the format *"regex=float"* where:
   - *"regex"* is a regular expression matching the set of ‘topic-name' for which the data must be routed at no higher rate than associated max frequency.
   - *"float"* is the maximum frequency in Hertz; if publication rate is higher, downsampling will occur when routing.
 
This option can be used multiple times for different frequency per-topic.  
For instance, if we want such limtations (topic names are fictitious):  
 - from the racecar, we want to re-publish over zenoh those topics with a maximum frequency for each:
   - `position`: 10Hz
   - `speed`: 10Hz
   - `levels`: 1Hz
 - from the base station we want to re-publish over zenoh those topics with a maximum frequency for each:
   - `race_flags`: 5Hz
   - `emergency_stop`: 20Hz
 
Now the full zenoh/DDS bridge commands will be:  
 - In the racecar:  
   ```bash
   zenoh-bridge-dds --no-multicast-scouting -l udp/0.0.0.0:7447 \
     --allow "position|speed|levels|race_flags|emergency_stop" \
     --max-frequency "position|speed=10" \
     --max-frequency "levels=1"
   ```

 - In the basestation (assuming racecar’s IP is 192.168.1.2):
   ```bash
   zenoh-bridge-dds --no-multicast-scouting -l udp/0.0.0.0:7447 \
     -e udp/192.168.1.2:7447 \
     --allow "position|speed|levels|race_flags|emergency_stop" \
     --max-frequency "race_flags=5" \
     --max-frequency "emergency_stop=20"
   ```

-------
## Conclusion

Communication over wireless networks can be challenging, especially when communicating elements, such as robots, cars and race-cars, move around and more importantly move fast.
The zenoh protocol is well known for having an extremely low wire-overhead -- a handful of bytes per data packet!. The zenoh/DDS bridge has demonstrated that any DDS/ROS2 system can transparently leverage zenoh to overcome the challenges posed by R2X communication over wireless transports, Wide Area Networks and constrained networks in general.

We are proud to support the IAC team and look forward to a great competition on Oct. 23, 2021.  
Best of luck to all the teams!   :checkered_flag:



[**--JE**](https://github.com/JEnoch/)
