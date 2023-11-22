+++
title = 'Hello World'
date = 2023-11-22
draft = false
series = "ESPwn - Hacking Wi-Fi networks with 20$ of hardware"
+++

# ESPwn - Introduction

I recently bought an Espressif ESP32 board and was looking for a little hacking project to learn its capabilities. I had already heard of devices Wi-Fi sniffing and deauthentication attack, so I decided it would be quite a nice endeavour to try and make one myself. Through this series of articles, you will learn about Wi-Fi protocol and vulnerabilities, esp32 programming, electronics, and even a little bit of reverse engineering. Without further ado, let's dive into the rabbit hole.

## Part 1 - A little bit of Wi-Fi theory

In order to hack Wi-Fi, we must first understand Wi-Fi, so in this article we will be doing all the necessary theory for the rest of the series. 

### Wi-Fi - What is it really ?

Wi-Fi is a set of wireless network protocols from the IEEE 802.11 standards. There is quite an expansive list of protocols (see [Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11#Protocol)) based on the version and the type of network. For instance, your home router might use the 802.11ac standard (Wi-Fi 5). 

As we will see a little bit later, the protocol version is not negligeable at all. Each new version tends to come with additionnal security, which can thwart our efforts.

### The basics

As you probably already know, there are two kinds of Wi-Fi devices : Access Points (APs) and Clients. The clients connect to an AP to have access to the rest of the network.

In order for the clients to know about the APs, the latter periodically transmit their informations (we will see later how). 

### Band and channel

It is important to understand how Wi-Fi physically works. The Wi-Fi antenna in your router or phone can only emit on a selected number of bands (you might have heard of 2.5GHz, 5GHz, and the brand new 6GHz introduced in 802.11ax as Wi-Fi 6E). 

In order to reduce taffic cluttering, each of these band is divided into channels. For the 2.5GHz band, there are 14 channels (1 to 14), with only the first 11 being commonly used. 

Each Wi-Fi device has to choose a channel to communicate, and doing so it cannot see what goes on on the other channels. Typically, the APs choose the less cluttered available channel, while the clients stick to the channel of the AP they are connected to.

![channels.png](images\c8ce5b2e-90c0-41d2-b3fa-10c6403f5824.png)

### Terminology : Mac, SSID and BSSID

On a Wi-Fi network, each device is recognized by its MAC address (6 bytes). This address is supposed to be unique to the device, and is set by the network card. 

SSID is the logical name of the network, advertise by the AP and set by the AP administrator. It is a 32 characters string unique to the network. If two APs set the same SSID (and password), a client can connect to the AP it wants (generally the one with better signal strength). This feature allows roaming in Wi-Fi networks.

BSSID is the AP MAC address on its network. In the case where the AP is a router, the BSSID is not to be confused with the router mac address on the network. For instance, an router AP can have a MAC address for its ethernet interfact connecting it to the network, a BSSID for its 2.5GHz Wi-Fi and another slightly different BSSID for its 5GHz Wi-Fi.  

### Wi-Fi frame basics

Wi-Fi devices communicate with each other by sending frames over the air. Those frames have an established format :

- A MAC header of variable length, which includes a Frame Control field
- A body
- A FCS (Frame Control Sequence)

![wifi_frame.png](/images/d042e29a-4fa0-4748-8f72-743e6d9864ec.png)

#### MAC Header

The MAC header has at least 5 fields : 

- Frame Control - Frame Control contains important informations about the frame (see next section).
- Duration - The duration is the amount of time the sending radio reservers for pending acknowledgement frame.
- Address 1 - Destination MAC address.
- Address 2 - Source MAC address.
- Address 3 - BSSID.

#### Frame Control

![frame_control.png](/images/3e837ab1-7fda-41db-8a5c-3e83804499c1.png)

Frame control is a set of flags that give informations on the frame. Most important part of the frame control are bits 2 to 6. Bits 2 and 3 indicate the frame type, while bits 4 to 7 indicate the subtype.

### Wi-Fi frame types

There are 4 types of frame :
- Management (type 00) - Used for control and management of the Wi-Fi authentication.
- Control (type 01) - Used for management of data frames delivery (acknowledgements etc.).
- Data (type 10) - Used to carry the actual data.
- Extension (type 11) - Other, rarely used.

For each of those frame types, there are up to 16 frame subtypes. However only some of theme matter to us. Here is a little description of the ones we will use (see [Wikipedia](https://en.wikipedia.org/wiki/802.11_Frame_Types#Types_and_SubTypes) for a complete list).

| Frame type          | Type value | Subtype value |
|---------------------|------------|---------------|
|Beacon|00 (management)| 1000|
|Authentication|00 (management)|1011|
|Association request|00 (management)|0000|
|Association response|00 (management)|0001|
|Quality of Service Data|10 (data)|1000|

Beacon frames are sent by access points to advertise their presence on a channel. They contain important information used in the association process such as the authentication method or the SSID. 

The role of the other frames will be explained in more details in the chapter about authentication. 

### Conclusion

In this first part of our series on Wi-Fi hacking with an ESP32 board, we've covered the basics of Wi-Fi protocols, bands and channels, along with key terminology like MAC, SSID, and BSSID. We explored the structure of Wi-Fi frames, focusing on the MAC header and Frame Control, laying the groundwork for understanding Wi-Fi authentication.

In the next part, we'll go for an in-depth explanation of the Wi-Fi authentication process.

## References
[1] https://en.wikipedia.org/wiki/IEEE_802.11
[2] https://www.makeuseof.com/wi-fi-router-channels-explained-what-do-they-all-do/
[3] https://en.wikipedia.org/wiki/802.11_Frame_Types
[4] https://www.atera.com/blog/computer-terms-unwrapped-what-is-bssid/