# Sestavení Armbianu pro DOTPO1503

Níže je postup pro ověřenou výchozí verzi. Čistý obraz byl sestaven na Ubuntu 24.04 x86-64 z commitu `d4112b544a73369686572e051b65e83e645bc09f` projektu `armbian/build`. Výstup byl Debian Trixie, Xfce minimal, kernel 6.18.53 a U-Boot 2025.10. Novější commit či kernel mohou vyžadovat úpravu patchů.

## Zdrojové soubory

Na linuxovém kompilačním stroji stáhněte tento repozitář a Armbian do dvou samostatných adresářů. Existující pracovní build není nutné měnit.

```bash
git clone https://github.com/AidenShaw2020/armbian-dotpo1503.git
git clone https://github.com/armbian/build.git armbian-build
cd armbian-build
git checkout d4112b544a73369686572e051b65e83e645bc09f

git apply --check ../armbian-dotpo1503/patches/armbian/0001-dotpo1503-integration.patch
git apply ../armbian-dotpo1503/patches/armbian/0001-dotpo1503-integration.patch

install -d userpatches/kernel/archive/rockchip-6.18
cp ../armbian-dotpo1503/patches/kernel/*.patch userpatches/kernel/archive/rockchip-6.18/
install -d userpatches/u-boot/v2025.10/board_dotpo1503
cp ../armbian-dotpo1503/patches/u-boot/v2025.10/board_dotpo1503/*.patch \
  userpatches/u-boot/v2025.10/board_dotpo1503/
```

Patch `armbian/0001` vytvoří také `userpatches/config-dotpo1503.conf`, `userpatches/customize-image.sh`, konfigurační soubor jádra a textové soubory pro overlay. Binární pomocný program pro pravý klik se vytváří v dalším kroku a záměrně není v GitHubu.

## Pomocný program pro dlouhý stisk

Na Debianu Trixie **armhf** (například na již spuštěné pokladně) sestavte původní projekt [evdev-right-click-emulation](https://github.com/PeterCxy/evdev-right-click-emulation) přesně z commitu `350939b8d2e95ed1be352f1ac734bdedf14e43c7`. V ověřeném obrazu byl použit tento příkaz:

```bash
sudo apt-get install gcc libc6-dev libevdev-dev
git clone https://github.com/PeterCxy/evdev-right-click-emulation.git
cd evdev-right-click-emulation
git checkout 350939b8d2e95ed1be352f1ac734bdedf14e43c7
gcc -O2 -Wall -std=c11 -D_POSIX_C_SOURCE=199309L \
  -I/usr/include/libevdev-1.0 uinput.c input.c rce.c -levdev \
  -o dotpo-evdev-rce-armhf
sha256sum dotpo-evdev-rce-armhf
```

Přeneste výsledný soubor do `armbian-build/userpatches/overlay/dotpo1503/dotpo-evdev-rce-armhf`. Ověřená binárka měla SHA-256 `d6c010a90f845959cb79eeb0a25d1cb26dfcc6e80dfdd16fcd2daf378bf82dc3`; build skript obraz při jiné hodnotě úmyslně zastaví. Odlišný hash nejdřív prověřte proti použitému commitu, architektuře a sestavovacímu prostředí. V případě jiného legitimního sestavení aktualizujte kontrolní součet v `userpatches/customize-image.sh`.

## Kompilace

Armbian build potřebuje běžné hostitelské závislosti a `sudo`; použijte jej pod neprivilegovaným uživatelem. Konfigurace v `userpatches/config-dotpo1503.conf` nastavuje board, Trixie, Xfce, SSH a kernel bez interaktivní změny konfigurace.

```bash
cd armbian-build
./compile.sh dotpo1503
cd output/images
sha256sum -c Armbian-unofficial_*_Dotpo1503_*_xfce_desktop.img.sha
```

Očekávaný název obrazu pro ověřený commit je `Armbian-unofficial_26.11.0-trunk_Dotpo1503_trixie_current_6.18.53_xfce_desktop.img`. Úspěšný build ověřte také přítomností `rk3288-dotpo1503.dtb`, povolené služby `ssh.service` a balíků `onboard`, `gpiod`, `libevdev2` v rootfs.

## Příprava bootovacího média

Výstupní obraz Armbianu poskytuje rootfs, ale samotný na testované pokladně z SD nestartoval. Ověřená karta měla toto rozložení:

| Oblast | Začátek | Velikost | Obsah |
| --- | ---: | ---: | --- |
| zavaděč | 0 MiB | 64 MiB | první 64 MiB z lokálně ověřené Multitool karty |
| oddíl 1 | 64 MiB | 128 MiB | FAT32, boot soubory z nového Armbianu |
| oddíl 2 | 192 MiB | velikost ext4 rootfs | ext4 oddíl z nového Armbianu |

Do FAT32 byly z rootfs zkopírovány `/boot/zImage`, verzovaný `/boot/initrd.img-*` jako `initrd.img` a `/boot/dtb/rk3288-dotpo1503.dtb`. Soubor `extlinux/extlinux.conf` ukazuje na tyto tři soubory a na UUID druhého oddílu (`root=UUID=… rootwait rootfstype=ext4 rw`). Pro zobrazení splash screenu byly ověřeny parametry `quiet splash loglevel=3 plymouth.ignore-serial-consoles vt.global_cursor_default=0 consoleblank=0`; sériová konzole je `ttyS4,115200n8`.

Binární hlavičku Multitool ani hotový obraz zde nezveřejňujeme. Než přepíšete eMMC, ověřte start a periferie z SD a uložte úplnou zálohu původní eMMC. Instalace do eMMC má používat již ověřený SD zavaděč; samotný U-Boot balík z Armbian buildu jej nenahrazuje.
