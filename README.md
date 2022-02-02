# ibm-tape-drive-automatic-standalone
 # How to turn an IBM Drive into automatic standalone mode


Hello all 

I did get an IBM LTO 6  lately and was having trouble setting it in automatic drive online Mode. After a lot of reading of the barely available documentation and lending an oscilloscope from a friend, I was able to put things together and get the drive to work without an library. 
Hopefully I am able to save someone on this planet some time and struggle by sharing my documentation. 

In the first place I did get an RS-422-USB cable with an screw terminal, but due to the undocumented pinout of the JST-SH 10 Pin connector I wasn’t successful in establishing an communication (maybe just bad luck). 

The second step was to search out the circuit board and check the components. There I found an chip from TI, which is similar to this one THVD1451. TI has an great documentation for its components!
https://www.ti.com/product/THVD1451

![image](https://user-images.githubusercontent.com/98891123/152162290-752d8bbf-0fc4-456e-8809-531cda45393c.png)
![image](https://user-images.githubusercontent.com/98891123/152162314-c49de7ca-9f51-4cc7-95e9-9b17d5773d2e.png)
![image](https://user-images.githubusercontent.com/98891123/152162334-05ad65c0-9cbd-4c85-b72e-97fa1ea07222.png)

Perspective from the Drive Side ( Issue #1) 

![image](https://user-images.githubusercontent.com/98891123/152176847-c555ccfa-2e87-47ef-961c-52edfab7a02a.png)
![image](https://user-images.githubusercontent.com/98891123/152177129-d5e34251-cee9-4cab-a584-445cd794601d.png)


Following the Electrical Pathways  I was assuming the following cabling:
- Receive:
  - A → Yellow OR Orange
  -	B → Orange OR Red
- Send:
  -	Z → Red OR Brown 
  - Y → Brown OR Black

With the help of the  oscilloscope it was possible to evaluate, although it was the fist time for me utilizing one. 

![image](https://user-images.githubusercontent.com/98891123/152162736-84c0ad4c-1e22-429d-8ea6-106cb97cc356.png)

![image](https://user-images.githubusercontent.com/98891123/152162847-dea67800-ae85-40d9-96e9-71dc8a4ec280.png)

![image](https://user-images.githubusercontent.com/98891123/152162875-e5c36c60-7c8f-489d-ad4e-b6ffae3ece9c.png)


Now I knew i was receiving signals on the YELLOW and the ORANGE Cables. The RS422 standard specifies differential signaling. In the picture above you can see the signals switching polarity which is causing an increased amplitude at the receiving side. I am not an electrical engineer and therefor can’t go any deeper in detail. 

But what was it I was receiving? There would come the IBM Library/Drive Interface Specification in handy.  http://t10.org/ftp/t10/document.02/02-022r0.pdf

There are the Control Characters which did appear 0x02 and 0x03 for Start an End of Text.  

![image](https://user-images.githubusercontent.com/98891123/152162997-69e45124-c0a5-49d5-8c48-2b1768a3b464.png)


The middle Part seemed to be the Message for a Config_Request. To see them it is crucial to set the drive in non-polled Mode with the dip switches on the back. I have set them all to OFF. 

![image](https://user-images.githubusercontent.com/98891123/152163013-b7b6498f-39a3-4c2a-abce-3820d4816906.png)
![image](https://user-images.githubusercontent.com/98891123/152163053-b3efa985-fe00-4cc0-8fab-36bf3539d206.png)
![image](https://user-images.githubusercontent.com/98891123/152163067-5f7ea18a-5ebb-48a0-88ee-a48df08b76a9.png)


Packets does have the following format:
![image](https://user-images.githubusercontent.com/98891123/152163104-7ca47035-da86-4f52-a035-0a7329f9e351.png)


So this would make sense for an drive that was took out of an library. At page 30 is a description of the Interface Scenarios. The drive is operation in an **non-polled Mode** due to the **dip switch**. This means the drive is initiating the handshaking process and asking for a config. 

![image](https://user-images.githubusercontent.com/98891123/152163134-550e998d-d1a5-4380-a1e2-0156872dc25b.png)

To fulfill the handshake we need to “craft” an package. From Page 16 on is the description how an **Set_Config** message should look like. 

![image](https://user-images.githubusercontent.com/98891123/152163159-192aaf2d-7fc1-4f8c-a00b-b2d065ca246c.png)

And a useful information was in the interface scenario on Page 30. 

![image](https://user-images.githubusercontent.com/98891123/152163178-3bb03fc6-546f-48e1-960e-1ca4cb2b33c2.png)

If we would just craft the package with the first 54 bytes, we would be firmware independent. 

Still we do not know what the sending cables are. The good news is there are packets we can try to send to the library which are outside of the package protocol, meaning we don’t need start-text, length, checksum, end-text, acknowledgments, or byte stuffing. 
If we send just hex 00 to the drive, we should receive something with “I” “B” “M”. 

![image](https://user-images.githubusercontent.com/98891123/152163214-0588d6c0-9726-45c0-b103-f97ed7d57281.png)

If you remember the lines I assumed as sending line have been: 
- Z → Red OR Brown 
- Y → Brown or Black

It turn out to be: 
- Z → Red
- Y → Brown

- Ground is Black. 

After connecting Red to TxD+, Brown to TxD- and Black to GND on our  RS-422-USB Adapter, we are ready to send the Set_Config package to the drive. The setting we try to accomplish is to set **Bit 3 in Byte 53** to 0. 
This would allow the drive to raise the SCSI or Fiber Channel Ports without the need of receiving an Set_Config Package every time it powers up. 

![image](https://user-images.githubusercontent.com/98891123/152163290-d9100219-32f8-40ec-8fc1-91d1485469a5.png)

If we where successful, the drive would send and Acknowledge **0x06** an reboot like stated in the Scenario for Startup Process.

![image](https://user-images.githubusercontent.com/98891123/152163306-3f24bb5a-9139-402d-ac0a-e50cad04572a.png)

How to build the Package is described on the Pages 11 and 12 in the LDI Specification. 

For transmitting the package I did use the following script:
```
#!/bin/bash 
stty -F /dev/ttyUSB0 speed 38400 -cstopb -parenb -echo

while read -r line
do 
	echo -en “$line” > /dev/ttyUSB0
done < “$1”
```
Save this as echo_hex and change the mode to execute 
```
chmod +x ./echo_hex
```
Then we need an file with the hex code. Just name it code for example. 
```
\x02\x00\x36\xAC\x01\x00\x00\x00\x01\xFF\xF2\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xE6\x03
```

Then we can issue the command 
```
./echo_hex code
```
and see the Packages 0x06 0x03, for ACK and ETX on the receiving side. 

![image](https://user-images.githubusercontent.com/98891123/152163518-da7152b4-953a-43f7-9697-e13dda06578d.png)

Now the drive is rebooting and we can see the Link online. This qaucli is a command from qlogic (MARVELL) which in my case produced the HP Branded FC Card. 

![image](https://user-images.githubusercontent.com/98891123/152163541-30075b85-7de1-441d-9a2a-ce39d1bea7b2.png)

Also lsscsi is outputting the the Tape now. 

![image](https://user-images.githubusercontent.com/98891123/152163611-95b1c297-5de1-4b06-9e24-9f1be23f8520.png)


SUMMARY

- Set all DIP Switches on the drive to off. 
- Connect the  RS-422-USB Adapter as follows:
  - TxD+ → Red
  - TxD- → Brown
  - RxD+ → Yellow
  - RxD- → Orange
  - GND → Black
- Send the Package to the drive: ```
\x02\x00\x36\xAC\x01\x00\x00\x00\x01\xFF\xF2\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xE6\x03 ```
- Drive restarts and goes online to SCSI or Fibre Channel automatically.
