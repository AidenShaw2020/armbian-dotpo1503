# Armbian pro DOTPO1503 (Dotykačka) (RK3288)

Zdrojové úpravy pro pokladnu DOTPO1503 (Dotykačka): Debian Trixie, jádro Armbian `current` 6.18, Xfce a SSH. Základ je pevně určený commit projektu [armbian/build](https://github.com/armbian/build) `d4112b544a73369686572e051b65e83e645bc09f`. Přesný postup je v [BUILD.md](BUILD.md).

Historie Gitu obsahuje pouze textové patche a dokumentaci. Ověřený čistý obraz, hlavička zavaděče a balík U-Boot jsou samostatné soubory v [Releases](https://github.com/AidenShaw2020/armbian-dotpo1503/releases); do Gitu nepatří velké binární soubory. Nezveřejňujeme původní Android, přístupové údaje, lokální konfiguraci kiosku ani experimentální skripty.

## Co patche mění

- `patches/armbian/`: definice desky, boot skript, konfigurace jádra a nastavení čistého obrazu. Součástí je podpora české dotykové klávesnice Onboard, displeje, zvuku, LED tlačítka a dlouhého podržení pro pravý klik.
- `patches/kernel/`: device tree převzatý z analýzy původního Androidu, 1920×1080 eDP, oprava zhasínání eDP při PSR, RTL8821CS přes SDIO, ES8316 a vypnutí přes RK808.
- `patches/u-boot/`: podklady profilu XT-Q8L-V10 pro U-Boot 2025.10, které Armbian používá při sestavení balíků. Původní patche napsal Paolo Sabatino.

Licence tohoto repozitáře je GPL-2.0; její plné znění je v [LICENSE](LICENSE). U převzatých patchů platí také původní autorská označení a SPDX uvedená přímo v souborech (zejména alternativní `GPL-2.0+ OR MIT` u kernelového device tree). Licence binárních součástí obrazu jsou popsány v [BUILD.md](BUILD.md#p%C5%99%C3%ADprava-bootovac%C3%ADho-m%C3%A9dia) a v samotném obrazu.

Na fyzické pokladně byl ověřen start čistého obrazu z SD a eMMC, obraz přes celý panel, Ethernet, Wi-Fi, USB dotyk, zvuk, dvojklik, dlouhý stisk pro pravý klik, česká klávesnice, zhasnutí LED při vypnutí a následné zapnutí tlačítkem. SSH server je v obrazu. Uživatelské přihlášení a případnou aplikaci kiosku si nastavuje každý sám po prvním startu.

**Důležité:** výsledný `.img` přímo z Armbian build systému není na této pokladně samostatně bootovatelný. Ověřený obraz z SD používal první 64 MiB funkčního Multitool zavaděče, za nimi 128MiB oddíl FAT32 s kernelem, initramfs a DTB a pak ext4 rootfs z nového Armbianu. Hotový komprimovaný obraz a samostatná hlavička jsou v Releases; podrobnosti jsou v [BUILD.md](BUILD.md#p%C5%99%C3%ADprava-bootovac%C3%ADho-m%C3%A9dia).

Změny vycházejí z konkrétní revize desky DOTPO1503. Před instalací na jinou revizi ověřte napájení SD/eMMC, zapojení RK808, eDP panel a GPIO podle jejího schématu nebo původního DTB.
