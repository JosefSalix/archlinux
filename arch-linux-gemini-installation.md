## Disk Partitioning

```bash
fdisk /dev/sda

g           # Vytvori novou tabulku oddilu typu GPT (nutne pro moderni UEFI)

n           # Vytvorit 1. oddil (Boot/EFI)
[Enter]     # Výchozi cislo oddilu (1)
[Enter]     # Vychozi prvni sektor
+1G         # Velikost: 1 Gigabajt

n           # Vytvorit 2. oddil (Swap)
[Enter]     # Vychozi cislo oddilu (2)
[Enter]     # Vychozi prvni sektor
+8G         # Velikost: 8 Gigabajtu

n           # Vytvorit 3. oddil (Root)
[Enter]     # Vychozi cislo oddilu (3)
[Enter]     # Vychozi prvni sektor
+100G       # Velikost: 100 Gigabajtu pro system a programy

n           # Vytvorit 4. oddil (Home)
[Enter]     # Vychozi cislo oddilu (4)
[Enter]     # Vychozi prvni sektor
[Enter]     # Vychozi posledni sektor (vyuzije zbytek disku pro tva data)

t           # Zmena typu oddilu (Type)
1           # Vybereme oddil 1
1           # Nastavime typ na "EFI System"

t           # Zmena typu oddilu
2           # Vybereme oddil 2
19          # Nastavime typ na "Linux swap"

w           # Zapsat zmeny na disk a ukoncit fdisk (Write)
```

## File System Formatting

```bash
mkfs.fat -F32 /dev/sda1     # Naformatuje prvni oddil na FAT32 (nutne pro UEFI boot)
mkswap /dev/sda2            # Pripravi druhy oddil pro pouziti jako Swap (virtualni pamet)
mkfs.ext4 /dev/sda3         # Naformatuje treti oddil (Root) na stabilni system ext4
mkfs.ext4 /dev/sda4         # Naformatuje ctvrti oddil (Home) na ext4 pro tva uzivatelska data
```

## Mounting File Systems

```bash
mount /dev/sda3 /mnt        # Pripoji hlavni systemovy oddil (Root) do adresare /mnt
mkdir /mnt/home             # Vytvori slozku pro uzivatelska data uvnitr pripojeneho systemu
mount /dev/sda4 /mnt/home   # Pripoji datovy oddil (Home) do teto nove slozky

mkdir /mnt/boot             # Vytvori slozku pro bootovaci soubory
mount /dev/sda1 /mnt/boot   # Pripoji EFI systemovy oddil (Boot) do teto slozky

swapon /dev/sda2            # Aktivuje swap space (virtualni pamet), aby s ni system uz mohl pocitat
```

## Base System Installation

```bash
# Nainstaluje naprosty zaklad systemu do slozky /mnt (zatim bez jader a site, ty dame v chrootu)
pacstrap -i /mnt base

# Vygeneruje tabulku ulozist (fstab) podle toho, jak mame ted disky pripojene.
# Pouzivame prepinac '-U' pro UUID (unikatni kody disku, coz je bezpecnejsi).
genfstab -U /mnt >> /mnt/etc/fstab
```

## System Configuration (Chroot)

```bash
# Tento prikaz te "teleportuje" do tveho noveho systemu na SSD disku.
# Od ted vsechny prikazy spustis primo v novem Arch Linuxu.
arch-chroot /mnt

vim /etc/pacman.conf
# Otevres konfiguracni soubor spravce balicku Pacman.
# Najdi v sekci [options] a odkomentuj nebo pridej tyto radky:
  # ParallelDownloads = 5   # stahuje 5 balicku naraz
  # Color                   # zapne barvy v terminalu
  # ILoveCandy              # Pac-Man misto nudnych tecek
```

## Linux Kernels & Hardware Support

```bash
# Ted, kdyz uz nam pacman stahuje rychle, doinstalujeme jadra a ovladace:
# - klasicke nejnovejsi jadro (linux) a zalozni stabilni jadro (linux-lts) vcetne jejich headers
# - 'linux-firmware' jsou univerzalni ovladace pro hardware
# - 'intel-ucode' je mikrokod pro tvuj i5 procesor (stabilita a bezpecnost)
# - 'vim' je textovy editor, abychom mohli pokracovat v konfiguraci
pacman -S linux linux-headers linux-lts linux-lts-headers linux-firmware intel-ucode vim
```

## Remote Access (SSH)

```bash
# Nainstaluje vyvojarske nastroje (base-devel) a SSH server pro vzdaleny pristup
pacman -S base-devel openssh

# Nastavi SSH server tak, aby se automaticky spustil pri kazdem startu pocitace
systemctl enable sshd
```

## Network Configuration

```bash
# Nainstaluje moderniho spravce site pro kabel i wifi
pacman -S networkmanager

# Zapne automaticke spousteni sitoveho spravce po startu PC (Pozor na velka pismena!)
systemctl enable NetworkManager
```

## Localization & Network Identity

```bash
# Otevre soubor s prehledem jazyku. Najdi v nem (pomoci klavesy '/' pro vyhledavani)
# radek '#en_US.UTF-8 UTF-8' a odkomentuj ho.
vim /etc/locale.gen

# Tento prikaz nyni procte soubor a vygeneruje anglickou lokalizaci do systemu.
locale-gen

# Nastavi anglictinu jako hlavni systemovy jazyk.
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Nastavi casove pasmo na Prahu (stredni Evropa), takze budes mit spravny cas.
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime

# Synchronizuje hardwarove hodiny na zakladni desce s casem, ktery jsme prave nastavili.
hwclock --systohc

# Nastavi jmeno tveho pocitace (Hostname). Misto 'archlinux' si muzes zvolit jakekoliv jmeno.
echo "archlinux" > /etc/hostname

# Otevre soubor pro lokalni smerovani adres. Pridej na konec souboru tyto tri radky:
# (Pokud jsi zvolil jine jmeno pocitace nez archlinux, zmen ho i tady na poslednim radku)
  # 127.0.0.1 localhost
  # ::1 localhost
  # 127.0.1.1 archlinux.localdomain archlinux
vim /etc/hosts
```

