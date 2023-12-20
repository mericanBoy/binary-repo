Stack Overflow
Products
Search…
Join Stack Overflow to find the best answer to your technical question, help others answer theirs.
 
Home
Questions
Tags
Users
Companies
COLLECTIVES
Explore Collectives
LABS
Discussions
TEAMS
Stack Overflow for Teams – Start collaborating and sharing organizational knowledge. 
Virtual network interface in Mac OS X
Asked 15 years, 3 months ago
Modified 1 year, 5 months ago
Viewed 139k times
57
 
I know that you can make a virtual network interface in Windows (see here), and in Linux it is also pretty easy with ip-aliases, but does something similar exist for Mac OS X? I've been looking for loopback adapters, virtual interfaces and couldn't find a good solution.
 
You can create a new interface in the networking panel, based on an existing interface, but it will not act as a real fully functional interface (if the original interface is inactive, then the derived one is also inactive).
 
This scenario is needed when working in a completely disconnected situation. Even then, it makes sense to have networking capabilities when running servers in a VMWare installation. Those virtual machines can be reached by their IP address, but not by their DNS name, even if I run a DNS server in one of those virtual machines. By configuring an interface to use the virtual DNS server, I thought I could test some DNS scenario's. Unfortunately, no interface is resolving DNS names if none of them are inactive...
 
macosnetworkingdnsip
Share
Follow
edited Jun 27, 2022 at 6:14
JM Gelilio's user avatar
JM Gelilio
3,56211 gold badge1212 silver badges2323 bronze badges
asked Sep 17, 2008 at 20:41
Hans Doggen's user avatar
Hans Doggen
1,80622 gold badges1717 silver badges1313 bronze badges
on your same topic, more or less compileyouidontevenknowyou.blogspot.com/2009/03/… – 
Dan Rosenstark
 Mar 27, 2009 at 1:56
Add a comment
 
Report this ad
12 Answers
Sorted by:
 
Highest score (default)
66
 
The loopback adapter is always up.
 
ifconfig lo0 alias 172.16.123.1 will add an alias IP 172.16.123.1 to the loopback adapter
 
ifconfig lo0 -alias 172.16.123.1 will remove it
 
Share
Follow
answered Mar 9, 2009 at 1:03
Dave Whitla
1
This is correct for creating an alias, but to my knowledge you cannot define a DNS Server linked to the loopback adapter, so it will not work in my scenario as described above. – 
Hans Doggen
 Nov 22, 2009 at 11:11
for me this is the best solution. so I can ping from host to vm and vice versa. – 
Richard Octovianus
 Feb 8, 2020 at 12:14
Perhaps my comment is long overdue, but note that an alias can be created on any interface. – 
Greg A. Woods
 Jan 30 at 20:02
Add a comment
30
 
Replying in particular to:
 
You can create a new interface in the networking panel, based on an existing interface, but it will not act as a real fully functional interface (if the original interface is inactive, then the derived one is also inactive).
 
This can be achieved using a Tun/Tap device as suggested by psv141, and manipulating the /Library/Preferences/SystemConfiguration/preferences.plist file to add a NetworkService based on either a tun or tap interface. Mac OS X will not allow the creation of a NetworkService based on a virtual network interface, but one can directly manipulate the preferences.plist file to add the NetworkService by hand. Basically you would open the preferences.plist file in Xcode (or edit the XML directly, but Xcode is likely to be more fool-proof), and copy the configuration from an existing Ethernet interface. The place to create the new NetworkService is under "NetworkServices", and if your Mac has an Ethernet device the NetworkService profile will also be under this property entry. The Ethernet entry can be copied pretty much verbatim, the only fields you would actually be changing are:
 
UUID
UserDefinedName
IPv4 configuration and set the interface to your tun or tap device (i.e. tun0 or tap0).
DNS server if needed.
Then you would also manipulate the particular Location you want this NetworkService for (remember Mac OS X can configure all network interfaces dependent on your "Location"). The default location UUID can be obtained in the root of the PropertyList as the key "CurrentSet". After figuring out which location (or set) you want, expand the Set property, and add entries under Global/IPv4/ServiceOrder with the UUID of the new NetworkService. Also under the Set property you need to expand the Service property and add the UUID here as a dictionary with one String entry with key __LINK__ and value as the UUID (use the other interfaces as an example).
 
