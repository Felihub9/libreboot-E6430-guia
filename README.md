# Instal·lació de Libreboot a un Dell Latitude E6430 gravant el microprogramari internament

Aquesta guia documenta el procés real que vaig seguir per instal·lar
[Libreboot](https://libreboot.org) (microprogramari lliure basat en coreboot) a un
Dell Latitude E6430, gravant-lo des del mateix sistema operatiu, sense obrir la
carcassa ni fer servir cap programador extern (Raspberry Pi, CH341A, etc.).

## Motivació i context

No volia telemetria de cap tipus, ni la de l'Intel ME ni la del microprogramari
propietari del fabricant.

Abans de tocar res vaig confirmar:

- El Dell Latitude E6430 és un model **oficialment suportat** per Libreboot.
- La documentació oficial indica que els portàtils Dell Latitude d'aquesta generació
  permeten gravar-hi el microprogramari des del mateix sistema, sense necessitat de
  desmuntar-los.
- **Punt crític a comprovar abans de començar:** que el model sigui la variant amb GPU
  Intel (la variant amb GPU Nvidia no és compatible).

> Segons la documentació de Libreboot, en aquesta generació de portàtils (Ivy Bridge)
> l'Intel **ME no es pot eliminar del tot**. Libreboot en redueix el microprogramari amb
> `me_cleaner`, i coreboot pot demanar-li que es desactivi després d'arrencar (ME Soft
> Temporary Disable). Per tant, aquest procés **redueix** l'Intel ME però no l'apaga del tot,
> i no he mesurat què fa o deixa de fer després. Si el teu objectiu és el mateix que el
> meu, tingues-ho present.

Aquest document recull el procés complet ja fet i verificat en un dispositiu real.

## ⚠️ Avís important

- Gravar el microprogramari **comporta risc de deixar l'equip inutilitzable** si el
  procés falla o s'interromp. Fes sempre una còpia de seguretat del microprogramari
  original abans de començar, i guarda-la fora de l'equip (en un suport extern).
- Aquest procés es va fer en un Dell Latitude E6430 concret; en altres unitats el
  comportament pot variar.
- Segueix els passos amb calma, llegeix cada comanda abans d'executar-la, i no
  interrompis el procés de gravació un cop iniciat.

## Equip utilitzat

- Un Dell Latitude E6430 amb **GPU Intel** (HD Graphics 4000).
- Debian (aquí es va fer servir Debian 13 "Trixie" amb XFCE, kernel 6.12).
- Carregador connectat durant tot el procés.
- Connexió a internet per descarregar les eines i la imatge de Libreboot.
- Un suport extern (USB) per guardar la còpia de seguretat del microprogramari original.

## Pas 1 — Preparar l'entorn i les eines

Instal·la les dependències de compilació:

```bash
sudo apt install -y git build-essential libpci-dev libusb-1.0-0-dev \
  pkg-config zlib1g-dev libftdi1-dev cmake meson ninja-build \
  wget curl xz-utils python3 flashrom autoconf automake libtool \
  innoextract python-is-python3
```

Clona les eines necessàries:

```bash
git clone https://codeberg.org/libreboot/lbmk.git ~/lbmk
git clone https://github.com/nic3-14159/dell-flash-unlock.git ~/dell-flash-unlock
```

Compila `dell-flash-unlock`:

```bash
cd ~/dell-flash-unlock && make
```

## Pas 2 — Comprovar l'accés a la flash SPI

```bash
sudo flashrom -p internal
```

En el meu cas, el chipset (Intel QM77) es detectava correctament, però l'accés a
`/dev/mem` estava bloquejat pel kernel amb l'error `Operation not permitted`.

**Solució:** vaig afegir el paràmetre de kernel `iomem=relaxed` a
`/etc/default/grub` (línia `GRUB_CMDLINE_LINUX_DEFAULT`), vaig executar
`sudo update-grub` i vaig reiniciar.

Un cop reiniciat, vaig comprovar que el paràmetre s'havia aplicat:

```bash
cat /proc/cmdline
```

## Pas 3 — Desbloquejar la flash SPI

Amb el carregador connectat:

```bash
sudo ~/dell-flash-unlock/dell_flash_unlock
```

Aquesta primera execució marca l'override del descriptor de flash per a la propera
arrencada. Segons la sortida de l'eina, cal apagar **completament** el sistema (no
reiniciar):

```bash
sudo shutdown -h now
```

En el meu cas, l'equip no es va engegar automàticament després d'apagar-se (segons la
documentació de l'eina, això és el comportament esperat en aquesta generació de
Latitude), així que el vaig engegar manualment.

Un cop arrencat de nou, vaig tornar a executar la mateixa comanda per confirmar el
desbloqueig:

```bash
sudo ~/dell-flash-unlock/dell_flash_unlock
```

Va respondre `Flash is unlocked. Internal flashing should work.` Ho vaig verificar amb:

```bash
sudo flashrom -p internal
```

Totes les regions (Flash Descriptor, BIOS, Management Engine, Gigabit Ethernet)
apareixien com a `read-write`.

## Pas 4 — Fer una còpia de seguretat del microprogramari original

```bash
mkdir -p ~/libreboot-backup && cd ~/libreboot-backup
sudo flashrom -p internal -r factory_backup_1.rom
sudo flashrom -p internal -r factory_backup_2.rom
sudo flashrom -p internal -r factory_backup_3.rom
sha256sum factory_backup_*.rom
```

