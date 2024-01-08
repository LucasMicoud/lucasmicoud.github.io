+++
title = 'ESPwn Part 2 - Wi-Fi Authentication'
date = 2023-11-23
draft = false
series = "ESPwn - Hacking Wi-Fi networks with 20$ of hardware"
+++

# Part 2 - Wi-Fi Authentication

After having seen in the last article the basics of Wi-Fi, let's talk about the Wi-Fi security features, and vulnerabilities.

## 802.11 Security and WEP

A client can be in three different states :

- Not authenticated nor associated
- Authenticated but not associated
- Authenticated and associated

The authentication process was used in the original 802.11 protocol and supports two modes: **open** mode (basically no authentication) and **shared** mode, used in Wired Equivalent Privacy (WEP). 

### A word on WEP

WEP relied on RC4 to ensure confidentiality. RC4 is a proprietary symmetric stream cipher. It is initialized by a 24-bit IV concatenated with the 40-bit network secret key. The RC4 algorithm then generates a keystream that is xored with the plain text data of the frames.

WEP was broken in 2001 when a paper ([2](#References)) showed that with a sufficient amount of packets, the RC4 key can be retrieved and was deprecated in favor of WPA.

## WPA - The need for more security

Wi-Fi Protected Access (WPA) is the successor of WEP. To ensure backward compatibility, it has been deployed in an additional layer of authentication, **Extensible Authentication Protocol (EAP)** that happens after the 802.11 authenticate/associate protocol (which, in that case, is in open mode). 

*EAP is an authentication framework that provides various authentication methods used by many protocols, including WPA.*

The standard has seen three versions: WPA, WPA2 and WPA3. Let's review each of these versions.

### WPA

WPA was quickly developed and deployed as a temporary solution to mitigate the impact of the flaws. It is an implementation of the Temporal Key Integrity Protocol (TKIP) protocol, a set of algorithms wrapped around WEP to address some of its flaws (so the protocol still relies on RC4). 

![TKIP.png](/images/36089b38-fabb-4e32-ab85-e6b70ded7d59.png)

Some of the major changes are :
- The mixing of the IV and the key (as opposed to the simple concatenation in WEP)
- A Michael message integrity code 
- The use of a different key for each frame

However, these changes were short-lived and the standard quickly evolved to the more robust WPA 2.

### WPA2

Here comes the interesting part. WPA2 was released in 2004 (802.11i-2004) and was only replaced in 2018 by WPA3. The protocol is still used by many home routers, and it is the one we are targeting with ESPwn.

#### Encryption protocol

WPA2 uses Counter Mode CBC-MAC Protocol (CCMP)  ([3](#References)), which is basically **AES-CBC-MAC** with a 128 bits key. Each frame is protected by a temporal key generated from a **session key** and a 48-bit packet number.

#### So many keys

There are two kinds of keys involved in a WPA2 communication: **pairwise keys** and **group keys**. The pairwise keys are used to encrypt traffic between an AP and a specific device, while group keys are used for broadcast and multicast traffic between an AP and multiple devices.

However the protocol would not be very secure if the same key was used for every communication, so we end up with **master keys** and **transient keys**, the former being used a the beginning of a new session to generate a new pair of the latter. 

In more details, we have :
- Pairwise Master Key (PMK). Pairwise key generated from the passphrase (the password you type in your device). It is known by both the AP and the device before the communication.
- Pairwise Transient Key (PTK). Pairwise key generated from the PMK and used to encrypt unicast traffic during a session.
- Group Master Key (GMK). This group key is only known by the AP.
- Group Transient Key (GTK). Group key generated from the GMK and used to encrypt broadcast and multicast traffic during a session.

When authenticating, the AP and the supplicant (device trying to authenticate) have to agree on a PTK without sending it on the air, or it would be vulnerable to sniffing attacks. This is done with a 4-way handshake.

#### Key exchange in WPA2

The **4-way handshake** consists of 4 EAPOL-Key packets exchanged between the AP and the device right after the association is completed. At the end of this process, the supplicant and the AP have to have agreed on the same PTK and GTK.

![4-way handshake.png](/images/ffbf1717-7d22-4dfc-b043-1295b5624f84.png)

Concerning the keys, the four steps of the handshake are :

- The AP sends an Anonce (Authenticator Nonce) to the supplicant. The supplicant uses the PMK and the Anounce to generate the PTK.
- The supplicant sends an Snonce (Supplicant Nonce) to the AP, which the AP uses alongside the PMK to generate the same PTK.
- The AP sends the GTK to the supplicant.
- The supplicant acknowledges to the AP.

Once this exchange has taken place, the client and the supplicant can securely send and receive frames.

## Attacking WPA 2

### Attack principle

Now that we know about the connection process, we can try to attack it. As we know the algorithm, if we manage to intercept the frames of the 4-way handshake, we can try to brute-force the AP passphrase. 

However, this means that we have to be listening at the exact time a device is trying to connect to the AP, which isn't convenient. To circumvent this problem we can take advantage of a Wi-Fi feature: the de-authentication frames.

Those frames are normally sent by the AP or the device to inform the other the connection has been terminated. When a device receives a de-authentication frame, it will usually try to reconnect, triggering another 4-way handshake. 

Those frames are management frames, meaning that they are not encrypted. See the problem? We can broadcast de-authentication frames spoofing the AP's MAC address, and as those are not encrypted, each connected device will accept it and trigger the re-authentication process. We can now intercept the frames of the 4-way handshake and try to crack the passphrase.

### About the feasibility of brute-forcing the password

Nowadays ISPs are aware of the risks of these attacks and usually deliver their Wi-Fi boxes with strong and long passphrases. However, in some cases, the passphrases are borderline insecure. For instance, my ISP-provided passphrase is 12 characters long, with only lowercase letters and numbers, which is far from unfeasible to crack.

## Conclusion

In this second part of our series on Wi-Fi hacking with an ESP32 board, we've understood Wi-Fi authentication, specifically WEP and WPA 2, and we've described the attack we will implement in the next article.


## References
- [1] Cisco.com [802.11 Network Security Fundamentals](https://www.cisco.com/en/US/docs/wireless/wlan_adapter/secure_client/5.1.0/administration/guide/C1_Network_Security.html)
- [2] Fluhrer, S., Mantin, I., & Shamir, A. (2001) [Weaknesses in the Key Scheduling Algorithm of RC4](https://www.cs.cornell.edu/people/egs/615/rc4_ksaproc.pdf)
- [3] IEEE 802.11i-2004 [Amendment 6: Medium Access Control (MAC) Security Enhancements](https://sci-hub.hkvisa.net/10.1109/ieeestd.2004.94585)