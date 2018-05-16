##########################################
#  _    _ ______ _      _                #
# | |  | |  ____| |    | |        /\     #
# | |__| | |__  | |    | |       /  \    #
# |  __  |  __| | |    | |      / /\ \   #
# | |  | | |____| |____| |____ / ____ \  #
# |_|  |_|______|______|______/_/    \_\ #
#                                        #
##########################################
Hella GmbH & Jacobs University Bremen

JUB Wireless Project 

Beaglebone Green Documentation -- Seongjin Bien

0. Index
device pw: temppwd

1. CAN Parser
2. Start-up scripts
	a. can-setup 1 and 2
	b. timesync
	c. start.sh
	d. logging
	e. systemd services 
	f. Configuration files
	
3. Modem setup 
	a. Setting up a new modem
	b. setup_modem_sw-em7305.sh 

4. Data transmission
	a. Transmission script buffer_publisher.py

5. Hardware 

6. Running the device
	a. General instructions
	b. Pipeline diagram

7. Setting up a new device from scratch 
################################

# 1. CAN Parser 

	The CAN parsing software is found in ~/cdec/scds/socketcanDecodeSignal . The script takes in the raw datastream from the Hella CAN device, path to the DBC file name of the car, and the desired parameters. The base code for this program was sourced from https://github.com/ebroecker/SocketCandecodeSignals .

Example Usage:

'''
	candump -L can0 | ./cdec/scds/socketcanDecodeSignal <path-to-dbc>/cc_initial.dbc <channels separated by space> <car name> 
'''

The above example produces the parsed data onto terminal.

# 2. Start-up scripts

# a. can-setup 1 and 2 #

i. can-setup.sh

	Starts when the device is booted; waits until can0 network interface is up, then records the time at which can0 starts publishing data.

ii. can-setup2.sh

	Starts when can0 interface is up, and the internet is not yet connected. Logs the initial set of data that cannot be published due to the lack of connection to a file called cs-log.txt 

# b. timesync #

	timesync.sh, timesync.py

	Runs when network connection is established. It calculates the totla difference in time before and after internet connection to fix the timestamp of the data published. 

# c. start.sh #
	
	Core script of the project. Initiated when the internet comes up. Logs the post-connection time value to timevar.txt for calculation by timesync, and starts sending the data to buffer_publisher.py to be published. 

# d. logging #
	
	Logging is done inside the SD card attached. Not having an SD card will mean that none of the information will be logged onto the device. The script log-check.py also checks for data files that are more than 3 days old. Make sure that the SD card has writing permissions enabled.

# e. systemd services

	The systemd services can be found in /etc/systemd/system/ .

	can-setup.service controls can-setup.sh and can-setup2.sh . 

	hella_candec.service controls start.sh .

	modem-lte.service controls the modem connection; specifically, it runs the setup_modem_sw-em7305.sh script, further elaborated in part 3. 

# f. Configuration files
	
	timevar.txt : stores the time immediately after Hella's CAN device starts publishing data, and the time immediately after internet connection has been established. Beaglebone does not have an onboard battery, so the time data is always lost, hence requiring such fix. 

	config.cfg : stores the car's name (l1), desired CAN channels (l2) and the absolute path to the dbc file (l3). The variables' line position must be kept consistent.

# 3. Modem setup

	The current modem being used is em7305, and the device has been configured to use it. 

	a. In /lib/udev/rules.d/, 99-sierra-wireless-em7305-simple.rules has been added to run the modem-lte.service automatically when the correct modem is connected. The modem em7305's product id MUST be changed to 9056 via AT commands (see vendor instructions) for the above rule to take effect, and for the GPS to work. 

	b. setup_modem_sw-em7305.sh : Enables the modem and connects to the internet, or loops indefinitely until it is up. The fields for APN (currently set to web.vodafone.de), password, ip type, etc. can be found here (mmcli bearer arguments). It also syncs the time to ntp.org's.

# 4. Transmission script buffer_publisher.py 

	The script depends on todic.py for the todic method.

	When the script is started, it checks for connection to the backend, and attempts until it is established. The parsed data from socketcanDecodeSignal is fed directly into this script, where the data is further separated into dictionaries to set up the data for transmission to the backend. 

	The messages are converted into bytes in order to minimise the size. 
	
	At the start, the script processes the cs-log.txt file to sync the time (using the timesync.py script) and stores it into processed.txt which can be found in ~/cdec/scds/auxFiles , which will be pushed onto the publishing queue. 
	
	When network connection goes down in the midst of publishing, the script no longer attempts to transmit the data. Instead, a temporary log file is created in /auxFiles and the data is pushed there. When connection comes up again, the temporary log file is published alongside the new file via multithreading. 

# 5. Hardware
	
	This device is intended to work with a number of peripherals, namely the modem em7305 housed in a USB converter, a microSD card and the Hella CAN data device. 

# 6. Running the device

	a. General instructions
	
	Start up the device with all the other devices listed above. It will soon begin publishing. 

	b. Pipeline diagram

00000000
0 BOOT 0
00000000
   ||
000000000000000000000          000000000000000000000
0 can-setup.service 0          0 modem-lte.service 0
0    can-setup.sh   0 ======== 0     modem script  0
0    can-setup2.sh  0          0     timesync.sh   0
000000000000000000000          0     gps_getter.sh 0
			       0     logger.sh     0
		               000000000000000000000
				 	||
          		     can-setup.service killed
				              ||
 	   			000000000000000000000000000
   			        0 hella_candec.service    0
	  		        0     buffer_publisher.py 0
 		      	        0     timesync.py         0
			        0   #publishing starts    0
			        000000000000000000000000000
						    

# 7. Setting up a new device from scratch

a. Requirements
	i.	Debian 9 or higher
	ii.	Required packages:
		- mmcli (interacting and connecting with the modem) 
		- minicom (for accessing modem AT interface)
		- systemctl (should be installed by default) 
		- Python libraries 
			- paho mqtt
			- avro (.io and .schema, specifically)  
		- socketcan 
	iii. 	The core files listed above

b. Steps

i. 	Install everything specified above.
ii. 	Put all the .sh files into the home folder (/home/debian). 
iii. 	Put the .rules file to /lib/udev/rules.d/
iv. 	Put all the .service files to /etc/systemd/system/
v.	Enable only the can-setup.service file to start at boot using systemctl. 
vi. 	Put the cdec directory into the home folder (/home/debian). 
vii.    Make sure the absolute file paths are correct for each of the files you copy into the device (.service, .sh, .py files, especially).
viii. 	Configure the /etc/network/interfaces file as below by adding these lines at the end:
		#setting up CAN interface
		allow-hotplug can0
		iface can0 can static
		    bitrate 500000


		iface wwan0 inet dhcp
		    dns-nameserver 8.8.8.8
	This ensures that the device will communicate with the CAN device via the correct interface, and that wwan0 (modem) will have the right dns address. 
	

# 8. Possible issues

a. No disk space: 

	Check the log files /var/log/syslog and /var/log/daemon.log and their sizes.


# 9. Useful manpages 
"
https://www.freedesktop.org/software/ModemManager/man/1.0.0/mmcli.8.html
https://elinux.org/BeagleBone_Black_Extracting_eMMC_contents
https://help.ubuntu.com/community/Minicom
"
