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