Els tres fitxers van donar el mateix hash, confirmant que la lectura havia estat
fiable. Els vaig copiar a un pendrive extern i **no els incloc en aquest
repositori**: és microprogramari amb identificadors propis del meu equip (com la MAC
address original), no aporta cap valor per a algú altre, i no té sentit publicar-lo.

## Pas 5 — Descarregar la imatge oficial de Libreboot

Vaig descarregar la imatge (variant `e6430_12mb`, versió 26.01rev1) des d'un mirall
oficial, juntament amb el fitxer de checksum i la signatura:

```bash
wget <url_de_la_imatge>.tar.xz
wget <url_de_la_imatge>.tar.xz.sha512
wget <url_de_la_imatge>.tar.xz.sig
```

Vaig verificar la integritat:

```bash
sha512sum -c <url_de_la_imatge>.tar.xz.sha512
```

I l'autenticitat, amb la clau GPG oficial de Libreboot:

```bash
wget https://libreboot.org/lbkey.asc
gpg --import lbkey.asc
gpg --verify <url_de_la_imatge>.tar.xz.sig <url_de_la_imatge>.tar.xz
```

Va confirmar `Good signature from "Leah Rowe <info@minifree.org>"`.

## Pas 6 — Injectar els fitxers de proveïdor (Intel ME) i la MAC address

Els fitxers de la imatge oficial porten el prefix `DO_NOT_FLASH_YET` fins que no s'hi
injecta el microprogramari de la Intel Management Engine i la MAC address de la
interfície de xarxa. Aquest pas és imprescindible: sense fer-ho, l'equip no arrenca
després de gravar-hi el microprogramari.

```bash
cd ~/lbmk
git config --global user.email "<el_teu_correu>"
git config --global user.name "<el_teu_nom>"
./mk inject <ruta_a_la_imatge>.tar.xz setmac <la_teva_mac>
```

Aquest pas compila diverses eines internes (coreboot, cbfstool, ifdtool, me_cleaner,
entre d'altres) i descarrega/processa el microprogramari de la ME automàticament. Vaig
topar amb dues aturades per dependències que em faltaven (`autoconf`/`automake`/
`libtool` per compilar `libarchive`, i `innoextract`/`python-is-python3` per processar
el microprogramari de la ME); un cop instal·lades, el procés va continuar sense
problemes. També calia tenir configurada una identitat de `git`
(`user.email`/`user.name`), ja que l'eina aplica pedaços interns amb `git am`.

Un cop acabat, l'arxiu `.tar.xz` original queda modificat amb tot ja injectat, i cal
extreure'l de nou per obtenir les ROMs finals:

```bash
tar xf <ruta_a_la_imatge>.tar.xz
```

El fitxer corresponent a la variant escollida ja no porta el prefix `DO_NOT_FLASH_YET`.

## Pas 7 — Gravar el microprogramari

```bash
sudo flashrom -p internal -w <ruta_a_la_rom_ja_injectada>.rom
```

> **Nota sobre un problema que vaig trobar:** en una primera prova vaig forçar el
> tipus de xip amb `-c MX25L3205D/MX25L3208D` (seguint una indicació genèrica que
> havia trobat), i `flashrom` no va poder trobar el xip (`No EEPROM/flash device
> found`). No es va arribar a escriure res. Repetint la comanda sense forçar cap tipus
> de xip, deixant que `flashrom` fes la detecció automàtica, el procés va funcionar
> correctament.

El procés va acabar amb `Verifying flash... VERIFIED.`

## Pas 8 — Primera arrencada

Apagat complet i engegada manual (mateix comportament que al pas 3). L'equip va
arrencar directament el sistema Debian que ja hi havia instal·lat, a través del GRUB
de Libreboot, sense cap pas addicional.

## Verificació posterior

```bash
sudo dmidecode -s bios-vendor
sudo dmidecode -s bios-version
sensors
```

Va confirmar `coreboot` versió `26.01rev1` com a microprogramari actiu, i temperatures
de CPU normals en repòs (entre 48-54°C). La interfície de xarxa Ethernet mantenia la
MAC address original, encara que amb un nom d'interfície diferent al que tenia amb la
BIOS Dell.

## Resultat

L'equip arrenca amb Libreboot 26.01rev1, confirmat amb `dmidecode -s bios-vendor` →
`coreboot`. No vaig necessitar cap procediment de recuperació en cap moment del
procés.

## Reconeixement

Aquest procés l'he fet amb l'ajuda de Claude (Anthropic), tant per investigar
prèviament la viabilitat de gravar el microprogramari des del propi sistema en aquest
model abans de començar, com per guiar-me pas a pas durant tota la instal·lació i
interpretar els missatges i errors que anaven sortint. Totes les comandes i decisions
finals les he executat jo directament al meu dispositiu.

He fet això com a exercici propi d'aprenentatge i ho publico per ajudar altres persones
que els pugui interessar fer aquest procés.

## Referències

- [Libreboot — pàgina oficial](https://libreboot.org)
- [Libreboot — instal·lació als Dell Latitude](https://libreboot.org/docs/install/latitude.html)
- [dell-flash-unlock — GitHub](https://github.com/nic3-14159/dell-flash-unlock)
- [lbmk — Codeberg](https://codeberg.org/libreboot/lbmk)
