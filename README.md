# Instalación de Arch Linux con modelo de interfaz UEFI

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Introducción

#### ¿Qué es Arch Linux?

Arch Linux es una distribución GNU/Linux de propósito general, desarrollada independientemente para x86-64, que se esfuerza por proporcionar las últimas versiones estables de la mayoría del software, siguiendo un modelo de lanzamiento continuo (`rolling-release`). La instalación por defecto deja un sistema de base mínima, que el usuario configurará posteriormente agregando lo que necesite.

Los desarrolladores definen el proyecto bajo el acrónimo `KISS` (`Keep It Simple, Stupid`; en español Mantenlo simple, estúpido). Arch Linux se enfoca en 5 principios generales: simplicidad, modernidad, pragmatismo, centrado en el usuario y versatilidad; siendo el primero, su principal objetivo. Los desarrolladores de Arch son voluntarios no remunerados, a tiempo parcial, y no hay perspectivas de monetizar Arch Linux, por lo que seguirá siendo libre en todos los sentidos de la palabra.

#### ¿Qué es UEFI?

La `Unified Extensible Firmware Interface` (`UEFI` o `EFI` para abreviar) es un nuevo modelo de interfaz para interactuar entre los sistemas operativos y el firmware. Proporciona un entorno estándar para iniciar un sistema operativo y ejecutar aplicaciones previas al inicio. Es un método distito del comunmente usado "código de arranque `MBR`" seguido por los sistemas `BIOS` estándar presentado en las computadoras personales `IBM PC` como `IBM PC ROM BIOS`.

## Consideraciones previas

A los efectos prácticos de esta guía se instalará la distribución de `GNU/Linux` `Arch Linux` en un ordenador que dispone del modelo de interfaz `UEFI` activo a nivel de `BIOS`, y con las siguientes características de `hardware`.

- **CPU**: Intel Core i5 3.20GHz
- **RAM**: 8 GiB
- **SSD**: 120 GiB
- **HDD**: 1 TiB
- **Adaptador de red**: Gigabyte Onboard Ethernet

Se dispone además de un direccionamiento `IPv4` con acceso a `Internet`.

El sistema operativo instalado, será utilizado con el propósito de explotar un sistema de escritorio o estación de trabajo para usuarios finales.

## Instalación