## Users & Passwords

```bash
# Nastavi hlavni heslo pro uzivatele 'root' (spravce systemu).
# Terminal pri psani hesla neukazuje ani hvezdicky, takze pis naslepo a potvrd Enterem.
passwd

# Vytvori tveho normalniho uzivatele se jmenem 'salix'.
# Prepinac '-m' vytvori tvuj domovsky adresar (/home/salix).
# Prepinac '-G wheel' te prida do dulezite skupiny 'wheel', ktera ma pravo pouzivat prikaz sudo.
useradd -m -G wheel salix

# Nastavi prihlasovaci heslo pro tveho noveho uzivatele 'salix'.
passwd salix
```

## Sudo Privileges (Visudo)

```bash
# Otevre konfiguracni soubor pro prava 'sudo' v editoru Vim.
# Najdi radek: # %wheel ALL=(ALL:ALL) ALL  a odkomentuj ho.
EDITOR=vim visudo
```

## Bootloader Installation (GRUB)

```bash
# Nainstaluje samotny zavedac GRUB a nastroj efibootmgr, ktery umi komunikovat s UEFI tve zakladni desky
pacman -S grub efibootmgr

# Nainstaluje GRUB pro 64-bitove UEFI.
# Jako EFI adresar pouzijeme jednoduse '/boot', kde uz mame nas disk sda1 pripojeny.
# '--bootloader-id=GRUB' je nazev, ktery uvidis v BIOSu/UEFI tveho PC pri vyberu bootovani.
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Tento prikaz prohleda system (najde klasicke i LTS jadro, ktere jsme nainstalovali)
# a vygeneruje hlavni konfiguracni soubor pro GRUB menu.
grub-mkconfig -o /boot/grub/grub.cfg

# POZNAMKA PRO DUALBOOT (Pokud bys nekdy mel na disku i Windows):
# Pokud bys potreboval os-prober, nainstaluj ho pres 'pacman -S os-prober',
# otevri soubor '/etc/default/grub', na konec pridej radek 'GRUB_DISABLE_OS_PROBER=false'
# a znovu spust ten prikaz 'grub-mkconfig' vyse.
```

## Time Synchronization & Display Server

```bash
# Zapne systemovou sluzbu, ktera bude pres internet automaticky udrzovat presny cas
systemctl enable systemd-timesyncd

# Nainstaluje zakladni zobrazovaci server Xorg, ktery je potreba pro beh vetsiny grafickych prostredi
pacman -S xorg-server
```

## System Exit & Reboot

```bash
# Odejde z prostredi chroot zpet na instalacni medium Archu
exit

# Bezpecne a hlavne rekurzivne (prepinac -R) odpoji vsechny disky a oddily pripojene v /mnt
umount -R /mnt

# Restartuje pocitac. V tuto chvili muzes vytahnout instalacni USB flashku.
# Flashku vytahni hned pote, co zadas prikaz `reboot` a obrazovka ti zcerna (pocitac se zacne vypinat a restartovat).
# Pokud ji vytahnes pred zadanim `reboot`, riskujes, ze se nedokonci nejake zaverecne zapisy na disk.
# Pokud ji tam naopak nechas moc dlouho, pocitac muze znovu nabootovat do instalacniho media misto do noveho systemu.
# Kdyby se to stalo, nic se nedeje, instalace se nerozbila – jen PC vypni tlacitkem, vytahni flashku a zapni ho znova.
reboot
```

## Graphical Environment (Cinnamon & LightDM)

```bash
# Nainstaluje prostredi Cinnamon, jeho krasne aplikace a prihlasovaci obrazovku LightDM s grafickym nastavenim
# - balicek `accountsservice` zarucuje, ze si LightDM zapamatuje tvuj avatar a nastaveni.
pacman -S cinnamon lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings accountsservice

# Aktivuje prihlasovaci obrazovku. Po restartu uz nenabehne cerna konzole, ale graficke prihlaseni.
systemctl enable lightdm
```

## GTK Themes & Icons (Pro hezky vzhled v XMonadu)

```bash
# Nainstaluje jedno z nejhezcich modernich temat (Arc), skvely set ikon (Papirus)
# a nastroj LXAppearance, ve kterem si pak v XMonadu jednoduse naklikas, aby vsechny aplikace vypadaly krasne.
pacman -S arc-gtk-theme papirus-icon-theme lxappearance
```

## XMonad & Power Tools (Tiling WM Setup)

```bash
# Nainstaluje samotny XMonad, jeho rozsireni (contrib) a kompilator jazyka Haskell (ghc + cabal)
# nutny pro spravne fungovani a re-kompilaci konfiguracniho souboru xmonad.hs
pacman -S xmonad xmonad-contrib ghc cabal-install git

pacman -S kitty rofi dmenu polybar feh nitrogen picom
```

## Audio Setup (PipeWire)

```bash
# Nainstaluje moderni audio server PipeWire, ktery ma na starosti zvuk a mikrofon.
# - pipewire-pulse a alsa zajisti zpetnou kompatibilitu se vsemi programy a hrami
# - wireplumber je spravce relaci pro audio
# - pavucontrol je graficke ovladani hlasitosti (Volume Control), ktere spustis kdekoliv
pacman -S pipewire pipewire-pulse pipewire-alsa wireplumber pavucontrol
```