After you have modified your preferences.plist file, just reboot, and the NetworkService will be available under SystemPreferences->Network. Note that we have mimicked an Ethernet device so Mac OS X layer of networking will note that "a cable is unplugged" and will not let you activate the interface through the GUI. However, since the underlying device is a tun/tap device and it has an IP address, the interface will become active and the proper routing will be added at the BSD level.
 
As a reference this is used to do special routing magic.
 
In case you got this far and are having trouble, you have to create the tun/tap device by opening one of the devices under /dev/. You can use any program to do this, but I'm a fan of good-old-fashioned C myself:
 
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main()
{
   int fd = open("/dev/tun0", O_RDONLY);
   if (fd < 0)
   {
      printf("Failed to open tun/tap device. Are you root? Are the drivers installed?\n");
      return -1;
   }
   while (1)
   {
      sleep(100000);
   }
   return 0;
}
Share
Follow
edited Jun 6, 2013 at 9:30
Dave's user avatar
Dave
77477 silver badges1717 bronze badges
answered Jun 16, 2011 at 16:27
bmasterswizzle's user avatar
bmasterswizzle
30133 silver badges22 bronze badges
Will this solution allow you to turn on "Internet Sharing" to bridge the tap interface to the ethernet or airport interface? – 
Sukima
 Jan 11, 2012 at 20:41
2
I've tested it and yes, it does. I was able to share my OpenVPN tun0 device (from my Ethernet connection) over my WiFi using this method. – 
thenickdude
 Apr 2, 2013 at 9:36
Note: I tried this to a vmnet (VMware virtual network adapter) and I can't get it active with an IP. (For uses with tun/tap devices as discussed this solution really works) – 
TCB13
 Sep 2, 2013 at 1:33 
Is there a way around the "a cable is unplugged" issue? I followed the instructions and they work great, however with the "cable unplugged" OS X believes that the host is offline and trying navigate to any site in my browser tells me there is no internet connection. – 
Dan Ramos
 Jun 18, 2015 at 15:09
Hi, thanks for your reply. I'm testing your way in MacOS 12 Monterey, but open("/dev/tun0") will always return false. I think the tuntap (used brew install --cask tuntap to install) gets wrong somewhere. Is there a way to use tuntap in MacOS 12? – 
Lanistor
 Apr 5, 2022 at 9:53
Add a comment
14
 
In regards to @bmasterswizzle's BRILLIANT answer - more specifically - to @DanRamos' question about how to force the new interface's link-state to "up".. I use this script, of whose origin I cannot recall, but which works fabulously (in coordination with @bmasterswizzles "Mona Lisa" of answers)...
 
#!/bin/zsh
 
[[ "$UID" -ne "0" ]] && echo "You must be root. Goodbye..." && exit 1
echo "starting"
exec 4<>/dev/tap0
ifconfig tap0 10.10.10.1 10.10.10.255
ifconfig tap0 up
ping -c1 10.10.10.1
echo "ending"
export PS1="tap interface>"
dd of=/dev/null <&4 & # continuously reads from buffer and dumps to null
I am NOT quite sure I understand the alteration to the prompt at the end, or...
 
dd of=/dev/null <&4 & # continuously reads from buffer and dumps to null
 
but WHATEVER. it works. link light🚦: green✅. loves it💚.
 
enter image description here
 
Share
Follow
edited May 23, 2017 at 12:10
Community's user avatar
CommunityBot
111 silver badge
answered Jul 7, 2015 at 4:15
Alex Gray's user avatar
Alex Gray
16.1k99 gold badges9797 silver badges118118 bronze badges
How can you destroy the tap adapter (via script)? – 
Juuso Ohtonen
 Sep 23, 2016 at 11:02