Arch Linux se instala ejecutando el entorno `live` de la imagen `ISO` desde un dispositivo óptico, memoria `USB` o una red con `PXE`. La imagen `ISO` puede descargarse desde [Arch Linux - Downloads](https://www.archlinux.org/download/).

#### Preinstalación

1. Definir la distribución de teclado.

    ```bash
    loadkeys us-acentos
    ```

    > **NOTA**: Por defecto, la distribución del teclado de la consola es la de EE.UU.; las distribuciones de teclado disponibles se pueden listar ejecutando `ls /usr/share/kbd/keymaps/**/*.map.gz`.

2. Verificar la modalidad de arranque.

    Si el modo `UEFI` está activado, la imagen `ISO` arrancará mdeiante `systemd-boot`. Para comprobarlo, se debe listar el contenido del directorio `efivars`, ejecutando `ls /sys/firmware/efi/efivars`. Si no existe el directorio, el sistema ha iniciado en modo `BIOS` o `CSM` (`Compatibility Support Module`).

3. Configurar conexión de red para acceder a los repositorios de paquetes.

    > **NOTA**: La imagen `ISO` activa en el arranque `dhcpcd` para todos los dispositivos de red conocidos, tanto cableados como WiFi.

    Si no se encuentra en una red donde exista un servidor `DHCP`, se deben definir los parámetros de red de forma manual.

    a. Obtener nombre del dispositivo de red.

    ```bash
    ifconfig -a
    ```

    b. Definir dirección `IP` estática.

    ```bash
    ip addr add 200.55.143.194/29 dev enp3s0
    ip link set enp3s0 up
    ```
    o
    ```bash
    ifconfig enp3s0 up 200.55.143.194 netmask 255.255.255.248
    ```

    c. Definir puerta de enlace (`gateway`) por defecto.

    ```bash
    route add 0.0.0.0/0 via 200.55.143.193 dev enp3s0
    ```

    d. Establecer servidores de nombres de dominio.

    ```bash
    nano /etc/resolv.config

    nameserver 8.8.8.8
    nameserver 8.8.4.4
    ```

    e. Verificar que la interfaz de red está activada.

    ```bash
    ifconfig
    ip link show
    ip addr show
    ip route show
    ```

    > **NOTA**: Opcinalmente se puede utilizar el comando `ping` para verificar la conexión.

4. Actualizar la hora del sistema.

    Utilizar `timedatectl set-ntp true`, para asegurar que el reloj del sistema esté en hora.

    > **NOTA**: Para comprobar el estado del servicio, ejecutar `timedatectl status`.

5. Particionamiento de discos duros.

    Cuando el sistema reconoce los discos existentes, estos son asignados a dispositivos de bloque como por ejemplo `/dev/sda`. Para identificar los discos, se debe ejecutar `lsblk -f` o `fdisk -l`.

    Se utilizará el disco `SSD` de `120 GiB` nombrado `/dev/sda`, para que contenga la partición de arranque `EFI` con un tamaño de `512MiB` y el resto para la partición raíz `/`. Mientras que en el disco `HDD` de `1 TiB`, identificado como `/dev/sdb` se definirán las particiones `/var` de `12 GiB`, `/tmp` de `5 GiB`, espacio de intercambio `SWAP` de `4 GiB` y `/home`, esta última con el resto del espacio disponible. Al estarse utilizando `UEFI` el esquema de tabla de particiones a utilizar será `GPT`.

    a. Definir esquema de tabla de particiones en disco `/dev/sda`.

    ```bash
    fdisk /dev/sda
    ```

    Teclear la opción `g` para crear una nueva tabla de particiones de tipo `GPT`. Teclear la opción `n` para crear las nuevas particiones. La partición de arranque debe ser de tipo `EFI System` y la raíz, `Linux filesystem`. Al finalizar teclear la opción `w` para escribir los cambios.

    b. Definir esquema de tabla de particiones en disco `/dev/sdb`.

    ```bash
    fdisk /dev/sdb
    ```

    Teclear la opción `g` para crear una nueva table de particiones de tipo `GPT`. Teclear la opción `n` para crear las nuevas particiones. Las particiones `/var`, `/tmp` y `/home` deben ser de tipo `Linux filesystem`, y el área de intercambio, `Linux swap`.  Al finalizar teclear la opción `w` para escribir los cambios.

    > **NOTA**: También se pueden usar las herramientas `gdisk`, `sfdisk` y `parted` para discos particionados con `GPT`.

    c. Formatear las particiones.

    ```bash
    mkfs.fat -F32 -n UEFI /dev/sda1
    mkfs.ext4 -L / /dev/sda2
    mkfs.ext4 -L /var /dev/sdb1
    mkfs.ext4 -L /tmp /dev/sdb2
    mkswap -L SWAP /dev/sdb3
    mkfs.ext4 -L /home /dev/sdb4
    ```

    d. Montar las particiones en el sistema de archivos.

    ```bash
    mount /dev/sda2 /mnt
    mkdir /mnt/{boot,var,tmp,home}
    mount /dev/sda1 /mnt/boot
    mount /dev/sdb1 /mnt/var
    mount /dev/sdb2 /mnt/tmp
    swapon /dev/sdb3
    mount /dev/sdb4 /mnt/home
    ```

#### Instalación del sistema

1. Definir repositorios de paquetes.

    Se deben editar los ficheros `/etc/pacman.d/mirrorlist` y `/etc/pacman.conf`, el primero; con el objetivo de seleccionar la dirección `URL` de los servidores de réplicas en `Internet` más cercano a su ubicación, donde se encuentren los repositorios de paquetes; y el segundo, para habilitar los depósitos o repositorios de paquetes (`core`, `extras`, `community` y `multilib`).

2. Instalar sistema base mínimo.

    ```bash
    pacstrap -i /mnt base base-devel linux-firmware
    ```

    El paquete `base` no incluye todas las herramientas presentes en el medio de instalación `live`, por lo que puede ser necesario instalar otros paquetes para que un sistema base mínimo sea completamente funcional. Se recomienda instalar herramientas necesarias para la conexión a redes (`net-tools`, `dnsutils` e `iputils`), un editor de texto (`nano`, `vi` o `vim`) y paquetes para acceder a la documentación en las páginas de manual o información (`man-db`, `man-pages` y `texinfo`).

    Para instalar otros paquetes o grupos de paquetes se debe añadir sus nombres al comando `pacstrap` o utilizar `pacman`, durante el entorno de jaula `chroot`.

    > **NOTA**: Se puede sustituir `linux` por el paquete de `kernel` de su elección (`linux-hardened`, `linux-lts` o `linux-zen`). Ver [Kernel - ArchWiki](https://wiki.archlinux.org/index.php/Kernel) para mayor información.

#### Configuración básica del sistema

1. Generar el fichero `/etc/fstab`.

    El archivo `/etc/fstab` es usado para definir cómo las particiones, distintos dispositivos de bloques o sistemas de archivos remotos deben ser montados e integrados en el sistema.

    ```bash
    genfstab -U -p /mnt >> /mnt/etc/fstab
    ```

    > **NOTA**: Se debe comprobar el archivo resultante en `/mnt/etc/fstab`, y modificarlo añadiendo las opciones `noatime` y `discard` en el caso de la partición raíz y solamente `noatime` para la particón de arranque `EFI`; estas características son necesarias para prolongar la vida últil del disco `SSD`.

2. Activar entorno de jaula `chroot`.

    `Chroot` es el proceso por el que se cambia el directorio `root` del disco de instalación `live` a otro directorio `root`. Al cambiar a otro directorio `root` no se puede acceder a los archivos y comandos fuera de ese directorio. Este directorio se llama jaula `chroot`. El cambio de `root` se hace comúnmente para el mantenimiento del sistema, como puede ser volver a instalar el gestor de arranque o restablecer una contraseña olvidada.

    ```bash
    arch-chroot /mnt
    ```

3. Definir idioma del sistema.

    a. Editar el fichero `/etc/locale.conf` descomentando la definición de idioma que se requiera (por ejemplo, para Español México sería `es_MX.UTF-8 UTF-8`); luego de modificado, se debe generar con el comando `locale-gen`.

    ```bash
    nano /etc/locale.gen
    locale-gen
    ```

    > **NOTA**: Es recomendable también activar `en_US.UTF-8 UTF-8`.

    b. Crear el archivo `/etc/locale.conf`, y definir la variable `LANG` según lo establecido en el paso anterior, por ejemplo:

    ```bash
    echo "LANG=es_MX.UTF-8" > nano /etc/locale.conf
    export LANG=es_MX.UTF-8
    ```

    c. Definir la distribución de teclado en consola para que permanezca en cada reinicio.

    ```bash
    echo "KEYMAP=us-acentos" > nano /etc/vconsole.conf
    ```
4. Definir la zona horaria.

    ```bash
    ln -sf /usr/share/zoneinfo/America/Havana /etc/localtime
    ```

5. Configurar la red.

    a. Crear el archivo `/etc/hostname`.

    ```bash
    echo "archlinux" > /etc/hostname
    ```

    b. Editar fichero `/etc/hosts`.

    ```bash
    nano /etc/hosts

    127.0.0.1   localhost.localdomain   localhost
    ::1         localhost.localdomain   localhost
    127.0.1.1   archlinux.localdomain   archlinux
    ```

    > **NOTA**: Si el sistema tendrá una dirección `IP` permanente, se debe usar dicha dirección, en lugar de `127.0.1.1`.

6. Generar la imagen `initramfs`.

    ```bash
    mkinitcpio -P
    ```

    > **NOTA**: Normalmente no es necesario crear una imagen `initramfs` nueva, dado que `mkinitcpio` se ejecuta durante la instalación del paquete `kernel` con `pacstrap`.

7. Instalar y configurar el gestor de arranque.

    #### `Systemd-boot`

    a. Instalar paquetes necesarios.

    ```bash
    pacman -S systemd-boot intel-ucode
    ```

    > **NOTA**: En sistemas con procesadores `AMD`, se debe instalar el paquete `amd-ucode` en lugar de `intel-ucode`.

    b. Instalar gestor de arranque.

    ```bash
    bootctl --path=/boot install
    ```

    c. Configurar gestor de arranque.

    ```bash
    nano /boot/loader/loader.conf

    default arch
    timeout 4
    console-mode keep
    editor no
    ```

    ```bash
    nano /boot/loader/entries/arch.conf

    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /intel-ucode.img
    initrd  /initramfs-linux.img
    options root=LABEL=/ rw
    ```

    > **NOTA**: En sistemas con procesadores `AMD`, definir `initrd  /amd-ucode.img` en lugar de `initrd  /intel-ucode.img`.

    #### `GRUB`

    a. Instalar paquetes necesarios.

    ```bash
    pacman -S grub efibootmgr
    ```

    b. Instalar gestor de arranque.

    ```bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    ```

    c. Configurar gestor de arranque.

    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

8. Establecer contraseña del superusuario `root`.

    ```bash
    passwd
    ```

9. Abandonar entorno `chroot`, desmontar las particiones y reiniciar el sistema.

    ```bash
    exit
    umount -R /mnt
    reboot
    ```

    > **NOTA**: Se debe retirar el medio de instalación e iniciar sesión en el sistema con la cuenta del superusuario `root`.

## Posinstalación

1. Crear cuenta de usuario.

    ```bash
    useradd -m -G users,wheel,sudo -s /bin/bash -c "Descripción de usuario" <nombre_usuario>
    passwd <nombre_usuario>
    ```

## Referencias

* [ArchWiki](https://wiki.archlinux.org/)
* [UEFI FAQs | Unified Extensible Firmware Interface Forum](https://uefi.org/faq)
* [BIOS](https://es.wikipedia.org/wiki/BIOS)
* [Unified Extensible Firmware Interface](https://es.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
