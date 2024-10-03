# Корневая файловая система

Сборка ядра и dtb файла происходит в [задании 19](https://github.com/EltexEmbeddedC/cross-compilation). Для получения полной прошивки нужно также собрать корневую файловую систему.

## Сборка простой корневой файловой системы

- Напишем простейший init процесс, который выводит приветственное сообщение и через время завершается. Пример кода можно посмотреть в документации ядра Linux ([ссылка](https://github.com/torvalds/linux/blob/master/Documentation/filesystems/ramfs-rootfs-initramfs.rst#contents-of-initramfs));

  ```c
  #include <stdio.h>
  #include <unistd.h>
  
  int main(int argc, char *argv[])
  {
    printf("Hello world!\n");
    sleep(20);
  }
  ```
- `arm-linux-gnueabihf-gcc -static init.c -o init` - соберем init процесс под ARM архитектуру, статически прилинковав libc;

> [!NOTE]
> Для проверки свойств файла можно выполнить `file ./init`.
- `echo init | cpio -o -H newc | gzip > initramfs.cpio.gz` - упакуем файл в архив;
- Положим этот архив рядом с файлом ядра и dtb файлом. Таким образом, получаем простейшую прошивку;
- `QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -kernel zImage -initrd initramfs.cpio.gz -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0" -nographic` - запустим эмулятор с новой прошивкой:

  ![image](https://github.com/user-attachments/assets/8ba82ae6-9cf5-4b15-88e2-af12f6a6f240)

  После старта увидим приветственное сообщение, выводимое init процессом:
  
  ![image](https://github.com/user-attachments/assets/0f5c7a83-8d9a-4a84-a628-91a28fa76865)

  По истечении таймера процесс init завершится и ядро упадет с ошибкой `Kernel panic`:

  ![image](https://github.com/user-attachments/assets/62a01422-beef-48dc-84c2-92d0298ff648)

## Сборка корневой файловой системы на основе BusyBox и включение в неё OpenSSH

### Сборка OpenSSH

- Склонируем репозитории [OpenSSH](https://github.com/openssh/openssh-portable), а также [zlib](https://github.com/madler/zlib) и [OpenSSL](https://github.com/openssl/openssl), необходимые для сборки;
- Соберем zlib:
  - `CC=arm-linux-gnueabihf-gcc ./configure --static --prefix=$PWD/_install` - выполним конфигурацию Makefile:

    ![image](https://github.com/user-attachments/assets/3ed02d45-6931-4a5d-b8e9-c9b4ace9712f)
  - `make -j <число ядер>` - выполним сборку:

    ![image](https://github.com/user-attachments/assets/831ef48b-b1e1-40d9-aafa-baf96d6c15e5)

  - `make install` - выполним установку:

    ![image](https://github.com/user-attachments/assets/0aa1e6c0-4983-4bfc-a03a-68000ed0b932)

- Соберем OpenSSL:
  - `./Configure linux-armv4 shared --cross-compile-prefix=arm-linux-gnueabihf- --prefix=$PWD/_include` - выполним конфигурацию под целевую платформу:

    ![image](https://github.com/user-attachments/assets/b9b50acf-1480-4c39-98f9-867a5c85bdd9)
  - `make -j <число ядер>` - выполним сборку:

    ![image](https://github.com/user-attachments/assets/ab8e3214-78b4-4797-b912-fe6112d126b7)

  - `make install` - выполним установку:
 
    ![image](https://github.com/user-attachments/assets/c5e4ea32-6dae-4de7-865c-3f0dd6682eaa)

- Выполним сборку OpenSSH:

  - `./configure --prefix=$PWD/_install --disable-strip --with-zlib=$PWD/../zlib/_install --with-ssl-dir=$PWD/../openssl/_include --host=arm-linux-gnueabihf` - выполним конфигурацию под целевую платформу:

    ![image](https://github.com/user-attachments/assets/e4ccf106-5ce8-4b45-8bf9-10c7344183aa)
  - `make` - выполним сборку:

    ![image](https://github.com/user-attachments/assets/472a4c20-2bf0-4650-8800-2dcfcbd7f383)
  - `make install` - выполним установку:
 
    ![image](https://github.com/user-attachments/assets/5554b965-dc09-414c-89bf-5a97c69577b3)

  - Увидим, что файлы собрались:

    ![image](https://github.com/user-attachments/assets/02f65c86-4be6-4aa9-b0b4-c74856a69b1e)

  - Все получившиеся файлы будут помещены в соответствующие директории, которые получатся на следующем этапе;
 
  - Запуск `ssh` также будет проведен ниже.
  
### Сборка и установка BusyBox

- Перейдя в директорию с BusyBox выполним сборку под ARM: `ARCH=arm make defconfig`;

  ![image](https://github.com/user-attachments/assets/40ce7d73-9356-4bee-9ffb-2b00e828768e)

- `ARCH=arm make menuconfig` - отредактируем дефолтную конфигурацию;

  Во вкладке Settings:
  - `Build static binary (no shared libs)` - включить;
  - `Cross compiler prefix` указать `-arm-linux-gnueabihf-`.

  ![image](https://github.com/user-attachments/assets/cb3a2b75-368f-4cd6-b2ef-a73ecbd14335)

- `make -j <число ядер>` - выполним сборку;

  ![image](https://github.com/user-attachments/assets/dc61b695-f836-462e-9967-185860e4bf28)

- `make install` - выполним установку;


  ![image](https://github.com/user-attachments/assets/3c70975d-b61f-4e14-bff2-606cf310efff)

- `find . | cpio -o -H newc | gzip > initramfs.cpio.gz` - выполним сборку архива, перейдя в папку `/_install` из корневой директории BusyBox;
- Положим этот архив рядом с файлом ядра и dtb файлом. Таким образом, получаем прошивку;
- `QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -kernel zImage -initrd initramfs.cpio.gz -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0 rdinit=/bin/ash" -nographic` - запустим эмулятор и увидим, что теперь есть доступ к консоли. Проверим работу команд `echo` и `ssh`:

  ![image](https://github.com/user-attachments/assets/1ac6c894-391d-459e-8420-9550a0dbf5d2)

Таким образом, обе прошивки были успешно собраны и протестированы.

## Доп. задание: сборка и запуск U-Boot под ARM

1. Склонируем репозиторий по ссылке ([ссылка](https://github.com/u-boot/u-boot));
2. `ARCH=arm make qemu_arm_defconfig` - создадим конфигурацию;

   ![image](https://github.com/user-attachments/assets/992d5d4c-3914-4f77-b39e-316dacde37a5)
3. `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j <число ядер>` - выполним сборку;

   ![image](https://github.com/user-attachments/assets/271c0be4-ac8b-4c38-b73c-a66b6f9f9cda)
4. `QEMU_AUDIO_DRU=none qemu-system-arm -M virt -bios u-boot.bin -nographic` - запустим U-Boot. Так как рядом с загрузчиком ничего нет, то запустится только его терминал, где можно, к примеру, вывести переменные среды окружения (`env print`) или список доступных команд (`help`)

   ![image](https://github.com/user-attachments/assets/b735c44a-0933-49ce-b66e-44bb6e8a67f8)

<details>

<summary>=> env print</summary>

```
arch=arm
baudrate=115200
board=qemu-arm
board_name=qemu-arm
boot_targets=qfw usb scsi virtio nvme dhcp
bootcmd=bootflow scan -lb
bootdelay=2
cpu=armv7
ethaddr=52:54:00:12:34:56
fdt_addr=0x40000000
fdt_high=0xffffffff
fdtcontroladdr=465ddea0
initrd_high=0xffffffff
kernel_addr_r=0x40400000
loadaddr=0x40200000
preboot=usb start
pxefile_addr_r=0x40300000
ramdisk_addr_r=0x44000000
scriptaddr=0x40200000
stderr=serial,vidconsole
stdin=serial,usbkbd
stdout=serial,vidconsole
usb_ignorelist=0x1050:*,
vendor=emulation
```

</details>

<details>

<summary>=> help</summary>

```
?         - alias for 'help'
base      - print or set address offset
bdinfo    - print Board Info structure
blkcache  - block cache diagnostics and control
boot      - boot default, i.e., run 'bootcmd'
bootd     - boot default, i.e., run 'bootcmd'
bootdev   - Boot devices
bootefi   - Boots an EFI payload from memory
bootelf   - Boot from an ELF image in memory
bootflow  - Boot flows
bootm     - boot application image from memory
bootmeth  - Boot methods
bootp     - boot image via network using BOOTP/TFTP protocol
bootvx    - Boot vxWorks from an ELF image
bootz     - boot Linux zImage image from memory
chpart    - change active partition of a MTD device
cls       - clear screen
cmp       - memory compare
coninfo   - print console devices and information
cp        - memory copy
crc32     - checksum calculation
date      - get/set/reset date & time
dfu       - Device Firmware Upgrade
dhcp      - boot image via network using DHCP/TFTP protocol
dm        - Driver model low level access
echo      - echo args to console
editenv   - edit environment variable
eficonfig - provide menu-driven UEFI variable maintenance interface
env       - environment handling commands
erase     - erase FLASH memory
exit      - exit script
ext2load  - load binary file from a Ext2 filesystem
ext2ls    - list files in a directory (default /)
ext4load  - load binary file from a Ext4 filesystem
ext4ls    - list files in a directory (default /)
ext4size  - determine a file's size
false     - do nothing, unsuccessfully
fatinfo   - print information about filesystem
fatload   - load binary file from a dos filesystem
fatls     - list files in a directory (default /)
fatmkdir  - create a directory
fatrm     - delete a file
fatsize   - determine a file's size
fatwrite  - write file into a dos filesystem
fdt       - flattened device tree utility commands
flinfo    - print FLASH memory information
fstype    - Look up a filesystem type
fstypes   - List supported filesystem types
go        - start application at address 'addr'
help      - print command description/usage
iminfo    - print header information for application image
imxtract  - extract a part of a multi-image
itest     - return true/false on integer compare
lcdputs   - print string on video framebuffer
ln        - Create a symbolic link
load      - load binary file from a filesystem
loadb     - load binary file over serial line (kermit mode)
loads     - load S-Record file over serial line
loadx     - load binary file over serial line (xmodem mode)
loady     - load binary file over serial line (ymodem mode)
loop      - infinite loop on address range
ls        - list files in a directory (default /)
md        - memory display
mii       - MII utility commands
mm        - memory modify (auto-incrementing address)
mtd       - MTD utils
mtdparts  - define flash/nand partitions
mw        - memory write (fill)
net       - NET sub-system
nm        - memory modify (constant address)
nvme      - NVM Express sub-system
panic     - Panic with optional message
part      - disk partition related commands
pci       - list and access PCI Configuration Space
ping      - send ICMP ECHO_REQUEST to network host
poweroff  - Perform POWEROFF of the device
printenv  - print environment variables
protect   - enable or disable FLASH write protection
pxe       - get and boot from pxe files
qfw       - QEMU firmware interface
random    - fill memory with random pattern
reset     - Perform RESET of the CPU
run       - run commands in an environment variable
save      - save file to a filesystem
saveenv   - save environment variables to persistent storage
scsi      - SCSI sub-system
scsiboot  - boot from SCSI device
setcurs   - set cursor position within screen
setenv    - set environment variables
setexpr   - set environment variable as the result of eval expression
showvar   - print local hushshell variables
size      - determine a file's size
sleep     - delay execution for some time
source    - run script from memory
test      - minimal test like /bin/sh
tftpboot  - load file via network using TFTP protocol
tpm       - Issue a TPMv1.x command
tpm2      - Issue a TPMv2.x command
true      - do nothing, successfully
usb       - USB sub-system
usbboot   - boot from USB device
vbe       - Verified Boot for Embedded
version   - print monitor, compiler and linker version
virtio    - virtio block devices sub-system
```

</details>
