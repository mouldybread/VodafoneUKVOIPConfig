# Vodafone UK VOIP Residential FTTP Configuration notes
A reference for using your own VOIP hardware (Grandstream WP810) with Vodafone UK Residential FTTP + OPNSense

As of writing Vodafone ships a router with a built in component that will allow you to connect your existing PTSN phone to the Vodafone VOIP network. This should mean you can connect any of your existing VOIP equipment to their network. However because money, Vodafone would rather residential customers be unable to use their own equipment. [Here it is in their own words.](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2744786/highlight/true#M3568):
  ```
    “As this is a consumer Home Broadband service and not a business service where this is explicitly provided, Vodafone are unable to offer this as a service to the customer”
  ```

## Sources
Most of the information contained here is from [this thread](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2709457) on the vodafone forums.

This sprawling thread is essentially the authority on the subject. I'm going to condense it down for my own reference and deploy a WP810. In the hopes that this may help someone else, here are my notes. All credit goes to the members there sharing their hard work. 

## OPNSense VOIP / SIP Configuration.

This is actually the easy part but there are a huge number of wrong/outdated/incomplete guides/notes/posts. Helpfully I had a confirmed working Polycom phone on hand to test my network settings. But in short, no, you don't have to use a DMZ, port forward, or open up anything. 

First I found [this post](https://www.3cx.com/docs/pfsense-firewall/) with some nice short instructions for Pfsense. Step 2 is completed on OPNsense by going to Firewall -> NAT -> Outbound and adding a selecting "Hybrid outbound NAT rule generation".

Step 1 is completed by making an alias Alias under Firewall -> Aliases and added my Polycom to the list. Then I went back to Firewall -> NAT -> Outbound and added a manual rule, Interface: WAN Source:VOIPDEVICES(Alias) Source Port: tcp/udp/* Destination:* Destination Port: tcp/udp/* NAT address: Interface Address NAT Port * Static Port: YES

Static port being the important part. Contrary to the title of the section in the 3CX documentation this is not port forwarding. This should ensure that the source port of SIP packets are not rewritten by the firewall.

Finally I added rules to permit all outbound traffic from the VOIP phones alias. Not ideal but easy enough to tighten later.

Then I found [this](https://www.reddit.com/r/opnsense/comments/16n2fr3/voipdectbasestation_behind_opnsense_firewall/?rdt=52318) post which also seemed to say it should be this simple.

That's all it took for me to make my Polycom phone work, and later on the WP810. Just add the new IP to the VOIP alias list.

#### Some detail on these settings

Usually NAT will rewrite the source port of outbound packets as they traverse the router and keep track of the changes in the state table. When it receives the resulting inbound packets it will rewrite the destination port to match the original and send the packet over the LAN, allowing two outbound connections with the same source port to traverse the NAT at the same time.

This causes an issue for RTP(see [here](https://www.sonicwall.com/support/knowledge-base/troubleshooting-a-scenario-where-source-remap-is-causing-the-voip-issues/170504967157192/) for an explanation).

To work around this we can use the amazing versatility of OPNSense to disable source port remapping for our VOIP phone. We do this with the above OUTBOUND nat rule that specifies all packets from the phone as STATIC=YES. Hence we also specify the phones alias as the matching source, because we really need this feature for every other device on the network.

However there are further considerations. RTP by default uses a random UDP port from 10000-32767. Ripshods configuration restricts the RTP port range to 20 from a base port of 10000. This though is suboptimal, picking a random unused source port is especially important as we have disabled outbound remapping. A bigger range is better. The phone is unaware of which ports are in use. If something else is (or has) been using UDP/10000(for example), the call will fail(this is the problem that remapping aims to fix after all). So a bigger range is better and relying on a single port or small pool may cause intermittent issues.

The elegance of this approach though is that we do not have to expose anything to the internet. No DMZ, no port mapping - which would be the other way around this issue.

## WP810 Configuration

Ripshod very kindly posted screenshots of his configuration [here](https://forum.vodafone.co.uk/t5/Landline/Landline-phone-with-own-router-on-FTTP/m-p/2734408/highlight/true#M2802)
for download from [here](https://drive.google.com/drive/folders/1aquAVMgOeln9x0-O_nRCGQL-dzxZrFhi?usp=sharing)

I followed these settings almost exactly as is for my WP810 with one exception. I set "Local RTP Port Range" to 200. The pictures show 24, however according to the pop up help tip, this is outside of the acceptable range. I did not put a space between "Vox3.0". The one or two missing options were inconsequential.

I will note that for the OPNSense configuration posted above, the correct NAT traversal is as shown: "No." I did provide a stun server address but I didn't use it. I also entered my current external IP. I do know it doesn't change very often but I suspect if it does I will have to update it manually. Selecting the STUN option for NAT traversal did not work.

Finally I applied some of the localisation settings as noted below.

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

## USERNAME/PASSWORD

You will need to contact Vodafone via their live chat to request your VOIP username, password & server. You may need to make it clear that you are requesting your VOIP credentials and not your broadband credentials. You should guard these credentials. You are unable to change them and, if obtained, can be used to place phone calls that will be billed to your account. 

## THE PROXY SERVER

When you request your details from Vodafone they should give you the address of your SIP (proxy) server. The first agent I spoke to did not do this, and I spent some time trying other servers published on line. This did not work. So it seems that you MUST use the proxy server address provided to you by Vodafone.

## LOCALISATION
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

### Purchased a Grandstream HT813 telephone adapter?
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

