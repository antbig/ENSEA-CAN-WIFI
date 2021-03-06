# Nitrogen8M passerelle CAN <> Wi-Fi

## Installation du rootfs
Téléchargement de yocto sur [https://boundarydevices.com/yocto-release-for-i-mx8mq/](https://boundarydevices.com/yocto-release-for-i-mx8mq/)
Puis pour effectuer le transfert sur la emmc [https://boundarydevices.com/programming-emmc-on-i-mx8/](https://boundarydevices.com/programming-emmc-on-i-mx8/)

Pour cela, un va utiliser le USB Mass Storage pour transférer yocto sur l’emmc.
```shell
U-Boot > ums mmc 0
```
```shell
user@ubuntu:~$ zcat 20191101-boundary-image-multimedia-full-nitrogen8m.wic.gz | sudo dd of=/dev/sdb bs=1M
```

## Installation du serveur TFTP
Pour installer et configurer le serveur TFTP, nous avons suivit un [super tutoriel](https://mohammadthalif.wordpress.com/2010/03/05/installing-and-testing-tftpd-in-ubuntudebian/).
A la suite de quoi nous disposons du dossier /tftpboot.

## Compilation du noyau linux

Même si la version 4.14 est disponible, on va utiliser la version 4.9 car le driver wifi pour le module QCA9377 ne fonctionne pas en 4.14.

```shell
user@ubuntu:~$ sudo apt-get install crossbuild-essential-arm64 ncurses-dev
user@ubuntu:~$ git clone https://github.com/boundarydevices/linux-imx6
user@ubuntu:~$ cd linux-imx6
user@ubuntu:~/linux-imx6$ export ARCH=arm64
user@ubuntu:~/linux-imx6$ export CROSS_COMPILE=aarch64-linux-gnu-
user@ubuntu:~/linux-imx6$ make boundary_defconfig
user@ubuntu:~/linux-imx6$ make menuconfig
```
Un interface va s’ouvrir pour configurer les différents éléments que l’on souhaite avoir dans le noyau linux ou dans les modules. [M] indique que l’élément va être dans un module. [*] indique que l’élément va être dans le noyau.
> \> Networking support > Wireless > [*]Generic IEEE 802.11 Networking Stack (mac80211)

> \> Device Drivers > USB support > USB Serial Converter support > <\*>  USB FTDI Single Port Serial Driver

>  \> Networking support > CAN bus subsystem support > Raw CAN Protocol (raw access with CAN-ID filtering)

>  \> Networking support > CAN bus subsystem support > Broadcast Manager CAN Protocol (with content filtering)

>  \> Networking support > CAN bus subsystem support > CAN Gateway/Router (with netlink configuration)

On peut maintenant compiler le noyau Linux
```shell
user@ubuntu:~/linux-imx6$ make -j8
user@ubuntu:~/linux-imx6$ cp ./arch/arm64/boot/Image /tftpboot/
user@ubuntu:~/linux-imx6$ cp ./arch/arm64/boot/dts/freescale/imx8mq-nitrogen8m.dtb /tftpboot/
```
Les 2 dernières commandes permettent de copier le noyau compilé dans le dossier du serveur tftp

## Mise en place du noyau sur la carte
Il faut arrêter le lancement par défaut de U-boot en appuyant sur une touche.

```shell
U-Boot > env default –a
U-Boot > setenv ethaddr 00:19:B8:05:25:45
U-Boot > setenv ipaddr 10.10.3.212
U-Boot > setenv serverip 10.10.3.243
U-Boot > setenv netmask 255.255.254.0
U-Boot > saveenv
U-Boot > tftpboot 0x40480000 Image
U-Boot > tftpboot 0x43000000 imx8mq-nitrogen8m.dtb
U-Boot > setenv bootargs console=ttymxc0,115200 root=/dev/mmcblk0p2
U-Boot > booti 0x40480000 - 0x43000000
```
Il est possible de gagner du temps en créant un script de démarrage qui contient l’ensemble des instructions à exécuter.
```shell
U-Boot > setenv br 'tftpboot 0x40480000 Image;tftpboot 0x43000000 imx8mq-nitrogen8m.dtb;setenv bootargs console=ttymxc0,115200 root=/dev/mmcblk0p2;booti 0x40480000 - 0x43000000'
U-Boot > saveenv
U-Boot > run br
```
Pour les fois suivantes, il faut juste exécuter la commande ‘run br’

## Compilation et installation du driver WIFI
On va commencer par la compilation du driver WIFI
```shell
user@ubuntu:~$ git clone https://github.com/boundarydevices/qcacld-2.0 -b boundary-LNX.LEH.4.2.2.2
user@ubuntu:~$ cd qcacld-2.0/
user@ubuntu:~/qcacld-2.0$ export ARCH=arm64
user@ubuntu:~/qcacld-2.0$ export CROSS_COMPILE=aarch64-linux-gnu-
user@ubuntu:~/qcacld-2.0$ KERNEL_SRC=/home/user/linux-imx6 CONFIG_CLD_HL_SDIO_CORE=y make
```
Une fois la compilation terminée, le driver est présent sous la forme du fichier 'wlan.ko'. Pour le transférer dans le rootfs, nous allons encore une fois utiliser le serveur tftp.
```shell
user@ubuntu:~/qcacld-2.0$ cp wlan.ko /tftpboot/
```
Ayant utilisé le rootfs de yocto, nous devons changer le nom du dossier qui contient les modules utilisés par le noyau linux pour correspondre à notre version du noyau.
Pour cela, nous devons dans un premier temps obtenir le nom de la version du noyau que nous utilisons:
```shell
root@Nitrogen8m:~$ uname -r
4.9.128-g75784999a272
```
Une fois le nom attendu obtenu, nous pouvons renommer le dossier:
```shell
root@Nitrogen8m:~$ mv /lib/modules/4.14.xxxx /lib/modules/4.9.128-g75784999a272
``` 
Il ne reste plus qu'à transférer le driver wifi 
```shell
root@Nitrogen8m:~$ tftp -g -r wlan.ko 10.10.3.243
root@Nitrogen8m:~$ cp ./wlan.ko /lib/modules/4.9.128-g75784999a272/extra/
``` 
Il faut ensuite redémarrer la carte pour voir apparaître l'interface réseau "wlan0". Pour configurer l'interface wifi nous allons utiliser la commande "nmcli" qui permet d’interagir avec le "network manager".
```shell
root@Nitrogen8m:~$ nmcli device wifi rescan
root@Nitrogen8m:~$ nmcli device wifi list
root@Nitrogen8m:~$ nmcli device wifi connect MyWifi password MyPassword
```
## Mise en place du Bus CAN
Pour communiquer sur le bus CAN, nous allons utiliser une passerelle [CANUSB](https://www.kanda.com/products/Lawicel/CANUSB.html). Nous avons déjà activé les différents driver CAN lors de la configuration du noyau linux. Il nous reste donc à cross-compiler les utilitaires CAN pour pouvoir configurer correctement le bus.
```shell
user@ubuntu:~$ sudo apt-get install autoconf libtool
user@ubuntu:~$ git clone https://github.com/linux-can/can-utils.git
user@ubuntu:~$ cd can-utils
user@ubuntu:~/can-utils$ ./autogen.sh
user@ubuntu:~/can-utils$ ./configure --host=aarch64-linux-gnu
```
Une fois la compilation terminée, il faut transférer les fichiers "slcand", "candump" et "cansend" (en utilisant le serveur TFTP) sur la carte nitrogen. 
Nous pouvons maintenant configurer le bus can en utilisant l'utilitaire "slcand":
```shell
root@Nitrogen8m:~$ tftp -g -r slcand 10.10.3.243
root@Nitrogen8m:~$ tftp -g -r candump 10.10.3.243
root@Nitrogen8m:~$ tftp -g -r cansend 10.10.3.243
root@Nitrogen8m:~$ ./slcand -o -s4 -t hw -S 3000000 /dev/ttyUSB0
root@Nitrogen8m:~$ ip link set up slcan0
``` 
Les paramètres de la commande slcand sont "-s4" pour indiquer que l'on souhaite une vitesse de transmission de 125 kbit/s, "-S 3000000" pour dire que l'on veut une vitesse de communication USB de 3Mbit/s. Enfin il faut activer l'interface avec la commande "ip".

On peut effectuer des tests pour vérifier que l'on arrive à communiquer avec le bus CAN en utilisant les utilitaires "candump" et "cansend".
```shell
root@Nitrogen8m:~$ ./candump slcan0
root@Nitrogen8m:~$ ./cansend slcan0 123#DEADBEEF
``` 
## Compilation de libwebsokets

L'objectif est de cross compiler un serveur websocket utilisant la librairie [libwebsokets](https://github.com/warmcat/libwebsockets) pour l’exécuter sur une carte développement [Nitrogen8M](https://boundarydevices.com/product/nitrogen8m/).

```shell
user@ubuntu:~$ sudo apt-get install crossbuild-essential-arm64 ncurses-dev
user@ubuntu:~$ git clone https://github.com/warmcat/libwebsockets.git
user@ubuntu:~$ cd libwebsockets
user@ubuntu:~/libwebsockets$ mkdir build
user@ubuntu:~/libwebsockets$ cd build
```
Il faut créer le fichier de configuration pour le cmake: **aarch64-linux-gnu.cmake**
Il faut modifier 
```CMake
# CMake Toolchain file for crosscompiling on ARM.
# This can be used when running cmake in the following way:
# cd build/
# cmake .. -DCMAKE_TOOLCHAIN_FILE=./aarch64-linux-gnu.cmake

# Target operating system name.
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)


# Name of C compiler.
set(CMAKE_C_COMPILER "aarch64-linux-gnu-gcc")
set(CMAKE_CXX_COMPILER "aarch64-linux-gnu-g++")

if (CMAKE_BUILD_TYPE MATCHES RELEASE OR CMAKE_BUILD_TYPE MATCHES Release OR CMAKE_BUILD_TYPE MATCHES release)
set(CMAKE_C_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE} -O2)
set(CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE} -O2)
endif()

#-nostdlib
SET(CMAKE_C_FLAGS "-DGCC_VER=\"\\\"$(GCC_VER)\\\"\" -DARM64=1 -D__LP64__=1 -Os -g3 -fpie -mstrict-align -DOPTEE_DEV_KIT=../../../../out/arm-plat-hikey/export-ta_arm64/include -I../../../../lib/libutee/include -fPIC -ffunction-sections -fdata-sections -I../../../../core/include" CACHE STRING "" FORCE)

# Where to look for the target environment. (More paths can be added here)
set(CMAKE_FIND_ROOT_PATH "/home/user/linux-imx6")

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)

set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

```shell
user@ubuntu:~/libwebsockets/build$ cmake .. -DCMAKE_TOOLCHAIN_FILE=./cross-aarch64.cmake -DLWS_WITH_SSL=OFF
user@ubuntu:~/libwebsockets/build$ make
```
Il ne reste plus qu'à attendre la fin de la compilation.
![](https://lh3.googleusercontent.com/fNW79bmHwaT3bNmBuQoCa88_g3S0UaDPd8tWC7rsy8bXDw-crZDfREevy-KIk4x5hRt_-4JwT4YifkKE9iRnDe8OrTC10WKxn9kFwVCQ9Nkw8QBsTbWVhJcgEV3WaeOnbxs_kp6n)La compilation va générer plusieurs fichiers et dossier.

![](https://lh3.googleusercontent.com/JhHroOicCc43uoCZhcktq2taiiztEYtMjiBqmm5cMfkxWdexMltXobU3NLGsnDuY0JS43qrKi-gsILaixXOuXemLqFHhfe5MEwS0-x6S_GG3zgNqEMmx3tGzQ8OvuBn-qxH1meXx)

Le dossier “bin” contient les exécutables pour les programmes d’exemple qui sont compilé en même temps que la librairie.
Le dossier “include” contient les fichiers .h qui définissent l’ensemble des fonctions de la librairie.
Le dossier “lib” contient la librairie en elle même. Elle est composé d’un .so et d’un .a.
Enfin le dossier “share” contient les fichiers web utilisés par les programmes d'exemple.


Il faut installer la librairie sur notre carte, pour cela nous transférons les fichiers “libwebsockets.so.15” et “libwebsockets.a” sur la carte en utilisant TFTP. Une fois les fichiers sur la carte, il faut les mettres dans le dossier /usr/lib.
Par le suite, nous pouvons compiler un projet souhaitant utiliser la libwebsockets avec l'instruction suivante:
```shell
user@ubuntu:~$ aarch64-linux-gnu-gcc -o myExe main.c -I./includes/ -L./lib/ -lwebsockets
```
Nous avons remarqué que l'ordre des paramètres est très important et que  leurs inversion résulte en un échec de la compilation
## Compilation du projet
Pour compiler le projet, un simple make suffit. Un obtient le fichier server qu'il faut mettre sur la carte nitrogen (en utilisant le serveur tftp) ainsi que le dossier "[mount-origin](https://github.com/antbig/ENSEA-CAN-WIFI/tree/master/mount-origin "mount-origin")" qui contient l'ensemble des fichiers web utilisé par le serveur.
```shell
root@Nitrogen8m:~$ chmod +x server
root@Nitrogen8m:~$ ./server
``` 
Un fois le serveur lancé nous pouvons nous rendre sur l’adresse ip de la carte nitrogen pour voir le site web et ainsi communiquer avec le bus CAN.
![https://i.antbig.fr/huYRPW.png](https://i.antbig.fr/huYRPW.png)