If you want to make a Python implementation, you might want to use stackoverflow.com/a/15139727/1097104 as a basis. – 
Juuso Ohtonen
 Sep 26, 2016 at 6:51
6
It gives me the following error: line 5: /dev/tap0: Operation not permitted – 
Jay
 Jan 3, 2017 at 11:18 
Still working in MacOS 12 Monterey? I tested and failed. – 
Lanistor
 Apr 5, 2022 at 10:06
You need Tunnelblick, and the following informations to use the TAP device driver from Tunnelblick: groups.google.com/g/tunnelblick-discuss/c/v5wnQCRZ8HY/m/… After installing Tunnelblick, you can load a tap kext with the following command in Terminal (or a script): ``` /Applications/Tunnelblick.app/Contents/Resources/openvpnstart loadKexts 2 ``` – 
Tobias Hochgürtel
 Jul 25, 2022 at 15:38 
Add a comment
10
 
A few others seemed to hint at this, but the following demonstrates using ifconfig to create a vlan and test DNS on the virtual interface (using minidns) on OS X 10.9.5:
 
$ sw_vers -productVersion
10.9.5
$ sudo ifconfig vlan169 create && echo vlan169 created
vlan169 created
$ sudo ifconfig vlan169 inet 169.254.169.254 netmask 255.255.255.255 && echo vlan169 configured
vlan169 configured
$ sudo ./minidns.py 169.254.169.254 &
[1] 35125
$ miniDNS :: * 60 IN A 169.254.169.254
 
 
$ dig @169.254.169.254 +short test.host
Request: test.host. -> 169.254.169.254
Request: test.host. -> 169.254.169.254
169.254.169.254
$ sudo kill 35125
$ 
[1]+  Exit 143                sudo ./minidns.py 169.254.169.254
$ sudo ifconfig vlan169 destroy && echo vlan169 destroyed
vlan169 destroyed
Share
Follow
answered Apr 24, 2015 at 22:49
web-online's user avatar
web-online
12111 silver badge33 bronze badges
Add a comment
6
 
It's possible to use TUN/TAP device. http://tuntaposx.sourceforge.net/
 
Share
Follow
answered Nov 19, 2009 at 15:29
psv141's user avatar
psv141
6111 silver badge11 bronze badge
User yar made a similar suggestion through a blog post. I've looked at it, but I didn't test it out, because of lack of time and I don't have the need anymore. Nevertheless thanks. – 
Hans Doggen
 Nov 19, 2009 at 22:39
Add a comment
2
 
if you are on a dev environment and want access some service already running on localhost/host machine. in docker for mac you have another option.use docker.for.mac.localhost instead of localhost in docker container. docker.for.mac.host.internal should be used instead of docker.for.mac.localhost from Docker Community Edition 17.12.0-ce-mac46 2018-01-09. this allows you to connect to service running on your on mac from within a docker container.please refer below links
 
understanding the docker.for.mac.localhost behavior
 
release notes
 
Share
Follow
answered Jun 8, 2018 at 7:21
Arvin_Sebastian's user avatar
Arvin_Sebastian
1,05611 gold badge1212 silver badges1818 bronze badges
1
Thank you so much for this! – 
DragonBobZ
 Mar 1, 2021 at 22:15
Add a comment
1
 
What do you mean by
 
"but it will not act as a real fully functional interface (if the original interface is inactive, then the derived one is also inactive"
 
?
 
I can make a new interface, base it on an already existing one, then disable the existing one and the new one still works. Making a second interface does however not create a real interface (when you check with ifconfig), it will just assign a second IP to the already existing one (however, this one can be DHCP while the first one is hard coded for example).
 
So did I understand you right, that you want to create an interface, not bound to any real interface? How would this interface then be used? E.g. if you disconnect all WLAN and pull all network cables, where would this interface send traffic to, if you send traffic to it? Maybe your question is a bit unclear, it might help a lot if rephrase it, so it's clear what you are actually trying to do with this "virtual interface" once you have it.
 
