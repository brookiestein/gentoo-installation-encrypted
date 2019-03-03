__Guía de instalación de Gentoo (Personal)__

Ésta es una guía de instalación de Gentoo personal, es decir, todo lo que ella tenga son cosas que yo aplico a mi sistema.
Evidentemente ésto no impide que tú, querido lector, le saques provecho, así que sin más empecemos.

Si encuentras algún error, por favor, no dudes en hacermelo saber.

Ésta guía es una especie de fork de una que ya había hecho, con la peculiaridad de que en esta explico cómo hacer la instalación con LUKS y LVM. ¿Por qué lo hice así? Pues porque no a todos les interesa hacer una instalación con LVM y LUKS. Bien una vez aclarado esto, podemos iniciar...

Lo primero que hago/haremos es bootear un USB con la iso de Gentoo. La cuál puede ser encontrada en la [página oficial de Gentoo](https://www.gentoo.org/downloads/). El proceso de booteo varía dependiendo del sistema operativo que tengas instalado así que te dejaré a ti ese proceso, aunque sí que te daré algunas pistas: *__Si usas alguna de las tantas distribuciones GNU/Linux puedes usar el comando dd, que ya viene instalado en (casi) todas. Mientras que si usas Windows te puedo recomendar que uses "PowerISO" que es el que he utilizado y me ha funcionado bastante bien__*

Bien, ya teniendo nuestra USB booteada con Gentoo, procederemos a entrar desde ella en nuestra PC, éste paso como podrás adivinar es diferente para casi todos, pues depende de la PC y no todos tenemos la misma PC jeje, pero no es difícil, es más si quieres instalar Gentoo eso no debe ser un problema.

Llegados a éste punto toca particionar nuestro disco duro o SSD. Para éste fin yo utilizo la herramienta __"cfdisk"__ que ya viene en el livecd de Gentoo: Haremos 2 particiones, una para /boot (que no irá cifrada) y otra que sí lo irá.

```
# cfdisk
```

Particionar con cfdisk es realmente sencillo, seleccionas el espacio vacío o en el caso de que el disco ya tenga formato, pues eliminarlo con la opción __"Remove"__, *__te moverás con las flechas arriba, izquierda, abajo y derecha y seleccionas con enter__*.

Es el turno de darle un FS (FileSystem) o Sistema de Ficheros a nuestras particiones, hay muchos FS's pero yo utilizo ext*, si creaste las mismas particiones que yo (y en el orden que yo), de seguro que quedaría algo como: 

```
/dev/sda1 para boot
/dev/sda2 para LVM y LUKS
```

Bien, es momento de cifrar nuestra partición y aplicarle LVM.

```
# mkfs.ext2 /dev/sda1
# cryptsetup -y -c aes-xts-plain64 -s 512 -h sha512 --use-random luksFormat /dev/sda2
```
Es momento de explicar qué hace cada parámetro que le pasamos a **cryptsetup**
**-y o --verify-passphrase le indica que pida la contraseña dos veces y se queje si ambas no coinciden
-c es para indicar el algoritmo de cifrado
-s es para indicar el tamaño de la clave
-h es para indicar el hash
--use-random es para indicarle al kernel que genere números aleatorios utilizando la clave maestra
luksFormat es para aplicar el algoritmo LUKS
/dev/sda2 es el dispositivo sobre el cual se aplicará el cifrado.**

Ahora debemos descifrar la partición
```
# cryptsetup luksDump /dev/sda2
# crypsetup luksOpen /dev/sda2 gentoo
```

Es momento de configurar LVM. Aquí depende del tamaño de tu disco y qué espacio quieres aplicarle a cada volumen. Yo creo 3 volúmenes: swap, / y /home. Tengo un SSD de 120 GiB así que queda algo así
```
# rc-service lvm start
# pvcreate /dev/mapper/gentoo
# vgcreate gentoo /dev/mapper/gentoo
# lvcreate -C y -L 5G -n swap gentoo
# lvcreate -L 25G -n root gentoo
# lvcreate -l 100%FREE -n home gentoo
# vgchange -ay
# mkfs.ext4 /dev/mapper/gentoo-root
# mkfs.ext4 /dev/mapper/gentoo-home
# mkswap /dev/mapper/gentoo-swap
# swapon /dev/mapper/gentoo-swap
```
Antes de proceder al montado de las correspondientes particiones creo que tengo que explicar qué es lo que hemos hecho hasta el momento. Primero creamos un volumen físico (**pvcreate**), luego creamos un grupo de volúmenes (**vgcreate**), seguido creamos 3 volúmenes lógicos (**lvcreate**), luego los activamos (**vgchange**) y por último le aplicamos un FS a cada uno de ellos.

Descargar el stage (En el [Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation/es) recomiendan el stage 3, así que ese será el que descargaremos), en la misma página de dónde descargamos la iso de Gentoo está el stage 3. [__Aquí__](https://www.gentoo.org/downloads/)

Después de haber descargado el stage 3 debemos descomprimirlo:

```
# tar xfv stage3-amd64-*
```

Editar el make.conf. El archivo make.conf ubicado en /etc/portage/make.conf, es el archivo de configuración de portage, la herramienta que se utiliza para compilar los paquetes en Gentoo, asi que como adivinarás, es muy importante. Viene así:

```
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/usr/portage"
DISTDIR="/usr/portage/distfiles"
PKGDIR="/usr/portage/packages"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C
```

Yo lo dejo así:

```
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-O2 -march=native -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
MAKEOPTS="-j4"
LINGUAS="es es_ES"
L10N="es es-ES"
VIDEO_CARDS="intel"
GRUB_PLATFORMS="pc"

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR="/usr/portage"
DISTDIR="/usr/portage/distfiles"
PKGDIR="/usr/portage/packages"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C

USE="cryptsetup crypt mount truetype device-mapper"
```

Bien, es momento de explicar qué fue todo lo que le agregué:
En __COMMON_FLAGS__ le agregué: *__-march=native__* para que me compile los paquetes de manera específica para mi procesador.
En __MAKEOPTS__ debes agregarle el número de hilos de tu procesador, mi procesador es de 2 núcleos, 4 hilos, así que queda como "-j4"
En __LINGUAS__ y __L10N__ son variables para la configuración del idioma, de ahí algunas aplicaciones toman configuraciones para instalarse en tu idioma, por ejemplo.
EN __VIDEO_CARDS__ debes ponerle la tarjeta de vídeo que tienes, yo tengo una tarjeta de vídeo integrada en mi procesador intel así que queda como: *__VIDEO_CARDS="intel"__*, si tienes otra gráfica deberás ponerla, si no sabes cuál tarjeta gráfica tienes puedes guardar los cambios, salir y en el prompt escribir lo siguiente:

```
# lspci | grep VGA
```
Para obtener información sobre tu tarjeta gráfica. Y por último en __GRUB_PLATFORMS__ es para que el grub, que instalaremos más tarde sepa con qué configuración instalase, con decirle *__GRUB_PLATFORMS="pc"__* le estás diciendo que se instale con las configuraciones por defecto, ahora bien, si tienes EFI deberás realizar algunos pasos extra. Puedes informarte en el [Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation/es#Seleccionar_un_gestor_de_arranque).

Copiar la información DNS:

```
# cp -L /etc/resolv.conf /mnt/gentoo/etc
```

Montar los sistemas de ficheros necesarios para proceder
```
# mount /dev/mapper/gentoo-root /mnt/gentoo
# cd /mnt/gentoo
# mkdir boot home hostrun
# mount /dev/sda1 boot
# mount /dev/mapper/gentoo-home home
# mount --bind /run /mnt/gentoo/hostrun
# mount -t proc /proc /mnt/gentoo/proc
# mount --rbind /sys /mnt/gentoo/sys
# mount --rbind /dev /mnt/gentoo/dev
# mkdir /mnt/gentoo/usr/portage
# chroot /mnt/gentoo /bin/bash
# source /etc/profile
# export PS1="(chroot) $PS1"
# mkdir /run/lvm
# mount --bind /hostrun /run/lvm
```

Ahora vamos a instalar una instantánea de repositorio de ebuilds desde la web y actualizar el repositorio

```
# emerge-webrsync
# emerge --sync --quiet
```


Bien, ha llegado la hora de elegir un perfil para nuestra instalación. Gentoo, de manera predeterminada viene con un perfil recomendado seleccionado, ese es el que utilizo, pero tú puedes cambiarlo si quieres. Para listar todos los perfiles ejecuta:

```
# eselect profile list
```
Y para seleccionar el que desees:

```
# eselect profile set <<el_número_del_perfil>>
```

Como dije el que utilizo es el que recomienda Gentoo, que, al menos al momento de escribir ésta guía, es el número 12.

Una vez hallamos elegido el perfil toca actualizar para tener una base para nuestro sistema:

```
# emerge --ask --update --deep --newuse @world
```
Ahora vamos a configurar la zona horaria, para ver todas las zonas horarias disponibles en Gentoo, hacemos:

```
# ls /usr/share/zoneinfo
```

En éste caso usaré de ejemplo Madrid:

```
# echo "Europe/Madrid" > /etc/timezone
# emerge --config sys-libs/timezone-data
```

Ahora es el turno de configurar las localizaciones. Deberás poner tu localización al final del archivo, estilo:
```
es_ES.UTF-8 UTF-8
```

```
# nano -lw /etc/locale.gen
```

Guarda los cambios con *__CTRL + S__* y sal con *__CTRL + X__*. Ahora genera las localizaciones y actívalas con:

```
# locale-gen
# eselect locale list
# eselect locale set <<número_de_la_localización>>
# env-update && source /etc/profile && export PS1="(chroot) $PS1"
```

Bien, ha llegado quizás el momento más importante de toda la guía, la compilación del kernel/núcleo. Tienes dos opciones: 
*__Dicho en forma de broma__* Hacerlo como los hombres y compilar el kernel/núcleo de forma manual o utilizar la herramienta *__Genkernel__* que hace todo el proceso de forma automatica jeje. Bien vamos de menor a mayor, así que primero explicaré cómo hacerlo con *__Genkernel__*. Primero instalaremos algunos paquetes importantes. Éstos te servirán independientemente de la opción de compilación del kernel/núcleo que hallas elegido.

```
# emerge -a gentoo-sources genkernel-next sys-boot/grub:2 cryptsetup dhcpcd gentoolkit pciutils dosfstools ntfs3g
```

Aquí los paquetes importantes son: *__gentoo-sources, genkernel-next, sys-boot/grub:2, dhcpcd, gentoolkit y pciutils__* los otros dos son controladores para poder detectar dispositivos FAT32 (*__dosfstools__*) y dispositivos NTFS (*__ntfs3g__*). Cuándo termine de compilar esos paquetes, crea el enlace a las fuentes del kernel/núcleo:

```
# ls -l /usr/src/linux
```
Y para compilar:

```
# genkernel --lvm --luks all
```

Ahora prepara un café y descansa un poco, éste proceso durará más o menos dependiendo de la potencia de tu procesador, yo tengo un i5 2540M y me ha durado unos 40 minutos.

Bien ahora toca explicar cómo hacerlo o más bien cómo yo compilo el kernel/núcleo de forma manual, pues ésto depende de qué quieras compilar. Primero crear el enlace, entrar al directorio de las fuentes y entrar al menú de configuración:

```
# ls -l /usr/src/linux
# cd /usr/src/linux
# make menuconfig
```

Una vez aquí dentro vamos a activar algunas opciones que son necesarias para que el núcleo pueda funcionar. Algunas características necesarias Gentoo las trae activadas para hacernos la vida más fácil jeje.


__Habilitar soporte devtmpfst__
```
Device Drivers --->
  Generic Driver Options --->
    [*] Maintain a devtmpfs filesystem to mount at /dev
    [ ]   Automount devtmpfs at /dev, after the kernel mounted the rootfs
```

__Habilitar el soporte de disco SCSI__
```
Device Drivers --->
   SCSI device support  --->
      <*> SCSI disk support
```

__Seleccionar los sistemas de archivos necesarios__
```
File systems --->
  <*> Second extended fs support
  <*> The Extended 3 (ext3) filesystem
  <*> The Extended 4 (ext4) filesystem
  <*> Reiserfs support
  <*> JFS filesystem support
  <*> XFS filesystem support
  <*> Btrfs filesystem support
  DOS/FAT/NT Filesystems  --->
    <*> MSDOS fs support
    <*> VFAT (Windows-95) fs support
 
Pseudo Filesystems --->
    [*] /proc file system support
    [*] Tmpfs virtual memory file system support (former shm fs)
```

__Controladores para Ethernet y Wi-Fi__

Bien, aquí hay que hacer una pequeña pausa ya que necesitas saber qué controladores usa tu PC para Ethernet y Wi-Fi. Puedes obtener un poco de información con:
```
# lspci | grep Ethernet
# lspci | grep Network
```
Mi PC utiliza el controlador: *__Intel Wireless WiFi Next Gen AGN - Wireless-N/Advanced-N/Ultimate-N (iwlwifi)__* para la conexión inalámbrica y el controlador *__Intel (82586/82593/82596) devices__* para la conexión cableada. Si no obtienes información suficiente con el comando anterior debería buscar información en [Google](https://www.google.com/)

La sección para los controladores de red está en:
```
Device Drivers --->
  Network device support --->
```

__Activar el soporte SMP__
```
Processor type and features  --->
  [*] Symmetric multi-processing support
```

__Activar soporte para dispositivos de entrada USB__
```
Device Drivers --->
  HID support  --->
    -*- HID bus support
    <*>   Generic HID driver
    [*]   Battery level reporting for HID devices
      USB HID support  --->
        <*> USB HID transport layer
  [*] USB support  --->
    <*>     xHCI HCD (USB 3.0) support
    <*>     EHCI HCD (USB 2.0) support
    <*>     OHCI HCD (USB 1.1) support
```
__Seleccionar los tipos y las características del procesador__
```
Executable file formats / Emulations  --->
   [*] IA32 Emulation
```
__Habilitar el soporte para GPT__

```
-*- Enable the block layer --->
   Partition Types --->
      [*] Advanced partition selection
      [*] EFI GUID Partition support
```

Éste paso no es necesario si no tienes UEFI

__Habilitar el soporte para UEFI__

```
Processor type and features  --->
    [*] EFI runtime service support 
    [*]   EFI stub support
    [*]     EFI mixed-mode support
 
Firmware Drivers  --->
    EFI (Extensible Firmware Interface) Support  --->
        <*> EFI Variable Support via sysfs
```
__Habilitar el soporte para sonido (ALSA)__

Puedes encontrar información sobre tu tarjeta de sonido con:

```
# lspci | grep -i audio
```

```
Device Drivers --->
    <*> Sound card support
        <*> Advanced Linux Sound Architecture --->
            [*] PCI sound devices  --->
                Select the driver for your audio controller.
            HD-Audio  --->
                Select a codec or enable all and let the generic parse choose the right one:
                [*] Build Realtek HD-audio codec support
                [*] ...
                [*] Build Silicon Labs 3054 HD-modem codec support
                [*] Enable generic HD-audio codec parser
General setup --->
    [*] System V IPC
```

```
Device Drivers --->
    <*> Sound card support
        <*> Advanced Linux Sound Architecture --->
            [*] Dynamic device file minor numbers
```

Éste paso no es necesario si no tienes bluetooth

__Habilitar soporte para Bluetooth__

```
[*] Networking support --->
      <M>   Bluetooth subsystem support --->
              [*]   Bluetooth Classic (BR/EDR) features
              <*>     RFCOMM protocol support
              [ ]       RFCOMM TTY support
              < >     BNEP protocol support
              [ ]       Multicast filter support
              [ ]       Protocol filter support
              <*>     HIDP protocol support
              [*]     Bluetooth High Speed (HS) features
              [*]   Bluetooth Low Energy (LE) features
                    Bluetooth device drivers --->
                      <M> HCI USB driver
                      <M> HCI UART driver
      <*>   RF switch subsystem support --->
    Device Drivers --->
          HID support --->
            <*>   User-space I/O driver support for HID subsystem
            (32) Max number of sound cards
```

Y por último...

__Habilitar el soporte para contar con opciones criptográficas:__

__Habilitando el mapeador de dispositivos y el destino crypt__
```
[*] Enable loadable module support
Device Drivers --->
    [*] Multiple devices driver support (RAID and LVM) --->
        <*> Device mapper support
        <*>   Crypt target support
```

__Habilitar las funciones de la API criptográfica__
```
[*] Cryptographic API --->
    <*> XTS support
    <*> SHA224 and SHA256 digest algorithm
    <*> AES cipher algorithms
    <*> AES cipher algorithms (x86_64)
    <*> User-space interface for hash algorithms
    <*> User-space interface for symmetric key cipher algorithms
```

__Habilitar el soporte de initramfs__
```
General setup  --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
```

__Habilitación de la compatibilidad con tcrypt (TrueCrypt / tcplay / VeraCrypt)__
```
Device Drivers --->
    [*] Block Devices ---> 
        <*> Loopback device support 
File systems ---> 
     <*> FUSE (Filesystem in Userspace) support 
[*] Cryptographic API ---> 
     <*> RIPEMD-160 digest algorithm 
     <*> SHA384 and SHA512 digest algorithms 
     <*> Whirlpool digest algorithms 
     <*> LRW support 
     <*> Serpent cipher algorithm 
     <*> Twofish cipher algorithm
```

Bien, una vez hecho todos éstos pasos es hora de poner a compilar nuestro kernel/núcleo:

```
# make
# make modules_install
# make install
# genkernel --makeopts=-j4 --lvm --luks --install initramfs
```

*__make__* es para compilar el kernel/núcleo.

*__make modules_install__* es para compilar los módulos.

*__make install__* es para instalar el kernel.

*__genkernel --lvm --luks --install initramfs__* es para crear el initramfs con soporte para lvm y luks.

Bien, ya con eso tendríamos nuestro kernel/núcleo listo, ahora toca editar el archivo fstab ubicado en *__/etc/fstab__* que sirve para montar las particiones automáticamente al encender nuestra PC. Si tienes conocimientos, puedes editarlo por tu cuenta, sin embargo, sino, puedes dejarlo como lo dejo yo y no tendrás ningún problema, *__(Estoy tomando en cuenta de que haz seguido al pie de la letra la guía o al menos el punto de las particiones)__*

```
# nano -lw /etc/fstab
 ```
 
 Mi archivo fstab está así, puedes eliminar tranquilamente la última línea si no estás utilizando un SSD:
 
 ```
    # /etc/fstab: static file system information.
#
# noatime turns off atimes for increased performance (atimes normally aren't 
# needed); notail increases performance of ReiserFS (at the expense of storage 
# efficiency).  It's safe to drop the noatime options if you want and to 
# switch between notail / tail freely.
#
# The root filesystem should have a pass number of either 0 or 1.
# All other filesystems should have a pass number of 0 or greater than 1.
#
# See the manpage fstab(5) for more information.
#

# <fs>			<mountpoint>	<type>		<opts>		<dump/pass>

# NOTE: If your BOOT partition is ReiserFS, add the notail option to opts.
#
# NOTE: Even though we list ext4 as the type here, it will work with ext2/ext3
#       filesystems.  This just tells the kernel to use the ext4 driver.
#
# NOTE: You can use full paths to devices like /dev/sda3, but it is often
#       more reliable to use filesystem labels or UUIDs. See your filesystem
#       documentation for details on setting a label. To obtain the UUID, use
#       the blkid(8) command.

#LABEL=boot		/boot		ext4		noauto,noatime	1 2
#UUID=58e72203-57d1-4497-81ad-97655bd56494		/		ext4	noatime		0 1
#LABEL=swap		none		swap		sw		0 0

/dev/sda1		              /boot		ext2		noatime,nodiratime	0 0
/dev/mapper/gentoo-swap		none		swap		defaults		        0 0
/dev/mapper/gentoo-root		/		    ext4		noatime,nodiratime	0 1
/dev/mapper/gentoo-home		/home		ext4		noatime,nodiratime	0 2

tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
```
Ahora es el momento de ponerle nombre a nuestra PC jeje, éste nombre se lo agregaremos ahora pero puedes cambiarlo en cualquier momento si así lo deseas.

```
# nano -lw /etc/conf.d/hostname
```
Introduce el nombre que desees dentro de las comillas.

Ahora sería bueno que ejecutaras

```
# ifconfig
```
Para que sepas que nombre es asignado a tu tarjeta de ethernet y así poder configurarla para utilizar *__DHCP__*  y poder acceder a internet, al menos hasta que instales *__NetworkManager__* o cualquier otro gestor de red. En mi caso es asignado el nombre *__"eno1"__*, así que quedaría más o menos así:

```
# nano -lw /etc/conf.d/net
```

Dentro del archivo pones: 
```
config_eno1="dhcp"
Evidentemente *__eno1__* lo cambias por el nombre que se le asignó a tu tarjeta de ethernet.
```
Bien, ya casi hemos terminado, pero no te desconcentres porque ahora vamos por otro de los puntos más importantes y es el de asignar una contraseña al usuario root.

```
# passwd
```
Ponle la que desees, en el proceso no verás nada, pero de seguro eso ya lo sabes jeje.

Ahora estableceremos la distribución que tendrá nuestro teclado en la's tty's. Aquí tienes que tener cuidado, pues una mala configuración puede ocasionar resultados extraños cuándo estemos teclando.

```
# nano -lw /etc/conf.d/keymaps
```
Yo utilizo como distribución: *__Inglés (Estados Unidos Internacional)__* y el que trae por defecto sólo es *__Inglés__* así que, en mi caso, quedaría más o menos de la siguiente forma:

```
# Use keymap to specify the default console keymap.  There is a complete tree
# of keymaps in /usr/share/keymaps to choose from.
keymap="us-acentos"

# Should we first load the 'windowkeys' console keymap?  Most x86 users will
# say "yes" here.  Note that non-x86 users should leave it as "no".
# Loading this keymap will enable VT switching (like ALT+Left/Right)
# using the special windows keys on the linux console.
windowkeys="YES"

# The maps to load for extended keyboards.  Most users will leave this as is.
extended_keymaps=""
#extended_keymaps="backspace keypad euro2"

# Tell dumpkeys(1) to interpret character action codes to be
# from the specified character set.
# This only matters if you set unicode="yes" in /etc/rc.conf.
# For a list of valid sets, run `dumpkeys --help`
dumpkeys_charset=""

# Some fonts map AltGr-E to the currency symbol instead of the Euro.
# To fix this, set to "yes"
fix_euro="NO"
```

Aquí el punto importante es arriba dónde dice: *__keymap="us"__*

Ahora vamos a configurar el reloj del sistema. Por defecto viene en *__UTC__* pero lo puedes cambiar por el que quieras, yo ésto lo dejo así.

```
# nano -lw /etc/conf.d/hwclock
```

Es el turno del grub, toca instalación y configuración.
```
# nano -lw /etc/default/grub
```
Ubicas la línea que tiene **GRUB_CMDLINE_LINUX** y pones lo siguiente:
```
GRUB_PRELOAD_MODULES=lvm
GRUB_ENABLE_CRYPTODISK=y
GRUB_DEVICE=/dev/ram0
GRUB_CMDLINE_LINUX="crypt_root=/dev/sda2 real_root=/dev/mapper/gentoo-root rootfstype=ext4 dolvm quiet splash"
```

```
# grub-install --modules="linux crypto search_fs_uuid luks lvm" --recheck /dev/sda
```

Y la configuración
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

Si te muestra algunos errores, haz lo siguiente:
```
# exit
# mount --bind /run /mnt/gentoo/hostrun
# chroot /mnt/gentoo /bin/bash
# mount --bind /hostrun/lvm /run/lvm
```
Y regeneras el archivo...
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

Activar el servicio de lvm al inicio:
```
# rc-update add lvm boot
```

Salimos del chroot y desmontamos las particiones:

```
# exit
# cd ..
# cd ..
# umount -l /mnt/gentoo/dev{/pts,/shm,}
# umount -R /mnt/gentoo
```
Reiniciar...
```
# reboot
```

Felicidades, ya tienes Gentoo instalado, ahora queda hacer alguna que otra modificación para empezar a compilar e instalar los paquetes que necesitemos.

Primero, crear un usuario a parte del root, pues como seguro sabrás no es recomendable utilizar siempre el usuario root, es mejor tener un usuario "normal" que pueda invocar permisos de root sólo cuándo sea necesario. Nos logeamos con el user root y la contraseña que específicamos en la instalación.

```
# useradd -m -G users,wheel,audio,video,usb,portage,cdrom -s /bin/bash <<nombre_de_usuario>>
# passwd <<nombre_de_usuario>>
```

Ahora instalamos *__sudo__* para poder utilizar permisos de administrador con nuestro usuario:

```
# emerge -a sudo
```

Editamos el fichero: *__/etc/sudoers__* y descomentamos la línea: # &wheel ALL=(ALL) ALL
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```

Cerramos sesión de la cuenta de root y nos logeamos con el usuario que acabamos de crear:

```
# exit
```

Y el último paso de ésta guía es instalar el xorg para poder instalar el DE o WM que deseemos.

```
# sudo emerge -a xorg-x11 xinit twm xconsole xclock xterm
```

Ahora instalas los programas que desees/necesites, ejemplo: VLC, Firefox, etc. Podrás encontrar más información en la [Wiki de Gentoo](https://wiki.gentoo.org/wiki/Main_Page) y buscando en la WWW. Ésto sólo fue una pequeña introducción, tienes todo un universo que descubrir con Gentoo. Así que con ésto me despido. Disfuta tu Gentoo!

__No necesitas suerte, sólo determinación y esfuerzo.__
