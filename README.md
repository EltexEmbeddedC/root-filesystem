# Корневая файловая система

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
- Положим этот архив рядом с файлом ядра и dtb файлом. Таким оюразом, получаем простейшую прошивку;
- `QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -kernel zImage -initrd initramfs.cpio.gz -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0" -nographic` - запустим эмулятор с новой прошивкой:

  ![image](https://github.com/user-attachments/assets/8ba82ae6-9cf5-4b15-88e2-af12f6a6f240)

  После старта увидим приветственное сообщение, выводимое init процессом:
  
  ![image](https://github.com/user-attachments/assets/0f5c7a83-8d9a-4a84-a628-91a28fa76865)

  По истечении таймера процесс init завершится и ядро упадет с ошибкой `Kernel panic`:

  ![image](https://github.com/user-attachments/assets/62a01422-beef-48dc-84c2-92d0298ff648)

## Сборка корневой файловой системы на основе BusyBox

- Перейдя в директорию с BusyBox выполним сборку под ARM `ARCH=arm make defconfig`;

  ![image](https://github.com/user-attachments/assets/40ce7d73-9356-4bee-9ffb-2b00e828768e)

- `ARCH=arm make menuconfig` - отредактируем дефолтную конфигураию:

  Во вкладке Settings:
  - `Build static binary (no shared libs)` - включить;
  - `Cross compiler prefix` указать `-arm-linux-gnueabihf-`.

  ![image](https://github.com/user-attachments/assets/cb3a2b75-368f-4cd6-b2ef-a73ecbd14335)

- `make -j <число ядер>` - выполним сборку:

  ![image](https://github.com/user-attachments/assets/dc61b695-f836-462e-9967-185860e4bf28)

- `make install` - выполним установку и увидим сообщение об успешной сборке:


  ![image](https://github.com/user-attachments/assets/3c70975d-b61f-4e14-bff2-606cf310efff)

- `find . | cpio -o -H newc | gzip > initramfs.cpio.gz` - выполним сборку архива, перейдя в папку `/_install` из корневой директории BusyBox;
- Положим этот архив рядом с файлом ядра и dtb файлом. Таким оюразом, получаем прошивку;
- `QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -kernel zImage -initrd initramfs.cpio.gz -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0 rdinit=/bin/ash" -nographic` - запустим эмулятор и увидим, что теперь есть доступ к консоли:

  ![image](https://github.com/user-attachments/assets/a5d33a8c-edab-4e0e-9763-4e06b42d4bbb)

Таким образом, обе прошивки были успешно собраны и протестированы.
