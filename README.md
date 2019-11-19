**Guía de instalación de Gentoo (Personal)**

Esta es una guía de instalación de Gentoo personal, es decir, todo lo que ella tenga son cosas que yo aplico a mi sistema.
Evidentemente esto no impide que tú, querido lector, le saques provecho.

**La copia y la redistribución de este documento está permitida, más no la modificación, pues como ya he mencionado es 
una guía que hago a modo de bloc de notas para mis instalaciones del sistema operativo Gentoo GNU/Linux.**

Si encuentras algún error, por favor, no dudes en hacermelo saber.

Esta guía es una especie de fork de una que ya había hecho, con la peculiaridad de que en esta explico cómo hacer la 
instalación con LUKS y LVM. ¿Por qué lo hice así? Pues porque no a todos les interesa hacer una instalación con LVM y LUKS.

Bien una vez aclarado esto, podemos iniciar...

**1. "Creación" de un pendrive autoarrancable con el archivo [ISO](https://es.wikipedia.org/wiki/Imagen_ISO) de Gentoo:**

Lo primero que hago/haremos es hacer que nuestro pendrive sea "autoarrancable" con la iso de Gentoo.

La cual puede ser encontrada en [su página oficial, sección de descargas](https://www.gentoo.org/downloads/).

El proceso de "autoarranque" varía dependiendo del sistema operativo que tengas instalado así que te 
dejaré a ti ese proceso, aunque sí que te daré algunas pistas:

**Si usas alguna de las tantas distribuciones GNU/Linux puedes usar el comando dd, que ya viene instalado en (casi) todas.** 

**Mientras que si usas Windows te puedo recomendar que uses "PowerISO" que es el que he utilizado y me ha funcionado bastante bien**

Bien, ya teniendo nuestra USB **"autoarrancable"** con Gentoo, procederemos a entrar desde ella en nuestro ordenador.

Este paso como podrás adivinar es diferente para casi todos, pues depende de la PC y no todos tenemos el mismo ordenador jeje.

Pero no es difícil. Es más, si quieres instalar Gentoo eso no debe ser un problema.

Bien, como haremos una instalación cifrada, algo muy recomendable es sobreescribir el disco con datos aleatorios o de 
ceros para hacer más difícil la recuperación de archivos que pudieran haber con anterioridad.

Si tienes un SSD ten cuidado, pues como seguro sabrás estos se degradan a cada escritura.

Siguiendo con las advertencias he de decir que este proceso durará mucho, dependiendo de qué tan grande sea tu 
disco durará más o menos.

**2. "Rellenar" el dispositivo donde se instalará Gentoo con datos pseudoaleatorios o de ceros:**

Primero verificamos los discos que tenemos en nuestro ordenador
```
# fdisk -l
```
Dependiendo del disco en el que harás la instalación, sustituyes **/dev/sda.**
```
# dd if=/dev/zero of=/dev/sda bs=4M status=progress
```
Bueno, cierra los ojos por un momento o sal afuera a tomar un poco de aire y vuelve en unos minutos, 
pues como te dije este proceso durará mucho.

**3. Particionado:**

Llegados a éste punto toca particionar nuestro disco duro o SSD. Para este fin yo utilizo la 
herramienta **"cfdisk"** que ya viene en el **livecd** de Gentoo:

**3.1 Haremos 2 particiones:**
```
1. /boot (que no irá cifrada) (en esta partición irá el cargador de arranque. En mis instalaciones suelo 
utilizar GRUB, pero si utilizas otro no hay problemas.)
2. Otra que sí lo irá, donde "crearemos" un volumen físico, grupo de volúmenes y algunos volúmenes lógicos 
para: /, swap y /home.
```
```
# cfdisk
```
Particionar con **cfdisk** es relativamente sencillo, seleccionas el espacio vacío o en el caso de que el disco 
ya tenga formato, pues eliminarlo con la opción **"Remove"**, **te moverás con las flechas arriba, izquierda, 
abajo y derecha y seleccionas con enter**.

Yo tengo un SSD de 120 GiB y creo estas particiones:
```
/dev/sda1 para boot : 256 MiB
/dev/sda2 con LVM y LUKS : Todo el espacio restante.
```

**4. Formateo de particiones y (des)cifrado:**

Formateamos la partición donde irá **/boot** con ext2 y ciframos la otra:
```
# mkfs -t ext2 /dev/sda1
# cryptsetup -y -c aes-xts-plain64 -s 512 -h sha512 -i 5000 --use-random luksFormat /dev/sda2
```
Es momento de explicar qué hace cada parámetro que le pasamos a **cryptsetup**
```
-y le indica que pida la contraseña dos veces y se queje si ambas no coinciden
-c es para indicar el algoritmo de cifrado (en este caso: aes-xts-plain64)
-s es para indicar el tamaño de la clave (en este caso: 512 bits)
-h es para indicar el hash (en este caso: sha512)
-i es para indicar el tiempo que tardará en descifrar la partición
--use-random es para indicarle al kernel que genere números aleatorios utilizando la clave maestra
luksFormat es para aplicar el algoritmo LUKS
/dev/sda2 es el dispositivo sobre el cual se aplicará el cifrado.
```
**4.1 Descifrado:**

Ahora debemos descifrar la partición para poder seguir con la instalación:

```
# cryptsetup luksDump /dev/sda2
# cryptsetup luksOpen /dev/sda2 gentoo
```
```
1. luksDump:"vuelca" las cabeceras de LUKS en el dispositivo (en este caso /dev/sda2)
2. luksOpen: desciframos el dispositivo. Este parámetro requiere de dos parámetros adicionales:
2.1: Dispositivo cifrado con LUKS
2.2: Nombre identificativo del dispositivo. Puede ser cualquiera. En este caso he elegido "gentoo".
```
**5. Configuración de LVM y formateo de volúmenes:**

Es momento de configurar LVM. Aquí depende del tamaño de tu disco y qué espacio quieres aplicarle a cada volumen.

Yo creo 3 volúmenes: swap, / y /home. Tengo un SSD de 120 GiB así que queda algo así:

5 GiB para swap, 25 GiB para root y todo el espacio restante para home
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

# mount /dev/mapper/gentoo-root /mnt/gentoo

# cd /mnt/gentoo

# mkdir boot home hostrun

# mount /dev/sda1 boot

# mount /dev/mapper/gentoo-home home
```
Antes de proceder al montado de las correspondientes particiones creo que debo explicar qué es
lo que hemos hecho hasta el momento.
```
1. "Creamos" un volumen físico;

2. "Creamos" un grupo de volúmenes;

3. "Creamos" 3 volúmenes lógicos;

4. Los activamos, y;

5. Le aplicamos un sistema de archivos a cada uno de ellos y los montamos.
```
**Nota: La carpeta "hostrun" no se utilizó en este momento, pero servirá para montar /run/ en ella y 
así tener los metadatos de lvm, que nos servirán al momento de instalar y configurar el cargador de arranque GRUB.**

**6. Descarga e instalación del stage 3 ofrecido por el [equipo de Gentoo](https://gentoo.org/inside-gentoo/developers/):**
Descargar el stage (En el [Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation/es) recomiendan 
el stage 3, así que ese será el que descargaremos).

En la misma página de dónde descargamos la iso de Gentoo está el stage 3. [Aquí](https://www.gentoo.org/downloads/)

Después de haber descargado el stage 3 debemos descomprimirlo:

```
# tar -vpx --numeric-owner --xattrs-include"*.*" -f stage3-amd64-*
```

Para saber qué hace cada parámetro que se utilizó de **tar**, leer su correspondiente **manual.**

**7. Editar el archivo make.conf:**

El archivo **make.conf** ubicado en **/etc/portage/make.conf**, es el archivo de configuración de portage, 
la herramienta que se utiliza para compilar los paquetes en Gentoo, asi que como adivinarás, es muy importante.

Viene así:
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

Yo lo dejo así de la siguiente forma al momento de la instalación, luego lo modifico con todas las
opciones que utilizo. Si quieres saber cuales son, 
[échale un ojo a mi repositorio dotfiles en la sección de gentoo](https://github.com/brookiestein/dotfiles/)
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

USE="bash bluetooth crypt cryptsetup dbus device-mapper mount symlink truetype"
```
Bien, es momento de explicar qué fue todo lo que le agregué:

En __COMMON_FLAGS__ le agregué: *__-march=native__* para que me compile los paquetes de manera 
específica para mi procesador.

En __MAKEOPTS__ debes agregarle el número de hilos de tu procesador, mi procesador es de 2 núcleos, 
4 hilos, así que queda como **"-j4"**

En __LINGUAS__ y __L10N__ son variables para la configuración del idioma, de ahí algunas aplicaciones 
toman configuraciones para instalarse en tu idioma, por ejemplo.

EN __VIDEO_CARDS__ debes ponerle la tarjeta de vídeo que tienes, yo tengo una tarjeta de vídeo integrada 
en mi procesador intel así que queda como:

*__VIDEO_CARDS="intel"__*, si tienes otra gráfica deberás ponerla, si no sabes cuál tarjeta gráfica tienes 
puedes guardar los cambios, salir y en el prompt escribir lo siguiente:

```
# lspci | grep VGA
```
Para obtener información sobre tu tarjeta gráfica.

En __GRUB_PLATFORMS__ es para que **GRUB** (que instalaremos más tarde) sepa con qué 
configuración instalarse, con decirle:

*__GRUB_PLATFORMS="pc"__* le estás diciendo que se instale con las configuraciones por defecto, 
ahora bien, si tienes EFI deberás realizar algunos pasos extra.

Puedes informarte en el [Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation/es#Seleccionar_un_gestor_de_arranque).

Y, por último, **USE ("la navaja suiza de Gentoo"):** aquí es dónde van todos los valores de 
soporte que queremos agregar a nuestros programas. En este caso agregué soporte para:

**bash, bluetooth, crypt, cryptsetup, dbus, device-mapper, mount, symlink y truetype**

**Si quieres saber sobre algunos ajustes USE que podrían interesarte, lee el archivo de los mismos en:**

**/usr/portage/profiles/use.desc**


**8. Seleccionar un servidor de réplica:**
```
# mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```

**9. Configurar el repositorio de ebuilds de Gentoo:**
```
# mkdir /mnt/gentoo/etc/portage/repos.conf/
# cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

**10. Copiar la información DNS:**
```
# cp -L /etc/resolv.conf /mnt/gentoo/etc/
```
**11. Montar los sistemas de ficheros necesarios para proceder:**
```
# mount --bind /run/lvm/ /mnt/gentoo/hostrun/

# mount -t proc /proc/ /mnt/gentoo/proc/

# mount --rbind /sys/ /mnt/gentoo/sys/

# mount --rbind /dev/ /mnt/gentoo/dev/
```

**12. Entrar en "una jaula chroot":**
```

# mkdir /mnt/gentoo/usr/portage/

# chroot /mnt/gentoo /bin/bash
```

**13. Cargar las configuraciones del archivo "profile" y cambiarle el valor a la variable "PS1".**

Esto último es opcional, pero recomiendo hacerlo para ver ese **"bonito": "chroot"** en el prompt jeje.
```
# source /etc/profile

# export PS1="(chroot) $PS1"
```
**14. Montar "/hostrun/" en "/run/lvm/"**

Esto lo hacemos para contar con el archivo de metadatos de LVM que necesitaremos al momento de instalar 
el cargador de arranque más adelante:
```
# mkdir /run/lvm/

# mount --bind /hostrun/ /run/lvm/
```
**15. Instalar una instantánea y actualizar el repositorio local:**

Ahora vamos a instalar una instantánea de repositorio de ebuilds desde la web y actualizar el repositorio
```
# emerge-webrsync -v
# emerge --sync
```
**16. Elección del perfil:**

Bien, ha llegado la hora de elegir un perfil para nuestra instalación.
Gentoo, de manera predeterminada viene con un perfil recomendado seleccionado, 
ese es el que utilizo, pero tú puedes cambiarlo si quieres. Para listar todos los perfiles ejecuta:
```
# eselect profile list
```
Y para seleccionar el que desees:
```
# eselect profile set <<el_número_del_perfil>> o <<su_nombre_completo>>
```

Como dije el que utilizo es el que recomienda Gentoo, que, al menos al momento de escribir ésta guía, es el número 16.

**17. Actualizar el sistema para tener:**

Una vez hayamos elegido el perfil toca actualizar para tener una base para nuestro sistema:
```
# emerge --ask --update --deep --changed-use --verbose @world
```
o de manera abreviada:
```
# emerge -auvDU @world
```
Para más información, lee el manual de **emerge**

**18. Configuración de localización:**

Ahora vamos a configurar la zona horaria, para ver todas las zonas horarias disponibles en Gentoo, hacemos:
```
# ls /usr/share/zoneinfo
```
En éste caso usaré de ejemplo Madrid:
```
# echo "Europe/Madrid" > /etc/timezone
# emerge --config sys-libs/timezone-data
```
Ahora es el turno de configurar las localizaciones.
Deberás poner tu localización al final del archivo, estilo:
```
es_ES.UTF-8 UTF-8
```
```
# nano -lw /etc/locale.gen
```
Guarda los cambios con **CTRL + S** y sal con **CTRL + X**.

Ahora genera las localizaciones y actívalas con:
```
# locale-gen

# eselect locale list

# eselect locale set <<número_de_la_localización>>
```
Por finalizar con este paso, "refrescamos" nuestro entorno con:
```
# env-update
# source /etc/profile
# export PS1="(chroot) $PS1"
```
**19. Compilación e instalación del núcleo, módulos e initramfs:**

Bien, ha llegado quizás el momento más importante de toda la guía, la compilación del núcleo.

Tienes dos opciones:
```
1. *Dicho en forma de broma*: Hacerlo como los hombres y compilar el núcleo de forma manual, o;
2. Utilizar la herramienta: "Genkernel" que hace todo el proceso de forma automatica jeje.
```
**19.1. Compilación con Genkernel:**

Bien vamos de menor a mayor, así que primero explicaré cómo hacerlo con **Genkernel**.

**19.1.1. Instalar programas necesarios:**

Primero instalaremos algunos paquetes importantes. 
Estos te servirán independientemente de la opción de compilación del kernel/núcleo que hayas elegido.

```
# emerge -a gentoo-sources genkernel grub dhcpcd gentoolkit pciutils
```

¿Cuál es la importancia de estos paquetes?
```
1. gentoo-sources: Son las fuentes del núcleo linux, Gentoo Edition xD
2. genkernel: Es una herramienta (muy útil) para realizar varias cosas en Gentoo, entre ellas:
Instalar el núcleo de manera automática, con las opciones necesarias para arrancar, instalar un
initramfs, entre otras.
3. grub: Es un cargador de arranque muy popular entre las distribuciones GNU/Linux. Existen otros también.
4. dhcpcd: Es una utilidad para utilizar el protocolo DHCP en las conexiones de red.
5. gentoolkit: Es una herramienta, desarrollada por el equipo de Gentoo con varias utilidades, entre ellas:
eclean, que sirve para hacer "una limpieza" de los archivos de código fuente que no se necesiten en el 
sistema, entre otras.
6. pciutils: Utilidades para la detección de hardware conectado en los puertos PCI de tu ordenador.
```
**19.1.2. Verificación del enlace simbólico a las fuentes:**

Cuándo termine de compilar esos paquetes, verifica que el enlace simbólico **"linux"** apunta a las fuentes del núcleo:

```
# ls -l /usr/src/linux
```

Si no es así, crealo:
```
ln -s /usr/src/linux /usr/src/linux-$(uname -r)
```
**19.1.3. Compilación e instalación: Módulos e initramfs con Genkernel:**

Y para compilar:

```
# genkernel --lvm --luks all
```

Con estos parámetros se "le dice" a **"genkernel"** que configure y compile el núcleo "él solo" y que además
agregue soporte para **LUKS** y **LVM**.

**Ahora prepara un café y descansa un poco, este proceso durará más o menos dependiendo de la potencia de tu
procesador.**

**Yo tengo un Intel Core i5 2540M con dos (2) núcleos, cuatro (4) hilos y me ha durado unos 40 minutos.**

**Esto es así, debido a que con esta opción genkernel realiza todo el proceso, incluyendo la "creación" del initramfs.
Es decir, no sólo compila el núcleo. Ya veremos que configurando nosotros mismos el núcleo se tarda menos tiempo.**

**19.2. Compilación manual:**

Bien ahora toca explicar cómo hacerlo o más bien cómo yo compilo el núcleo de forma manual, 
pues esto depende de qué quieras compilar.

A continuación habrán opciones y/o módulos que son necesarios para que el sistema arranque y otros que los activo
porque quiero tenerlos. Donde se aplican estos últimos, digo que no es necesario si no quieres tener dicho soporte.

**19.2.1. Verificación del enlace simbólico a las fuentes:**

Primero verifica que el enlace simbólico **"linux"** apunta a las fuentes correspondientes:
```
ls -l /usr/src/linux
```
**19.2.2. Creación del enlace simbólico a las fuentes:**

Si no es así, crealo:
```
ln -s /usr/src/linux /usr/src/linux-$(uname -r)
```

**19.2.3. Entrar en el directorio de las fuentes y menú de configuración:**

Entrar al directorio de las fuentes y entrar al menú de configuración:

```
# cd /usr/src/linux

# make menuconfig
```
**19.2.4. Configuración: Módulos y soporte en el núcleo:**

Una vez aquí dentro vamos a activar algunas opciones que son necesarias para que el núcleo pueda funcionar.

Algunas características necesarias Gentoo las trae activadas para "hacernos la vida más fácil" jeje.

**19.2.4.1. Soporte devtmpfst:**

__Habilitar soporte devtmpfst__
```
Device Drivers --->
  Generic Driver Options --->
    [*] Maintain a devtmpfs filesystem to mount at /dev
    [ ]   Automount devtmpfs at /dev, after the kernel mounted the rootfs
```

**19.2.4.2. Soporte para discos SCSI:**

__Habilitar el soporte de disco SCSI__
```
Device Drivers --->
   SCSI device support  --->
      <*> SCSI disk support
```

**19.2.4.3. Soporte de sistemas de archivos:**

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

**19.2.4.4. Soporte para conexión de red: Cableada e innalámbrica:**

__Controladores para Ethernet y Wi-Fi__

Bien, aquí hay que hacer una pequeña pausa ya que necesitas saber qué controladores usa tu PC para Ethernet y Wi-Fi. Puedes obtener un poco de información con:
```
# lspci | grep Ethernet
# lspci | grep Network
```
Mi ordenador utiliza el controlador: **Intel Wireless WiFi Next Gen AGN - Wireless-N/Advanced-N/Ultimate-N (iwlwifi)** 
para la conexión inalámbrica.

Y el controlador **Intel (82586/82593/82596) devices** para la conexión cableada.

Si no obtienes información suficiente con el comando anterior debería buscar información en 
[DuckDuckGo](https://www.duckduckgo.com/)

La sección para los controladores de red está en:
```
Device Drivers --->
  Network device support --->
```

**19.2.4.5. Soporte SMP:**

__Activar el soporte SMP__
```
Processor type and features  --->
  [*] Symmetric multi-processing support
```

**19.2.4.6. Soporte para USB:**

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

**19.2.4.7. Soporte para características del procesador:**

__Seleccionar los tipos y las características del procesador__
```
Executable file formats / Emulations  --->
   [*] IA32 Emulation
```

**19.2.4.8. Soporte para GPT:**

__Habilitar el soporte para GPT__

```
-*- Enable the block layer --->
   Partition Types --->
      [*] Advanced partition selection
      [*] EFI GUID Partition support
```

**19.2.4.9. Soporte para UEFI:**

Este paso no es necesario si no tienes UEFI

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

**19.2.4.10. Soporte para ALSA:**

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

**19.2.4.11. Soporte para Bluetooth:**

Este paso no es necesario si no tienes **Bluetooth**

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

**19.2.4.12. Soporte para LVM:**

__Habilitar soporte para LVM__
```
Device Drivers --->
 Multiple devices driver support (RAID and LVM) --->
  <*> Device mapper support
    <*> Crypt target support
    <*> Snapshot target
    <*> Mirror target
  <*> Multipath target
    <*> I/O Path Selector based on the number of in-flight I/Os
    <*> I/O Path Selector based on the service time 
```

**19.2.4.13. Soporte para criptografía:**

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

**19.2.4.14. Soporte para OpenVPN:**

Este paso no es necesario si no deseas utilizar [OpenVPN](https://es.wikipedia.org/wiki/OpenVPN)

**Habilitar soporte para TUN/TAP:**
```
Device Drivers  --->
    [*] Network device support  --->
        [*] Network core driver support
        <*>   Universal TUN/TAP device driver support
```

**19.2.4.15. Soporte para Samba:**

Este paso no es necesario si no deseas utilizar el servicio [Samba](https://es.wikipedia.org/wiki/Samba_(software))

**Habilitar soporte para el sistema de archivo Samba:**
```
File Systems --->
    [*] Network File Systems --->
        [*] CIFS support (advanced network filesystem, SMBFS successor)--->
            [*] CIFS Statistics
                [*] Extended Statistics
            [*] CIFS Extended Attributes
                [*] CIFS POSIX Extentions
            [*] SMB2 and SMB3 network file system support
```

**19.2.5. Compilación:**

Bien, una vez hecho todos estos pasos es hora de compilar nuestro núcleo:

Es bueno que sepas cuantos núcleos e hilos tiene tu procesador, para que aproveches esta
y todas las compilaciones que realices en Gentoo.

**19.2.5.1. Verificación de núcleos del procesador:**

Para verificar esto, ejecuta el siguiente comando y busca donde dice **"cpu cores"**
```
# cat /proc/cpuinfo
```

**19.2.5.2. Compilar:**

Ejecuta **make** con el parámetro **-j** y a este "pásale" el número de hilos que tenga tu procesador.

¿Recuerdas que he dicho que mi procesador tiene 4 hilos? Pues bien, así lo haré:

```
# make -j 4

# make modules_install

# make install

# genkernel --lvm --luks --install initramfs
```

**19.2.5.3. Explicación de los comandos/programas utilizados anteriormente:**
```
1. make: Es para compilar el kernel/núcleo.

2. make modules_install: Es para compilar los módulos.

3. make install: Es para instalar el kernel.

4. genkernel --lvm --luks --install initramfs: Es para crear el initramfs con soporte para LVM y LUKS.
```

Bien, ya con eso tendríamos nuestro núcleo listo.

**20. Edición de fstab:**

Este archivo está ubicado en **/etc/fstab** que sirve para montar las 
particiones automáticamente al encender nuestro ordenador.

Si tienes conocimientos, puedes editarlo por tu cuenta, sin embargo, sino, 
puedes dejarlo como lo dejo yo y no tendrás ningún problema.

**(Estoy tomando en cuenta de que haz seguido al pie de la letra la guía o al menos el punto de las particiones)**

En el manual de Gentoo recomiendan utilizar UUID (Identificador único universal) para agregar las particiones al fstab.

Tienes dos opciones:
```
1. Utilizar UUID
2. Utilizar su punto de montaje.
```
**20.1. Verificación de UUID(s) de particiones y/o volúmenes:**

Utiliza el que te sea más cómodo. Si quieres utilizar UUID(s) verifícalos con:
```
# blkid
```
Y lo agregas al **fstab.** Puedes copiarlo moviéndote con el cursor, seleccionándolo y con click derecho, 
se copiará y se pegará en dónde esté el cursor de escritura que de seguro estará en el prompt, 
así que borras lo que se pegó y escribes el comando:

```
# nano -lw /etc/fstab
```

Y así lo vas haciendo hasta que hayas agregado cada UUID de cada partición o volumen correspondiente.

Como ya comenté, también tienes la opción de utilizar su punto de montaje.

En mis instalaciones de Gentoo suelo utilizar UUID(s) pero para no hacer esta guía más larga 
utilizaré puntos de montaje, pero puedes utilizar el método que más te guste.

**Como nota final he de decir que mi fstab está más o menos configurado para un SSD, 
aunque no hay ningún problema si tienes un HDD y lo utilizas igual que yo.**

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

/dev/sda1               /boot   ext2    noatime,nodiratime  0 0
/dev/mapper/gentoo-swap none    swap    sw                  0 0
/dev/mapper/gentoo-root /       ext4    noatime,nodiratime  0 1
/dev/mapper/gentoo-home /home   ext4    noatime,nodiratime  0 2

tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
```
**21. Configurar nombre de host:**

Establecer nombre a nuestro host/ordenador:

Este nombre se lo agregaremos ahora pero puedes cambiarlo en cualquier momento si así lo deseas.
```
# nano -lw /etc/conf.d/hostname
```
Introduce el nombre que desees dentro de las comillas.

**22. Configuración del 
[protocolo DHCP](https://es.wikipedia.org/wiki/Protocolo_de_configuraci%C3%B3n_din%C3%A1mica_de_host):**

Ahora sería bueno que ejecutaras:

```
# ifconfig
```

Para que sepas que nombre es asignado a tu tarjeta de ethernet y así poder configurarla para 
utilizar el [protocolo DHCP](https://es.wikipedia.org/wiki/Protocolo_de_configuraci%C3%B3n_din%C3%A1mica_de_host)
y poder acceder a internet (de esta forma), al menos hasta que instales:

[NetworkManager](https://es.wikipedia.org/wiki/NetworkManager) (que es el gestor de red que utilizo) o 
cualquier otro gestor de red.

En mi caso es asignado el nombre **eno1**, así que quedaría más o menos así:

```
# nano -lw /etc/conf.d/net
```

Dentro del archivo pones: 
```
config_eno1="dhcp"
```
Evidentemente **eno1** lo cambias por el nombre que se le asignó a tu tarjeta de ethernet.

**23. Establecimiento de la contraseña del usuario root:**

Bien, ya casi hemos terminado, pero no te desconcentres porque ahora vamos por otro de los 
puntos más importantes y es el de asignar una contraseña al usuario root.

```
# passwd
```

Ponle la que desees, en el proceso no verás nada, pero de seguro eso ya lo sabes jeje.

**24. Configurar la distribución del teclado en la(s) tty(s):**

Ahora estableceremos la distribución que tendrá nuestro teclado en la(s) tty(s).

Aquí tienes que tener cuidado, puesto que una mala configuración puede ocasionar resultados 
extraños cuando estemos teclando.

```
# nano -lw /etc/conf.d/keymaps
```

Yo utilizo como distribución de teclado: **Inglés (Estados Unidos Internacional)** y el que trae por 
defecto sólo es **Inglés** así que, en mi caso, quedaría más o menos de la siguiente forma:

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

Aquí el punto importante es arriba donde dice: **keymap="us"**

**25. Configuración del reloj:**

Ahora vamos a configurar el reloj del sistema. Por defecto viene en *__UTC__* 
pero lo puedes cambiar por el que quieras, yo esto lo dejo así, por defecto.

Para cambiarlo, lo haces editando el siguiente archivo:

```
# nano -lw /etc/conf.d/hwclock
```

**26. Instalación y configuración de [GRUB](https://es.wikipedia.org/wiki/GNU_GRUB):**

[El cargador de arranque GRUB](https://es.wikipedia.org/wiki/GNU_GRUB) tiene su archivo de configuración en la ruta:

**/etc/default/grub,** aquí vamos a establecer una serie de valores necesarios para nuestros propósitos:

**Configurarlo para que soporte LUKS y LVM.**

```
# nano -lw /etc/default/grub
```

Ubicas la línea que tiene: **GRUB_CMDLINE_LINUX** y pones lo siguiente antes de ella:

**Nota: Aquí también puedes utilizar UUID(s) en lugar de puntos de montajes. Yo así es que lo hago, 
pero como comenté en el paso de edición del "fstab", utilizaré puntos de montajes en esta guía para 
no hacerla muy larga.**

**Pero si quieres utilizar UUID(s), sólo tendrás que utilizar la herramienta: "blkid", copiar el de 
la partición cifrada, agregarlo en la variable: "crypt_root" y luego copiar el UUID del volumen donde 
está root y lo agregas a "real_root".**
```
GRUB_PRELOAD_MODULES=lvm
GRUB_ENABLE_CRYPTODISK=y
GRUB_DEVICE=/dev/ram0
GRUB_CMDLINE_LINUX="crypt_root=/dev/sda2 real_root=/dev/mapper/gentoo-root rootfstype=ext4 dolvm"
```

**26.1. Instalación:**

Instalamos GRUB con los siguientes módulos: **linux, crypto, search_fs_uuid, luks y lvm**
```
# grub-install --modules="linux crypto search_fs_uuid luks lvm" --recheck /dev/sda
```

**26.2. Generar el archivo de configuración:**

Y la configuración
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

**27. Activar el servicio de lvm para que se ejecute al inicio:**
```
# rc-update add lvm boot
```

**28. Salir de "la jaula chroot" y desmontar las particiones:**
```
# exit

# cd ../../

# umount -l /mnt/gentoo/dev{/pts,/shm,}

# umount -R /mnt/gentoo
```

**29. Reiniciar:**
```
# reboot
```

¡Felicitaciones, ya tienes Gentoo instalado! Ahora queda hacer alguna que otra modificación 
para empezar a compilar e instalar los paquetes que necesitemos:

**30. "Creación" de un usuario con privilegios limitados:**

"Crear" un usuario a parte de root, puesto que como seguro sabrás no es recomendable 
utilizar siempre el usuario root. Es mejor tener **un usuario con privilegios limitados,** 
que pueda invocar permisos de root **sólo cuándo sea necesario.**

Iniciamos sesión con el usuario root y la contraseña que especificamos en la instalación.
```
# useradd -m -G users,wheel,audio,video,usb,portage,cdrom -s /bin/bash <<nombre_de_usuario>>
# passwd <<nombre_de_usuario>>
```
**31. Instalación de "sudo" para dotarlo de la posibilidad de ejecutar procesos con permisos de root:**

Ahora instalamos **sudo** para poder utilizar permisos de root con nuestro usuario:

```
# emerge -a sudo
```

**31.1. Editar el archivo "sudoers" para completar este paso:**

Editamos el fichero: **/etc/sudoers** y descomentamos la línea: # %wheel ALL=(ALL) ALL
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```
**32. Cerrar sesión:**

Cerramos sesión de la cuenta de root e iniciamos sesión con el usuario que acabamos de crear:
```
# exit
```
**33. Instalar el servidor xorg:**

Y el último paso de esta guía es instalar el xorg para poder instalar el DE o WM que deseemos:
```
# sudo emerge -a xorg-server
```
Ahora instalas los programas que desees/necesites, ejemplo: VLC, Firefox, etc.

Podrás encontrar más información en la [Wiki de Gentoo](https://wiki.gentoo.org/wiki/Main_Page) y buscando en la WWW.

Esto sólo fue una pequeña introducción, tienes todo un universo que descubrir con Gentoo.

Así que con esto me despido. ¡Disfuta del universo de Gentoo!

Te comparto estas dos frases que podrían ayudarte xD

**La suerte no existe sólo las Matemáticas y la ciencia**

**No necesitas suerte, sólo determinación y esfuerzo.**

**34. Enlaces que te podrían interesar:**

[Artículo en Wikipedia sobre SHA-512](https://es.wikipedia.org/wiki/SHA512)

[Manual en español de Gentoo](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation/es)

[Artículo en Wikipedia sobre Samba](https://es.wikipedia.org/wiki/Samba_(software))

[Artículo en Wikipedia sobre OpenVPN](https://es.wikipedia.org/wiki/OpenVPN)

[Artículo en Wikipedia sobre el protocolo DHCP](https://es.wikipedia.org/wiki/Protocolo_de_configuraci%C3%B3n_din%C3%A1mica_de_host)

[Artículo en Wikipedia sobre el gestor de red NetworkManager](https://es.wikipedia.org/wiki/NetworkManager)

[Artículo en Wikipedia sobre imágenes ISO](https://es.wikipedia.org/wiki/Imagen_ISO)

[Artículo en Wikipedia sobre GNU GRUB](https://es.wikipedia.org/wiki/GNU_GRUB)
