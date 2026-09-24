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

Ve vydání `v2026.09.24` jsou samostatně dostupné:

- `DOTPO1503-Armbian-clean-v4-public.img.xz`: celý obraz SD, komprimovaný formátem XZ. Byl připraven z fyzicky ověřené čisté verze; před zveřejněním byly odstraněny vygenerované SSH host keys. Při prvním startu se vytvoří nové.
- `DOTPO1503-tested-multitool-loader-first64MiB.bin`: prvních 64 MiB ověřeného obrazu pro reprodukci rozložení. Obsažené `idbloader.img` a `u-boot-dtb.bin` se binárně shodují s [Multitool pro RK3288](https://github.com/paolosabatino/multitool/tree/d7dc392fe33850f5975aa466538fd889629c430c/sources/rk3288).
- `linux-u-boot-dotpo1503-current_*.deb`: U-Boot 2025.10 vytvořený Armbian buildem ze [zdrojového commitu `e50b1e8`](https://github.com/u-boot/u-boot/tree/e50b1e8715011def8aff1588081a2649a2c6cd47) a zdejších U-Boot patchů. **Tento balík není zavaděčem, s nímž byl ověřen start SD a eMMC**, a nemá se naslepo instalovat na fungující zařízení.
- `DOTPO1503-package-manifest.tsv` a `SHA256SUMS`: verze nainstalovaných balíků a kontrolní součty release souborů.

Hotový obraz lze zapsat přímo z `.img.xz` pomocí balenaEtcher. Po stažení ověřte `sha256sum -c SHA256SUMS`. Obraz je pro konkrétní hardwarovou revizi DOTPO1503; nejprve ho otestujte na SD. Než přepíšete eMMC, uložte úplnou zálohu původní eMMC. Instalace do eMMC má používat již ověřený SD zavaděč; samostatný U-Boot balík z Armbian buildu jej nenahrazuje.

Zdrojové úpravy obrazu jsou v tomto repozitáři a přesný výchozí commit Armbianu je uveden výše. Multitool vychází z projektu Paola Sabatina pod GPL-2.0; binární komponenty Rockchip používají [licenci rkbin](https://github.com/rockchip-linux/rkbin/blob/master/LICENSE). Licenční texty balíků systému jsou v obrazu pod `/usr/share/doc/*/copyright` a přesné verze v manifestu. Při dalším šíření obrazu zachovejte tato oznámení a zpřístupněte odpovídající zdrojové kódy podle licencí jednotlivých komponent.

## Instalace z otestované SD na eMMC

Tento postup byl fyzicky ověřen na DOTPO1503 s interní eMMC **AJTD4R o velikosti 15 634 268 160 B**. Na testované jednotce byl běžící Armbian z SD `/dev/mmcblk0p2` a interní eMMC `/dev/mmcblk2`. Instalace eMMC celou přepíše včetně Androidu. Nejdříve ověřte start výše uvedeného release obrazu z SD a uložte úplnou zálohu původní eMMC mimo pokladnu; její SHA-256 ověřte znovu před instalací. Na jiném modelu eMMC ani při jiném přiřazení `/dev/mmcblk*` tento blok příkazů nespouštějte.

Na pokladně spusťte následující blok jako celek. Před zápisem kontroluje model a velikost eMMC, UUID kořene vydaného obrazu, hash funkčního zavaděče i to, že systém běží skutečně ze SD. Používá **zavaděč ze SD**, nikoli samostatný balík U-Boot 2025.10. Po kontrolách požaduje ruční opsání cílového zařízení.

```bash
sudo bash <<'INSTALL_DOTPO_EMMC'
set -euo pipefail
src=/dev/mmcblk0
dst=/dev/mmcblk2
sd_uuid=30730f05-5050-48c6-8d5e-8b15e738d8fc
loader_sha=22354e789e1928e1f6b8b9c746755da05407b63009ef849d1c7fd84caecfeab9
work=/run/dotpo-emmc-install

test "$(findmnt -n -o SOURCE /)" = "${src}p2"
test "$(blkid -s UUID -o value "${src}p2")" = "$sd_uuid"
test "$(blkid -s LABEL -o value "${src}p1")" = DOTPO_BOOT
test "$(blockdev --getsize64 "$dst")" = 15634268160
test "$(cat /sys/block/mmcblk2/device/name)" = AJTD4R
! findmnt -rn -o SOURCE | grep -q '^/dev/mmcblk2'
! findmnt -rn -o SOURCE | grep -q '^/dev/mmcblk0p1'
mmc extcsd read "$dst" | grep -Fq 'Boot configuration bytes [PARTITION_CONFIG: 0x00]'
source_hash=$(dd if="$src" bs=512 skip=1 count=131071 status=none | sha256sum | cut -d ' ' -f 1)
test "$source_hash" = "$loader_sha"

mkdir -p "$work/srcboot" "$work/root" "$work/boot"
cleanup() {
    for path in "$work/boot" "$work/root" "$work/srcboot"; do
        if mountpoint -q "$path"; then umount "$path"; fi
    done
}
trap cleanup EXIT
mount -o ro "${src}p1" "$work/srcboot"
grep -Fq "root=UUID=$sd_uuid" "$work/srcboot/extlinux/extlinux.conf"
grep -Fq "$sd_uuid" "$work/srcboot/boot/armbianEnv.txt"
grep -Fq "$sd_uuid" /boot/armbianEnv.txt
grep -Fq "$sd_uuid" /etc/fstab

echo 'Kontroly prošly. Cílová eMMC bude zcela přepsána.'
read -r -p 'Pro pokračování opište /dev/mmcblk2: ' confirm </dev/tty
test "$confirm" = "$dst"

wipefs -a "$dst"
dd if="$src" of="$dst" bs=1M count=64 iflag=fullblock conv=fsync status=progress
sfdisk --wipe never --wipe-partitions never "$dst" <<'PARTITIONS'
label: dos
label-id: 0xd0151503
unit: sectors

/dev/mmcblk2p1 : start=131072, size=262144, type=c
/dev/mmcblk2p2 : start=393216, size=30140416, type=83
PARTITIONS
partprobe "$dst"
udevadm settle
test "$(blockdev --getsz "${dst}p1")" = 262144
test "$(blockdev --getsz "${dst}p2")" = 30140416

mkfs.vfat -F 32 -n DOTPO_EMMC "${dst}p1"
mkfs.ext4 -F -L armbi_emmc "${dst}p2"
emmc_uuid=$(blkid -p -s UUID -o value "${dst}p2")
test -n "$emmc_uuid" && test "$emmc_uuid" != "$sd_uuid"
mount "${dst}p2" "$work/root"
mount "${dst}p1" "$work/boot"
rsync -aHAXx --numeric-ids / "$work/root/"
sync
rsync -aHAXx --numeric-ids / "$work/root/"
cp -a "$work/srcboot/." "$work/boot/"
for file in "$work/root/etc/fstab" "$work/root/boot/armbianEnv.txt" \
            "$work/boot/extlinux/extlinux.conf" "$work/boot/boot/armbianEnv.txt"; do
    test -f "$file"
    sed -i "s/$sd_uuid/$emmc_uuid/g" "$file"
done
grep -Fq "root=UUID=$emmc_uuid" "$work/boot/extlinux/extlinux.conf"
grep -Fq "UUID=$emmc_uuid / ext4" "$work/root/etc/fstab"
sync
cleanup
fsck.vfat -n "${dst}p1"
e2fsck -fn "${dst}p2"
target_hash=$(dd if="$dst" bs=512 skip=1 count=131071 status=none | sha256sum | cut -d ' ' -f 1)
test "$target_hash" = "$loader_sha"
echo "Instalace hotová; kořen eMMC má UUID $emmc_uuid"
INSTALL_DOTPO_EMMC
```

Po úspěšném dokončení vypněte pokladnu, vyjměte SD a zapněte ji tlačítkem. Ověřte, že kořenový oddíl běží z `/dev/mmcblk2p2`, a potom otestujte displej, síť, dotyk, zvuk a vypnutí. Neodstraňujte soukromou zálohu původního Androidu.
