




 

Bigip A:

Netmask	Self IP	Untagged VLAN	Interface
255.255.255.0	10.1.1.4		MGMT
255.255.255.0	10.1.10.100	client_vlan	1.1
255.255.255.0	10.1.20.100	server_vlan	1.2
255.255.255.0	10.1.30.100	ha_vlan	1.3


Bigip B:

Netmask	Self IP	Untagged VLAN	Interface
255.255.255.0			10.1.1.6		MGMT
255.255.255.0	10.1.10.101	client_vlan	1.1
255.255.255.0	10.1.20.101	server_vlan	1.2
255.255.255.0	10.1.30.101	ha_vlan	1.3
Create VLANs
We will create VLANs and assign self IP addresses to our VLANs.
You will need to create three untagged VLANs, one for client-side VLAN (client_vlan) and one server-side VLAN (server_vlan), and one for the HA cluster VLAN (ha_vlan).
1)	Login to Bigip-A, from the sidebar select Network >> VLANs then select Create
2)	Under General Properties:
a)	Name: client_vlan
b)	Tag: <leave blank>
3)	Under Resources in the Interfaces section:
a)	Interface: 1.1
b)	Tagging: Untagged
c)	Select the Add button. Leave all other items at the default setting

 

4)	When you have completed your VLAN configuration, hit the Finished button.
5)	Create another untagged VLAN named server_vlan on interface 1.2
6)	Create another untagged VLAN named ha_vlan on interface 1.3
7)	Login to Bigip-B and repeat on steps 1-6
8)	When you have completed your VLANs configuration, you should have something similar to the following:

 


















Assigning a Self IP address to your VLANs
1) Go to Network >> Self IPs, select Create.
2) Create a new three self IP, for the client_vlan and server_vlan and ha_vlan VLANs.
3) In Network >> Self IPs >> New Self IP, under Configuration enter:

 

Port Lockdown differences:
Allow None:
Means the Self IP would respond only to ICMP.
	Allow Defaults:
Opens the following ports on the self IP of the VLAN:
1)	TCP: ssh, domain, snmp, https
2)	TCP: 4353, 6699 (for F5 protocols, such as HA and iQuery)
3)	UDP: 520, cap, domain, f5-iquery, snmp
4)	PROTOCOL: ospf


4) For ha_vlan on the Port Lockdown choose: Allow Defaults.
5) When you have completed your self IP configuration, you should have something similar to the following:

 













Configure A Default Gateway
1)	Go to Network > Routes and then Add.
2)	Under Properties
a)	Name: default_gateway
b)	Destination: 0.0.0.0
c)	Netmask: 0.0.0.0
d)	Resource: Use Gateway…
e)	Gateway Address: 10.0.10.1
 
3)	When you have completed defining your default gateway, hit the Finished  button
4)	Verify your network configuration (need to edit)
a)	Use an SSH utility, such as puTTY, to access your BIG-IP management port at 10.1.1.4.
i.	User: root Password: default
ii.	Ping your default gateway, 10.1.10.1
iii.	Ping a web server at 10.1.20.10, 10.1.20.11, 10.1.20.12

Configure HA
Now we will set up HA cluster, on each BIG-IP prior to building the Device Trust, it is recommended to renew the BIG-IP self-signed certificate with valid information and regenerate the local Device Trust certificate:
1)	Under Device Management >> Device Trust >> Local Domain select Reset Device Trust…
2)	In the Certificate Signing Authority select Generate New Self-Signed Authority and hit Update
 

3)	On each BIG-IP configure the device object failover parameters the BIG-IP will send to other BIG-IPs that want to be a part of a sync-only or sync-failover group
a)	Under Device Management >> Devices, select the local BIG-IP. It will have the (Self) suffix.
b)	Under Device Connectivity on the top bar select:
c)	ConfigSync
d)	Use the Self IP address of the HA VLAN for your Local Address.
 
