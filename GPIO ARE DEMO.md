# General Purpose Input Output (GPIO) in the Application Runtime Environment (ARE) Demo

## Requirements
- IMX8ULP-EVK
- Azure Sphere NRE Image for IMX8ULP
- Mariner ARE Image for IMX8ULP

## Software Setup

1. Flash the NRE image to the IMX8ULP.[Azure Sphere documentation](https://github.com/Azure/azure-sphere-v2-nre-yocto/blob/c0df5a2978563d21c6dfbe80704de740b67b337a/docs/4-deploying-to-imx8ulp.md) for `deployin to imx8ulp`
3. Copy the ARE image using [Azure Sphere documentation](https://github.com/Azure/azure-sphere-v2-nre-yocto/blob/c0df5a2978563d21c6dfbe80704de740b67b337a/docs/5-booting-are-vm.md) for`sync-are-to-device`
4. Connect to the NRE through SSH.
5. Devices come up with unique hostnames in the form azsphere-XXXXXXXX.local. Names are reset each time the device is reflashed but stay the same between reboots. To find your devices name you have two options:

4.1.Connect to the serial console. You should see the hostname on the login prompt, like this:

```bash
Azure Sphere V2 NRE 2209 azsphere-cf5ab0a1ff83a841 ttyAMA0

azsphere-cf5ab0a1ff83a841 login: 
```
4.2.Use avahi-browse to discover the device (this command is run on the Dev PC):

```bash
avahi-browse -at

+ virbr0 IPv4 azsphere-a5317e1508791b21                     SSH Remote Terminal  local
```
5. Connect to the ARE through SSH. (e.g. `ssh -J admin@azsphere-1e88164535ed4f55.local root@10.10.10.2`) 
6. use password when prompted

## Demo 1: GPIO Device sysfs access

### Goal
Demonstrate the use of `arectl` to enable GPIO pins peripheral passthrough to the ARE.

### Steps

1. Connect to the NRE through SSH.
2. Enable GPIO passthrough:

```bash
#GPIOF PIN 20 controls GREEN LED
#GPIOF PIN 29 controls RED   LED

sudo arectl peripherals enable GPIOF PTF20=on PTF29=on
```
3. Reboot the device for the overlays to be applied: `sudo reboot now`

4. Check GPIOF settigns

```bash
arectl peripherals details GPIOF

Name:                GPIOF
Description:         GPIO instance F in the Application Domain
Used Resources:     
                     GPIOF
Data Files:         
                     /boot/0001-GPIOF-nre.dtbo
                     /run/images/active_are/0001-GPIOF-are.dtbo
Options:            
               ...
               PTF20 on
               PTF29 on
               ...
```
5. We can check from ARE terminal the current state of GPIO settings in the system

```bash
cat /sys/kernel/debug/gpio

#From the output we detect the GPIO Base for GPIOF 

gpiochip0: GPIOs 480-511, parent: platform/2d010080.gpio, 2d010080.gpio:
```
6. enabling GPIOs 20/29 in sysfs

```bash
#Each GPIO must be individually activated.
#base 480 + 20 for GREEN LED base
echo 500 > /sys/class/gpio/export

#base 480 + 29 for RED   LED base
echo 509 > /sys/class/gpio/export

#GPIO set as output 480 + 20 for GREEN LED base
echo out > /sys/class/gpio/gpio500/direction

#GPIO set as output 480 + 29 for RED   LED base
echo out > /sys/class/gpio/gpio509/direction

#Turn ON GPIO 20 (GREEN LED)
echo 1 > /sys/class/gpio/gpio500/value     

#Turn 0FF GPIO 20
echo 0 > /sys/class/gpio/gpio500/value

#Turn ON GPIO 29 (RED LED)
echo 1 > /sys/class/gpio/gpio509/value     

#Turn 0FF GPIO 29 
echo 0 > /sys/class/gpio/gpio509/value

```
![MicrosoftTeams-image (1)](https://user-images.githubusercontent.com/109561403/188789049-2c4e90ba-0f86-4e93-89c2-f1cab35e1216.png)

## Demo 2: Simple GPIOF-EVK-LEDs Interphase

### Goal
Demonstrate the use of `arectl` to enable GPIOF-EVK-LEDs peripheral passthrough to the ARE.

### Steps

1. Connect to the NRE through SSH.
2. Enable GPIOF-EVK-LEDs passthrough:

```bash
#GPIOF-EVK-LEDs controls
sudo arectl peripherals enable GPIOF
sudo arectl peripherals enable GPIOF-EVK-LEDs

```
3. Reboot the device for the overlays to be applied: `sudo reboot now`

4. Check GPIOF settigns

```bash
sudo arectl peripherals enable GPIOF-EVK-LEDs

Enabled peripherals:

                GPIOF: GPIO instance F in the Application Domain
       GPIOF-EVK-LEDs: EVK LEDs attached to GPIOF


```
5. We can check from ARE terminal the current state of GPIO settings in the system

```bash
cat /sys/kernel/debug/gpio

#From the output we detect the GPIO Base for GPIOF 

gpiochip0: GPIOs 480-511, parent: platform/2d010080.gpio, 2d010080.gpio:
 gpio-500 (                    |green:user          ) out lo 
 gpio-509 (                    |red:user            ) out lo
```
6. enabling GPIOF-EVK-LEDs

```bash

#Turn ON GPIO 20 (GREEN LED)
echo 1 > /sys/class/leds/green\:user/brightness 

#Turn ON GPIO 29 (RED LED)
echo 1 > /sys/class/leds/red\:user/brightness

#Turn 0FF GPIO 20

```

## Demo 3: Control GPIOs in Linux with a C application

Demonstrate the use of GPIO Controls in linux with C application in the ARE using chardev.

### Steps


1. Connect to the NRE through SSH.
2. Enable GPIO passthrough:

```bash
#GPIOF PIN 20 controls GREEN LED
#GPIOF PIN 29 controls RED   LED

sudo arectl peripherals enable GPIOF PTF20=on PTF29=on
```
3. Reboot the device for the overlays to be applied: `sudo reboot now`

4. Add required firewall rules. Run in the NRE:

```bash
sudo bash -c "cat > /etc/asfirewalld/are_development.asfirewalld.json" << 'EOF'
{
    "Environment": "are",
    "FirewallRules": [
        {
            "User": "*",
            "AllowedOutgoing": [
                "packages.microsoft.com"
            ],
            "Comment": "Mariner (t)dnf Repos"
        },
        {
            "User": "*",
            "AllowedOutgoing": [
                "pypi.python.org",
                "pypi.org",
                "pythonhosted.org",
                "files.pythonhosted.org"
            ],
            "Comment": "Python Pip Repos"
        },
        {
            "User": "*",
            "AllowedOutgoing": [
                "githubusercontent.com",
                "gist.githubusercontent.com",
                "raw.githubusercontent.com"
            ],
            "Comment": "GitHub Gist"
        }
    ]
}
EOF
```
5. Enabled the new rules by reloading the firewall: `sudo asfirewallctl appconfig reload`

6. Connect to the ARE through SSH. (e.g. `ssh -J admin@azsphere-1e88164535ed4f55.local root@10.10.10.2`)
7. Download software requirements in the ARE:

8. Search the list of packages and install build-essential package to get gcc compiler 

```bash
 tdnf list
 tdnf search build
 tdnf install build-essential
 gcc --version
```

9. <optional> install VIM editor

```bash
 tdnf install vim
```

10. Check for the presence of Linux kernel GPIO user space interface 
```bash
 ls /dev/gpiochip0 
```   

11. Create and compile a simple program to test the GPIO interface

11.1 Create a C file for example GpioCtrl.c
```bash
 > GpioCtrl.c
``` 

11.2 Edit the C file you can use this code as a simple example to experiment:  
```bash  
 vim GpioCtrl.c
 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/gpio.h>

int main () {
   // 
   // variable declarations
   //
   int                       gpioFile;
   struct gpiohandle_data    gpioData;
   struct gpiohandle_request redLed;
   struct gpiohandle_request greenLed;

   // 
   // Open gpiochip device file 
   //
   gpioFile = open ("/dev/gpiochip0", O_RDWR);
   
   if (gpioFile < 0) {
     perror("ERROR: unable to open gpiochip0");
     return -1;
   }

   // 
   // Green LED configuration (GPIOF-20)
   //
   greenLed.flags = GPIOHANDLE_REQUEST_OUTPUT;
   strncpy(greenLed.consumer_label, "GREEN LED", GPIO_MAX_NAME_SIZE);
   memset(greenLed.default_values, 0, sizeof(greenLed.default_values));

   // GPIOF pin 20 needs 1 GPIO line                                  
   greenLed.lines            = 1;    
   greenLed.lineoffsets[0]   = 20;  

   // 
   // Red LED configuration  (GPIOF-29)
   //
   redLed.flags = GPIOHANDLE_REQUEST_OUTPUT;
   strncpy(redLed.consumer_label, "RED LED", GPIO_MAX_NAME_SIZE);
   memset(redLed.default_values, 0, sizeof(redLed.default_values));

   // GPIOF pin 29 needs 1 GPIO line                       
   redLed.lines            = 1;
   redLed.lineoffsets[0]   = 29;   

   //
   // program GPIOF 20 to control Green LED
   //
   if (ioctl(gpioFile, GPIO_GET_LINEHANDLE_IOCTL, &greenLed) < 0) {
     perror("ERROR: unable to control Green LED - GPIOF 20");
     close(gpioFile);
     return -1;
   }
   
   //
   // program GPIOF 29 to control Red LED
   //
   if (ioctl(gpioFile, GPIO_GET_LINEHANDLE_IOCTL, &redLed) < 0) {
     perror("ERROR: unable to control RED LED - GPIOF 29");
     close(gpioFile);
     close(greenLed.fd); // close green led file handler
     return -1;
   }

   //
   // set Green LED
   //
   gpioData.values[0] = 1;
   if (ioctl(greenLed.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &gpioData) < 0) {
     perror ("ERROR: unable to set GREEN LED - GPIOF 20");
   }

   // insert 2s wait to see the result of the led action
   sleep (2);

   //
   // clear Green LED
   //
   gpioData.values[0] = 0;
   if (ioctl(greenLed.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &gpioData) < 0) {
     perror ("ERROR: unable to set GREEN LED - GPIOF 20");
   }

   // insert 2s wait to see the result of the led action
   sleep (2);

   //
   // set Red LED
   //
   gpioData.values[0] = 1;
   if (ioctl(redLed.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &gpioData) < 0) {
     perror ("ERROR: unable to set RED LED - GPIOF 29");
   }

   // insert 2s wait to see the result of the led action
   sleep (2);

   //
   // clear Red LED
   //
   gpioData.values[0] = 0;
   if (ioctl(redLed.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &gpioData) < 0) {
     perror ("ERROR: unable to set RED LED - GPIOF 29");
   }
 
   //                                                                         
   // close device file handles
   //
   close(gpioFile);
   close(redLed.fd);
   close(greenLed.fd);

   // exit program
   return 0;
}

``` 

11.3 Compile & execute the application:
```bash 
 gcc GpioCtrl.c -o GpioCtrl
 ./GpioCtrl 
``` 

(WIP)
