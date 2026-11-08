# Moto G32 (devon, XT2235-1) con iOS 16: notas para reconstruir

Base: fork de AdyaGMD/ios-spes (rama spes). Snapdragon 680 (SM6225), 6 GB, bootloader desbloqueado.

## Cambios respecto al repo original
- En el fork de GitHub: xnu/patches/0028-sweet-ufs-root-device.patch, la linea "su.lun = 0;" pasa a "su.lun = 4;" (userdata esta en la LUN 4).
- devon-changes.patch: diff local. tools/mkadt.py usa el unique-chip-id sintetico 0x4733320000000001. shim/build.sh usa os_version 17.0.0 y patch level 2026-10 (los que acepta el bootloader de Motorola).
- tools/build-devon-ios.sh: imagen de arranque para el G32 (tactil apagado, cpus=4, sweetfb_rate=0, escritura habilitada).

## Datos del equipo
- userdata es el disco sde27 (LUN 4), inicio en el sector 15044608 (de 512 B), o sea el bloque 0x1cb200 (de 4096 B). Tamano: 118235312128 bytes.
- Framebuffer 0x5c000000, 1080x2400, stride 4320. Panel TXD ili7807s. Tactil Ilitek por SPI (todavia sin driver).
- RAM para XNU: 0x100000000 + 3 GiB (lo calcula tools/spes_memory.py con el FDT en vivo).
- El bootloader arranca en un nucleo Silver (sin paginas de 16K): el shim pasa a un Gold por PSCI.

## Lo que NO esta en el repo y hay que rehacer
1. FDT en vivo: con Android y adb root, "adb pull /sys/firmware/fdt devon.dtb". Es privado, no se sube.
2. Respaldo de particiones (persist, modemst1/2, fsg, etc.). Privado.
3. IPSW iPhone X (iPhone10,3) 16.7.16 (20H392) desde ipsw.me, SHA256 ece1194f7e233e9109c5790a8d23a4564a10a254e15ec5b09978a343265568fb. DMG usados: 087-86698-044 (OS), 087-86593-047 (SystemOS), 087-86715-047 (AppOS).
4. research/ipsw-x/d22ap.adt: extraer Firmware/all_flash/DeviceTree.d22ap.im4p del IPSW y correr "pyimg4 im4p extract -i ESE.im4p -o d22ap.adt" (186084 bytes).
5. Kernel: GitHub Actions del fork, workflow xnu, rama spes. Bajar el artifact xnu-kernel a xnu/out/kernel.development.sweet.
6. Sistema: tools/mkiosroot.sh (Linux: apfs-fuse, hfsprogs, sudo) y tools/sweet-root-tweaks.py, 16G. Se grabo en userdata en formato sparse con fastboot desde el fastbootd del recovery de LineageOS.

## Armar la imagen de arranque y probar (solo RAM)
  MKBOOTIMG=$HOME/hackios-g32/mkbootimg/mkbootimg.py tools/build-devon-ios.sh xnu/out/kernel.development.sweet ~/hackios-g32/devon.dtb
  adb reboot bootloader
  fastboot boot shim/out/devon-ios.img
Nunca "fastboot flash" de boot, vendor_boot ni nada que no sea userdata.
Para sacar el log: apagar, entrar al recovery de LineageOS, adb root, tools/sweet-log-pull.sh logs/NOMBRE.txt.

## Estado (nov 2026)
El kernel monta el HFS+ de userdata y arrancan launchd, backboardd y SpringBoard. backboardd presenta cuadros en el panel. SpringBoard aborta en bucle con:
  Failed to load bounding path bitmap data from the asset manager for asset name: BoundingPathBitmap-1125x2436-375x812-3.00x
El Assets.car de ese recurso es identico al del IPSW. Ya probado sin efecto: nodo /product del D22AP en el ADT, UUID sintetico nuevo, sweetfb_rate=0, cpus=4.
Pendiente: preguntar al autor (issue en AdyaGMD/ios-spes) y, si no hay respuesta, simbolizar el crash contra la cache de iOS.

## Volver a Android (NO probado en este modelo)
Tiene que ser desde el recovery de LineageOS: formatear datos. El sistema de Android (super) nunca se toco.