e)	Failover Network
4)	In the Failover Unicast Configuration section select the Add button
5)	Use the Self IP address the HA VLAN for your Address
6)	Leave the Port at the default setting of 1026
 



7)	Mirroring
8)	Primary Local Mirror Address: use the Self IP address of the HA VLAN for your
9)	Secondary Local Mirror Address: None
 
10)	On Bigip-A build the Device Trust.
a)	Under Device Management >> Device Trust >> Device Trust Members and select Add to add other BIG-IP(s) you will trust.
b)	Device IP Address: <We will use HA IP of Bigip B which is 10.1.1.6 >
c)	Enter the Administrator Username (admin) and Password (iloveF5!) of the BIG-IP you are trusting
d)	Select Retrieve Device Information
11)	The certificate information and name from the other BIG-IP should appear
a)	Select Device Certificate Matches to proceed
b)	Select Add Device
12)	On each BIG-IP check the other BIG-IP in the Device Trust Members list.
13)	They should be In Sync. But wait! Although they show in sync, only a Sync-Only group was created. We now need to create a Sync-Failover group to facilitate failover.
14)	On Bigip A create a new Sync-Failover device group
a)	Under Device Management >> Device Group create a new device group
b)	Name: my-device-group
c)	Group Type: Sync-Failover
d)	Add the members of the group to the Includes box
e)	Check the Network Failover setting for the group
 
15)	Check Device Groups on each BIG-IP
16)	Did you have to create the Device Group on the other BIG-IP?
17)	Is the full configuration synchronized yet? (No! Only the Device Group is sync’d)
18)	What is your sync status?
19)	It should be Awaiting Initial Sync
a)	Click on the sync status or go to Device Management >> Overview (or click on Awaiting Initial Sync) of the BIG-IP with the good/current configuration
b)	Click the device with the configuration you want to synchronize. Sync Options should appear.
c)	Synchronize to Group. It could take up to 30 seconds for synchronization to complete.
20)	What are the statuses of your BIG-IPs? Do you have an active-standby pair?
21)	Now go to Device Management >> Traffic Groups and move the two devices from Load Aware to Preferred Order and click save.
 
22)	Now let’s create floating IPs, In Network >> Self IPs >> New Self IP, under Configuration enter:

































Creating Pool
We will build a pool and virtual server to set up our website in this lab.

1)	From the sidebar, select Local Traffic >> Pools then select Create.
2)	Under Configuration:
a)	Name: pool_app
b)	Description: <optional>
c)	Health Monitor: http
3)	Under Members:
a)	Load Balancing Method: <leave at the default Round Robin>
b)	Priority Group Activation: <leave at default>
c)	New Members:
Address	Service Port
10.1.20.10	80
10.1.20.11	80
10.1.20.12	80

4)	When you have completed your pool configuration, hit the Finished button
 








Creating Virtual Server
Now let’s build our virtual server:
1)	Under Local Traffic >> Virtual Servers, then select Create
2)  Under General Properties
a)	Name: vs_app
b)	Description: <optional>
c)	Type: Standard
d)	Source/Address: <leave blank>
•	The default is 0.0.0.0/0, all source IP address are allowed
e)	Destination Address/Mask: 10.1.10.10
•	The default mask is /32
f)	Service Port: 80 or HTTP
g)	Under Configurations >> Source Address Translation:  Auto Map

SNAT:
You can choose which IP address the backend server will see



	Automap:
The backend server will see the self-IP’s F5	None:
The backend server will see the client IP address
(If the F5 is not the default gateway of the backend server– it can cause to asymmetric route)

3) Under Resources
a)	iRules: none
b)	Default Pool: From the drop down menu, select the pool (pool_app) which you created earlier
c)	Default Persistence Profile: None
d)	Fallback Persistence Profile: None
4) When you have completed your virtual server configuration, hit the Finished button
6) Sync the last configuration to the second device
5) Now let’s see if our virtual server works!
a) Open the browser to the Virtual Server you just created (10.1.10.10)
b) Refresh the browser screen several times 








