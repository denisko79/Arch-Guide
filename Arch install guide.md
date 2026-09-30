<img src="https://upload.wikimedia.org/wikipedia/commons/3/35/Tux.svg" alt="Linux" width="24" />
**Вот исчерпывающий гайд по установке Arch Linux с учётом ваших требований:**

- Файловая система: **Btrfs** (включая `/boot`) + subvolumes  
- Сетевые возможности: **LAN, Wi-Fi, Bluetooth**  
- **ZRAM** вместо swap-раздела  
- Аудио/видео: **PipeWire**  
- Загрузчик: **GRUB**  
- Управление снапшотами: **Snapper**  
- Только консоль и X11 — без DE/WM

---

## 🔧 0. Подготовка

### Загрузка с установочного носителя
Скачайте официальный ISO Arch Linux и загрузитесь с него.

```bash
# Проверка UEFI
ls /sys/firmware/efi/efivars
# Проверьте подключение к интернету
ping -c3 ya.ru
# Если Wi-Fi
iwctl
> device list
> station wlan0 scan
> station wlan0 get-networks
> station wlan0 connect "ИМЯ_WIFI"
> exit
# Обновите системные часы
timedatectl set-ntp true
timedatectl status
```

---

## 💾 1. Разметка диска и Btrfs

> Предположим, что диск — `/dev/sda`. Замените на свой (`nvme0n1`, `sdb` и т.д.)

### Создание таблицы разделов (UEFI)

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
fdisk /dev/sda
```

- Создайте GPT-таблицу (`g`)
- Раздел 1: **EFI System Partition (ESP)** — 1 ГБ, тип `1` (EFI)
- Раздел 2: **Linux filesystem** — всё оставшееся место, тип `20` (Linux filesystem)

Результат:
- `/dev/sda1` → ESP (FAT32)
- `/dev/sda2` → Btrfs root

### Форматирование

```bash
mkfs.fat -F32 -n EFI /dev/sda1
mkfs.btrfs -f -L archroot /dev/sda2
```

### Монтирование и создание subvolumes

```bash
mount /dev/sda2 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache

btrfs subvolume list /mnt
umount /mnt
```

### Монтируем с нужными опциями

```bash
mount -o noatime,compress=zstd,space_cache=v2,subvol=@ /dev/sda2 /mnt

mkdir -p /mnt/{boot,home,.snapshots,var/log}

mount -o noatime,compress=zstd,space_cache=v2,subvol=@boot /dev/sda2 /mnt/boot
mount -o noatime,compress=zstd,space_cache=v2,subvol=@home /dev/sda2 /mnt/home
mount -o noatime,compress=zstd,space_cache=v2,subvol=@snapshots /dev/sda2 /mnt/.snapshots
mount -o noatime,compress=zstd,space_cache=v2,subvol=@var_log /dev/sda2 /mnt/var/log
```

> ⚠️ Важно: `/boot` — это **subvolume**, а не отдельный раздел! Это допустимо при использовании UEFI + GRUB + Btrfs.

### Монтируем ESP

```bash
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

---

## 📦 2. Установка базовой системы

```bash
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs vim nano sudo grub efibootmgr intel-ucode openssh
```

> Если у вас AMD — замените `intel-ucode` на `amd-ucode`.

---

## ⚙️ 3. fstab и chroot

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Проверьте `/mnt/etc/fstab` — все subvolumes должны быть с правильными `subvol=`.

```bash
arch-chroot /mnt
```

---

## 🌐 4. Базовая настройка системы

### Имя хоста

```bash
echo "archlinux" > /etc/hostname

nano /etc/hosts
127.0.0.1 localhost
::1 localhost
127.0.0.1 archlinux.localdomain archlinux
```

### Локали

```bash
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "ru_RU.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

nano /etc/vconsole.conf
KEYMAP=ru
FONT=cyr-sun16
```

### Часовой пояс

```bash
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime  # например, Europe/Moscow
hwclock --systohc
```

### Root-пароль

```bash
passwd
```

### Пользователь

```bash
useradd -m -G wheel -s /bin/bash denis
passwd denis
```

Разрешите `wheel` использовать `sudo`:

```bash
EDITOR=nano visudo
# Раскомментируйте: %wheel ALL=(ALL:ALL) ALL
```

---

## 📶 5. Сеть: LAN, Wi-Fi, Bluetooth

### Установка пакетов

```bash
pacman -S dhcpcd iwd bluez bluez-utils networkmanager
```

> Мы используем **iwd** (modern Wi-Fi daemon) + **NetworkManager** для удобства.

### Включение служб

```bash
systemctl enable dhcpcd
systemctl enable NetworkManager
systemctl enable bluetooth
```

> После перезагрузки вы сможете подключаться к Wi-Fi через `nmtui` или `nmcli`.

---

## 🔄 6. ZRAM (вместо swap)

```bash
pacman -S zram-generator
```

Создайте конфиг:

```bash
cat > /etc/systemd/zram-generator.conf <<EOF
[zram0]
compression-algorithm = zstd
zram-size = ram / 2
EOF
```

> Это создаст zram-устройство размером в половину ОЗУ с сжатием zstd.

Перезапуск не требуется — активируется при следующей загрузке.

---

## 🔊 7. PipeWire (аудио/видео)

```bash
pacman -S pipewire pipewire-pulse pipewire-alsa wireplumber
```

Включите пользовательские сервисы (для обычного пользователя):

> После первого входа в систему под своим пользователем:

