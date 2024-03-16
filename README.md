# Vodafone UK VOIP Residential FTTP Configuration notes
A reference for using your own VOIP hardware (Grandstream WP810) with Vodafone UK Residential FTTP + OPNSense

As of writing Vodafone ships a router with a built in component that will allow you to connect your existing PTSN phone to the Vodafone VOIP network. There is no sound technical reason to prevent you from using your own VOIP hardware via your own router. However because money, Vodafone would rather residential customers be unable to use their own equipment. [Here it is in their own words.](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2744786/highlight/true#M3568):
  ```
    “As this is a consumer Home Broadband service and not a business service where this is explicitly provided, Vodafone are unable to offer this as a service to the customer”
  ```

## Sources
Most of the information contained here is from [this thread](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2709457) on the vodafone forums.

This sprawling thread is essentially the authority on the subject. I'm going to condense it down for my own reference and deploy a WP810. In the hopes that this may help someone else, here are my notes. All credit goes to the members there sharing their hard work. 

## Configuring PFSense & OPNSense for VOIP via NAT

This section is only relevant if you are using OPNSense or PFSense as your router. If you are using different firmware you will need to find another way to achieve the following, though the basic principle should be the same. This section should also apply to other VOIP providers.

SIP uses RTP to transport audio data. By default, NAT rewrites the source port of traffic to enable multiple outbound connections to use the same source port, as no two devices can know which ports are in use. This causes a problem for SIP as described [here](https://www.sonicwall.com/support/knowledge-base/troubleshooting-a-scenario-where-source-remap-is-causing-the-voip-issues/170504967157192/)

There are three ways to work around this, only one of which is safe.

Method one is to put your phone in a DMZ which will fully expose it to the internet. This is bad for obvious reasons, but there are lots of posts telling you to do this to make VOIP work. **Don't do it, under any circumstance.**

Method two is to port forward the relevant ports. This is ugly and it may appear to work. It allows anyone on the internet to send packets to your phone and you may have to forward a range of ports which now won’t work for other devices causing connectivity issues. There are also lots of instructions telling you to do this. **Don't do it, it will break more than your VOIP. You're essentially unleashing a chaos monkey.**

Method three is to use Outbound NAT rules to disable source port rewriting for packets originating from your VOIP phone(s). This does not expose your phone and safely allows RTP to traverse the NAT while reducing the chance of a port conflict. 

+ Step 1: Create a host alias that references your phones IP or hostname.

+ Step 2: Create a port alias that references the SIP port(5065 for Vodafone, may vary for other providers) and the RTP port range. The instructions below specify the starting port as 10000 with a range of 200. However usually RTP uses a range of 10000-32767.

+ Step 3: Apply a firewall rule(s) to the relevant LAN interface that allows OUTBOUND connections from the host alias we set up in step 1. Depending on your configuration you will need to allow TCP/UDP connections via the port ranges specified in the alias we created in step 2.

+ Step 4: Go to Firewall -> NAT -> Outbound and select "Hybrid outbound NAT rule generation".

+ Step 5: Go to Firewall -> NAT -> Outbound and add a manual rule, Interface: WAN Source:(The alias we created in step 1) Source Port: (The port alias we created in step 2) Destination:* Destination Port: tcp/udp/* NAT address: Interface Address NAT Port * Static Port: YES

“Static port” is the option that disables source port rewriting for connections that match the rule.

 <sub>This information is based on instructions from [here](https://www.3cx.com/docs/pfsense-firewall/) and [here](https://www.reddit.com/r/opnsense/comments/16n2fr3/voipdectbasestation_behind_opnsense_firewall/?rdt=52318). Note that the 3CX document counter-intuitivly refers to "port mapping" in the context of outbound NAT traversal, rather than the more common inbound port to host mapping. </sub>

#### RTP Port Range Considerations

RTP by default uses a random UDP port from 10000-32767. The phone is unaware of which source ports are in use by other devices on the local network. If something else is (or has) been using UDP/10000 for example, the call will fail until the relevant state table entry expires. This is the problem that remapping aims to fix after all, and we have disabled it.

Ripshods configuration restricts the RTP port range to 20 from a base port of 10000. In cases where you are unable to dynamically limit the scope of source port remapping(ie. you're not using opnsense/pfsense) this limits the ports that you 'lose' to the workaround and decreases the chance of a conflict. However this is suboptimal and unnecessary when using Outbound NAT rules. A bigger range decreases the chance of a conflict and does not impact other devices.  Relying on a single port or small pool increases the chances of port conflicts and intermittent connectivity issues.

## WP810 Configuration

Ripshod very kindly posted screenshots of his configuration [here](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2734408/highlight/true#M2802)
for download from [here](https://drive.google.com/drive/folders/1aquAVMgOeln9x0-O_nRCGQL-dzxZrFhi?usp=sharing)

I followed these settings almost exactly as is for my WP810 with one exception. I set "Local RTP Port Range" to 200, a higher number is better for the reasons noted above. I did not put a space between "Vox3.0". The one or two missing options were inconsequential.

If you are using the OPNSense configuration posted above the correct NAT traversal option is as shown in the screen shots: "No." I did provide a stun server address but I was unable to get this to work and instead opted to manually enter my IP. Selecting the STUN option for NAT traversal did not work.

Finally I applied the localisation settings as noted below.

## Known working phones
As of 17/01/2024 [from this post](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2744752/highlight/true#M3565)

+ GXP1620 - Wired Phone (Tested by g7omn)
+ GXP1625 - Wired Phone (Tested by mdbloomfield)
+ HT801 - ATA (Tested by many)
+ HT802 - ATA (Tested by many)
+ HT812 - ATA (Tested by many)
+ WP810 - WiFi Phone (Tested by cf996 & Ripshod)
+ WP822 - WiFi Phone (Tested by Ripshod)
+ Cisco ATA 191
+ Cisco ATA 192

## Why these phones?

As per [this](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2744786/highlight/true#M3568) and [this](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2743365/highlight/true#M3509) post

Vodafone consider using your own VOIP equipment a business feature and have hidden behind an excuse of security to avoid providing anything more than username, password, and server. Further they have configured their system to require specific settings that are not exposed by all makes and models. For example, you must set the SIP User-Agent as Vox 3.0.

## DNS

Vodafone answers DNS queries with SRV records. However your device MUST USE VODAFONE DNS SERVERS TO RESOLVE THE RELEVANT DOMAINS.

+ Primary DNS: 90.255.255.91
+ Secondary DNS: 90.255.255.255

## Username & Password

You will need to contact Vodafone via their live chat to request your VOIP username, password & server. You may need to make it clear that you are requesting your VOIP credentials and not your broadband credentials. You should guard these credentials. You are unable to change them and, if obtained, can be used to place phone calls that will be billed to your account. 

## The SIP Proxy Server

When you request your details from Vodafone they should give you the address of your SIP (proxy) server. The first agent I spoke to did not do this, and I spent some time trying other servers published on line. This did not work. So it seems that you MUST use the proxy server address provided to you by Vodafone.

## Localisation
Grandstream devices often come configured for the american market and may require localisation for use the in the UK. The details are as follows, copied from the great post [here](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2740553/highlight/true#M3287)

As an example, on the Grandstream HT812 configure with the following:

Navigate to the BASIC SETTINGS page:
+ Time Zone: GMT (London, Great Britain)
+ Self-Defined Time Zone: GMT0BST,M3.5.0/1,M10.5.0
Navigate to the ADVANCED SETTINGS page:
+ System Ring Cadence: c=400/200-400/2000;
+ Dial Tone: f1=350@-19,f2=440@-22,c=0/0;
+ Ringback Tone: f1=400@-20,f2=450@-20,c=400/200-400/2000;
+ Busy Tone: f1=400@-20,c=375/375;
+ Reorder Tone: f1=400@-20,c=400/350-225/525-0/0;
+ Confirmation Tone: f1=1400@-10,c=0/0;
+ Call Waiting Tone: f1=400@-20,c=100/2000;
+ Prompt Tone: f1=350@-19,f2=440@-22,c=0/0;
+ Conference Party Hangup Tone: f1=400@-20,c=0/0;
+ Special Proceed Indication Tone: f1=350@-19, f2=440@-22, c=750/750-0/0;
+ NTP Server: uk.pool.ntp.org
+ Navigate to the PROFILE 1/2 (FXS PORT on HT813) page(s):
+ MWI Tone: Special Proceed Indication Tone
+ Dial Plan: { 10[015] | 11[129] | 999 | 11[68]xxx | 1[45]7[1-2] | 08001111 | 0845464x | 0[1235789]xxxxxxxxx | 1410[1235789]xxxxxxxxx | 14700[1235789]xxxxxxxxx | 00xxx. | x+ | \+x+ | *x+ | *xx*x+ }
+ SLIC Setting: UK
+ Caller ID Scheme: SIN 227 - BT
+ Hook Flash Timing: Minimum: 60 Maximum: 200
+ Ring Frequency: 25
+ Ring Tone 1: c=400/200-400/2000;
+ Ring Tone 2: c=400/200-400/2000;
+ Ring Tone 3: c=400/200-400/2000;
+ Ring Tone 4: c=400/200-400/2000;
+ Ring Tone 5: c=400/200-400/2000;
+ Ring Tone 6: c=400/200-400/2000;
+ Ring Tone 7: c=400/200-400/2000;
+ Ring Tone 8: c=400/200-400/2000;
+ Ring Tone 9: c=400/200-400/2000;
+ Ring Tone 10: c=400/200-400/2000;
+ Call Waiting Tone 1: f1=400@-20,c=100/2000;
+ Call Waiting Tone 2: f1=400@-20,c=100/2000;
+ Call Waiting Tone 3: f1=400@-20,c=100/2000;
+ Call Waiting Tone 4: f1=400@-20,c=100/2000;
+ Call Waiting Tone 5: f1=400@-20,c=100/2000;
+ Call Waiting Tone 6: f1=400@-20,c=100/2000;
+ Call Waiting Tone 7: f1=400@-20,c=100/2000;
+ Call Waiting Tone 8: f1=400@-20,c=100/2000;
+ Call Waiting Tone 9: f1=400@-20,c=100/2000;
+ Call Waiting Tone 10: f1=400@-20,c=100/2000;

Remember to click on the "Update" and "Apply" buttons located at the bottom of every page to save and activate the changes.

#### Purchased a Grandstream HT813 telephone adapter?
The HT813 is an analog telephone adapter that features 1 analog telephone FXS port and 1 PSTN line FXO port in order to offer backup lifeline support using a PSTN line. Additional UK regional settings are required for this model and I have included them below.
Navigate to the FXO PORT page:

+ Caller ID Scheme: SIN 227 - BT
+ FSK Caller ID Seizure Bits: 96
+ FSK Caller ID Mark Bits: 55
+ PSTN Disconnect Tone: f1=400@-30,f2=400@-30,c=0/0;
+ Country-based: UK
+ Impedance-based: COMPLEX3 -- 370 ohms + (620 ohms || 310nF)

## Connection settings
General connection settings from [here](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2741048/highlight/true#M3334)
 
+ Primary SIP Server: resvoip.vodafone.co.uk
+ Outbound Proxy: As provided by vodafone
+ SIP UserID and Authenticate ID: As provided by vodafone
+ Authenticate Password: As provided by vodafone
+ DNS mode has to be on SRV
+ Use NAT IP I set to be my static IP that Vodafone gave me
+ SIP User-Agent set as Vox 3.0

SIP Port should be 5065 and the RTP Port should be 10000

Sometimes, once you have the correct settings you have to reboot the ATA to get them working. As noted [here](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2734431/highlight/true#M2808).

## Asterisk

User clayface very helpfully kick started the effort to make all this work by posting a working config for Asterisk [here](https://github.com/clayface/VF-UK-Asterisk-config)