As you mentioned "alias IP" in your question, this would mean an alias interface. But an alias interface is always bound to a real interface. The difference is in Linux such an interface really IS an interface (e.g. an alias interface for eth0 could be eth1), while on Mac, no real interface is created, instead a virtual interface is created, that can configured and used independently, but it is still the same interface physically and thus no new named interface is generated (you just have two interfaces, that are both in fact en0, but both can be enabled/disabled and configured independently).
 
Share
Follow
answered Sep 17, 2008 at 20:58
Mecki's user avatar
Mecki
127k3333 gold badges251251 silver badges257257 bronze badges
Add a comment
1
 
Go to Network Preferences.
 
At the bottom of the list of network adapters, click the + icons
 
Select the existing interface that you want to arp (say Ethernet 1), and give the Service Name that you want for the new port (say Ethernet 1.1) then press create.
 
Now you have the new virtual interface in the gui and can manage IP addresses etc it in the normal way.
 
ifconfig -a will confirm that you have multiple IPs on the interface, and these will still be there when you reboot.
 
Its a Mac. Don't fight it, do it the easy way.
 
Share
Follow
edited Jan 5, 2011 at 15:05
answered Jan 5, 2011 at 14:25
Henry 3 Dogg's user avatar
Henry 3 Dogg
2922 bronze badges
4
As noted in the original question, this doesn't achieve the desired effect of a virtual interface isolated from a physical interface. The command you described simply aliases the same physical interface with a second IP configuration. – 
bleater
 Feb 19, 2012 at 23:58
Add a comment
0
 
i have resorted to running PFSense, a BSD based router/firewall to achieve this goal….
 
why? because OS X Server gets so FREAKY without a Static IP…
 
so after wrestling with it for DAYS to make NAT and DHCP and firewall and …
 
I'm trying this is parallels…
 
will let ya know how it goes...
 
Share
Follow
answered Jun 8, 2010 at 17:49
community wiki
Alex Gray
Add a comment
0
 
Take a look at this tutorial, it's for FreeBSD but also applies to OS X. http://people.freebsd.org/~arved/vlan/vlan_en.html
 
Share
Follow
answered Sep 24, 2010 at 1:32
Ariel Monaco's user avatar
Ariel Monaco
3,73911 gold badge2424 silver badges2121 bronze badges
Add a comment
-1
 
ifconfig interfacename create will create a virtual interface,
 
Share
Follow
answered Sep 17, 2008 at 21:01
Brian Mitchell's user avatar
Brian Mitchell
2,2781414 silver badges1212 bronze badges
Could you provide an example for that? If I try that, it does not work and has never worked as far as I know. – 
Mecki
 Sep 17, 2008 at 21:21
ifconfig vlan0 create as documented in the ifconfig man page "create Create the specified network pseudo-device. If the interface is given without a unit number, try to create a new device with an arbitrary unit number." – 
Brian Mitchell
 Sep 18, 2008 at 14:48
1
It will not work as the vlan0 above cannot be made active (status) and so it will not receive any packets... – 
Hans Doggen
 Sep 18, 2008 at 18:48
Well your question related a virtual interface, without any indication about what you wanted to do with it, which is specifically what I answered. This feature is for vlans, you can do something like ifconfig vlan0 vlan sometag vlandev en0 ipconfig set vlan0 DHCP If a vlan is your goal. – 
Brian Mitchell
 Sep 19, 2008 at 18:16
@HansDoggen do you know why the vlan interface cannot be made active? – 
Dmitry Minkovsky
 Nov 13, 2014 at 16:44
Add a comment
-1
 
Here's a good guide: https://web.archive.org/web/20160301104014/http://gerrydevstory.com/2012/08/20/how-to-create-virtual-network-interface-on-mac-os-x/
 
Basically you select a network adapter in the Networks pane of system preferences, then click the gear to "Duplicate Service". After the service is duplicated, you manually assign an IP in one of the private address ranges. Then ping it to make sure ;)
 
Share
Follow
edited Oct 10, 2019 at 17:22
Ry-'s user avatar
Ry-♦
220k5555 gold badges474474 silver badges482482 bronze badges
answered Oct 22, 2013 at 14:46
David Mann's user avatar
David Mann
1,90433 gold badges1616 silver badges1919 bronze badges
The original post is asking help for create one NEW virtual networking interface, not asking for adding new IP address for an existing networking interface. – 
mxi1
 Jul 11, 2016 at 9:24