```bash
systemctl --user enable pipewire pipewire-pulse

pactl info | grep "Server Name"
# Должно быть: PipeWire
```

(Это можно сделать позже — сейчас достаточно установить.)

---

## 🥾 8. Загрузчик: GRUB + Btrfs

### Установка GRUB

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```

### Настройка GRUB для Btrfs

Отредактируйте `/etc/default/grub`:

```bash
nano /etc/default/grub
```

Добавьте в `GRUB_CMDLINE_LINUX`:

```conf
GRUB_CMDLINE_LINUX="... rootflags=subvol=@"
```

Пример полной строки:

```conf
GRUB_CMDLINEINUX="root=/dev/sda2 rootflags=subvol=@ quiet loglevel=3"
```

> Убедитесь, что указан правильный `root=` (UUID или `/dev/sda2`).

Также разрешите автоматическое обнаружение ядер:

```conf
GRUB_DISABLE_SUBMENU=y
```

### Генерация конфига

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

> GRUB должен корректно загружать ядро даже если `/boot` — subvolume Btrfs (поддерживается с версии GRUB ≥2.04).

---

## 📸 9. Snapper + Btrfs

### Установка

```bash
pacman -S snapper grub-btrfs
```

### Настройка конфигурации по умолчанию

```bash
snapper -c root create-config /
```

Это создаст `/etc/snapper/configs/root`.

Отредактируйте его:

```bash
nano /etc/snapper/configs/root
```

Убедитесь, что:

```ini
SUBVOLUME="/"
FSTYPE="btrfs"
QGROUP=""
...
NUMBER_LIMIT="10"
NUMBER_LIMIT_IMPORTANT="20"
```

> Snapper будет делать снапшоты в `/.snapshots`.

💡 При попытке выполнить команду может возникнуть ошибка:

```
The config 'root' does not exist. Likely snapper is not configured.
```

В этом случае выполните следующие действия:

```bash
sudo rm -rf /.snapshots
sudo snapper -c root create-config /
sudo nano /etc/snapper/configs/root      # Открываем для настройки
sudo chmod a+rx /.snapshots              # Дадим права
sudo snapper -c root list                # Проверка
sudo snapper -c root create -d "test"    # Создадим снапшот для проверки
```
### Настройка прав доступа

```bash
chmod 750 /.snapshots
chown :wheel /.snapshots
```

(Или добавьте своего пользователя в группу, которой разрешён доступ.)

### Автоматические снапшоты при pacman

Установите хук:

```bash
pacman -S snap-pac
```

> Теперь при каждом `pacman -Syu` будет создаваться pre/post снапшот.

---

## 🖥️ 10. X11 (без DE/WM)

```bash
pacman -S xorg-server xorg-xinit xterm
```

> Вы можете позже установить любой WM (i3, dwm и т.д.), но сейчас — только X.

Для запуска X:

```bash
# Под пользователем
startx
```

(По умолчанию запустится `xterm`.)

---

## 🧼 11. Финальные шаги

### Выход и перезагрузка

```bash
exit
umount -R /mnt
reboot
```

---

## ✅ После первой загрузки

1. Войдите под своим пользователем.
2. Запустите `nmtui` для подключения к Wi-Fi (если нужно).
3. Включите пользовательские PipeWire-сервисы:

```bash
systemctl --user enable --now pipewire pipewire-pulse
```

4. Протестируйте звук:

```bash
speaker-test -t wav -c 2
```

5. Проверьте ZRAM:

```bash
zramctl
```

6. Проверьте Snapper:

```bash
snapper list
```

---

## 📝 Дополнительно (по желанию)

- Установите `btrbk` для резервного копирования Btrfs-снапшотов.
- Настройте `fwupd` для обновления firmware.
- Добавьте `tlp` или `powertop` для энергосбережения (особенно на ThinkPad).

---

---

### 🔧 Установка `yay`

1. **Убедитесь, что установлены необходимые зависимости**:
   ```bash
   sudo pacman -S --needed git base-devel
   ```

2. **Клонируйте репозиторий `yay`**:
   ```bash
   git clone https://aur.archlinux.org/yay.git
   cd yay
   ```

3. **Соберите и установите пакет**:
   ```bash
   makepkg -si
   ```

4. **Проверьте установку**:
   ```bash
   yay --version
   ```

---

### 💡 Советы

- После установки `yay` можно использовать как `pacman`, но с поддержкой AUR. Например:
  ```bash
  yay -S имя_пакета_из_AUR
  ```
- Чтобы обновить систему **включая AUR-пакеты**, просто выполните:
  ```bash
  yay
  ```

- При первом запуске `yay` предложит настроить параметры — можно оставить значения по умолчанию или настроить под себя.

---

Готово! У вас минимальная, но полностью функциональная Arch-система с Btrfs, ZRAM, PipeWire, Snapper и поддержкой всех указанных компонентов.

Если вы используете **ThinkPad X230i** — драйверы уже включены в `linux-firmware`, всё должно работать «из коробки».
 
## 💡 Запуск редактора `nano` с нумерацией строк 

Чтобы открыть файл в редакторе `nano` с включённой нумерацией строк, используйте флаг `-l` (или `--linenumbers`):

```bash
nano -l
```

Если вы хотите открыть конкретный файл с нумерацией строк:

```bash
nano -l имя_файла
```

> **Примечание:** Флаг `-l` доступен в версиях `nano` 5.0 и выше. Убедитесь, что ваша версия поддерживает эту опцию (`nano --version`).
```

Удачи! 🐧









