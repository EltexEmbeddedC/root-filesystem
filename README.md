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
- Положим этот архив рядом с файлом ядра и dtb файлов. Таким оюразом, получаем простейшую прошивку;
- `QEMU_AUDIO_DRV=none qemu-system-arm -M vexpress-a9 -kernel zImage -initrd initramfs.cpio.gz -dtb vexpress-v2p-ca9.dtb -append "console=ttyAMA0" -nographic` - запустим эмулятор с новой прошивкой:

  ![image](https://github.com/user-attachments/assets/8ba82ae6-9cf5-4b15-88e2-af12f6a6f240)

  После старта увидим приветственное сообщение, выводимое init процессом:
  
  ![image](https://github.com/user-attachments/assets/0f5c7a83-8d9a-4a84-a628-91a28fa76865)

  По истечении таймера процесс init завершится и ядро упадет с ошибкой `Kernel panic`:

  ![image](https://github.com/user-attachments/assets/62a01422-beef-48dc-84c2-92d0298ff648)