I agree that the question clearly specifies a virtual interface, and my answer and the link I provided don't delineate between a virtual interface and a physical interface. However the question doesn't really ask for any of this in particular. It asks for "something similar" to making a Virtual Network Device in Windows. – 
David Mann
 Jul 13, 2016 at 12:41
Add a comment
Your Answer
Sign up or log in
Post as a guest
Name
Email
Required, but never shown
 
By clicking “Post Your Answer”, you agree to our terms of service and acknowledge that you have read and understand our privacy policy and code of conduct.
 
Not the answer you're looking for? Browse other questions tagged macosnetworkingdnsip or ask your own question.
Featured on Meta
Seeking feedback on tag colors update
Update to our Advertising Guidelines
Temporary policy: Generative AI (e.g., ChatGPT) is banned
Rule proposal: Duplicate closure to roll-up questions are no longer allowed
 
Report this ad
19 people chatting
Linked
7
Interfacing with TUN\TAP for MAC OSX (Lion) using Python
1
Emulating a virtual host for UDP communication
-1
How to force MacOS to send network packets to local proxy even when Wi-Fi is not connected
0
Ionic Android Accessing Local Sites Set By /etc/hosts
Related
2
FOSS solution for a local machine: DNS
1
Virtual Network Interface in Python
0
Host LAN interface from iPhone/ Mac App
0
OS X Server DNS management
5
How to bind the VM docker-machine creates to OSX IP address?
10
specify ip address for docker for mac
1
Convert Network Interface Name
0
how to network to and from local VM (fusion) with fixed IP on mac os host
5
How to create virtual interface in macOS?
0
Mac equivalents for Linux ip command
Hot Network Questions
May an airplane with folding wings legally taxi on the street?
How do I create a SearchKit report that segments data based on a multi-valued custom field?
Is there any evidence that the modern word for "bear" is an euphemism which replaced the original taboo word?
trimming a string using C++20 ranges
Escape Room Puzzle: numbers in Christmas trees
Why are all of my "this" pointers the same value?
Can I use two ethernet card to increase transfer speed between two linux OS in lan?
Multiplying independent conditional probabilities
Am I vaccinated?
Can a treaty between sovereign states be anulled in the future on the basis that one of the signatories was not a democracy or otherwise "free"?
Mathematical benefit to use CPU/memory that increases by powers of 2 as 8-bit, 16-bit, 32-bit, 64-bit, etc?
Is path analysis equivalent to a series of regressions?
How to estimate LED lifetime?
How can I add an animation or video to a peer review report without using "unprofessional" sites like imgur or youtube?
How is rainfall measured when the news speaks of "two inches of rain"
Prismatic wall is described as vertical. A cylinder has vertical walls. So can a Prismatic wall be shaped as a cylinder?
Real-world applications of eigendecomposition?
How can I select elements of a list so that three first numbers of this elements present two or three times?
Book I read years ago about 2-dimensional space encroaching on our own and slowly encompassing everything
Can't identify this IC, package SOT23-5 top mark bPDFZ
70s movie about a secret series of robots that helped people
How to deal with multiple supervisors and different expectations?
Do we say "The water of the soup is yummy"?
Has there ever been a successful shift from a two-party system to a multi-party system in modern history?
 Question feed
 
STACK OVERFLOW
Questions
Help
PRODUCTS
Teams
Advertising
Collectives
Talent
COMPANY
About
Press
Work Here
Legal
Privacy Policy
Terms of Service
Contact Us
Cookie Settings
Cookie Policy
STACK EXCHANGE NETWORK
Technology
Culture & recreation
Life & arts
Science
Professional
Business
API
Data
Blog
Facebook
Twitter
LinkedIn
Instagram
Site design / logo © 2023 Stack Exchange Inc; user contributions licensed under CC BY-SA. rev 2023.12.15.2752
