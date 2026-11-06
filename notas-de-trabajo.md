# Notas de trabajo · Adquirir un equipo comprometido

**Investigador:** Luis Baena · **Caso:** ALMADRABA-2026-01 · **Equipo:** PC-ADMIN (VM `FORENSIC_10`, Windows 7 x64, nombre `FORENSE-06`, 1 GB RAM)
**USB de respuesta:** `LB16` (exFAT, serie `F417-A13D`), letra `D:` en el equipo

Horas en hora local de España (UTC+1), al minuto, según el reloj del equipo del analista. Salen de lo que anoté en el momento, de los registros de las herramientas (script, `recogida-win7.bat`, triaje, WinPmem) o, para el resto, de la hora de las capturas tomadas en cada paso.

## 0. Preparación

**27/10/2026**

- 22:17 · Importo `FORENSIC_10.ova` en VirtualBox con la política de MAC "incluir todas" (conservar la MAC original) y sin convertir el disco a VDI (dejarlo en su formato original). No la arranco.

**30/10/2026**

- 13:30 · Comparo el reloj del equipo del analista con hora.roa.es: va unos **4 s atrasado**. Lo uso como referencia horaria.
- 13:39 · Verifico las herramientas del USB contra `hashes-windows.txt`: **1417 coinciden, 0 fallan** ([verificacion-herramientas-usb.txt](hashes/verificacion-herramientas-usb.txt)).
- 13:42 · Vacío `D:\salida`: muevo al equipo del analista los restos de pruebas anteriores.
- 13:45 · Hash del `.ova`: `DAF0EF5255D98276A6912A53611DB5C0CBF2CCCBB49A180DD7FCC0F95E14930C`.
- 13:50–14:18 · Pruebas: Windows 7 no reconoce el USB con los controladores USB 3.0 ni 1.1 de VirtualBox. Elimino esa máquina.
- 14:20 · El hash del `.ova` coincide. 14:22 · Lo reimporto; esta vez el disco queda convertido a VDI (`FORENSIC_10_disk0.vdi`). 14:26 · Pongo **USB 2.0**, antes del primer arranque.

**31/10/2026**

- 08:45 · Hago una instantánea de la VM apagada ("Instalacion No hay arranque"). **Error:** el enunciado dice que no se hagan instantáneas, porque los cambios posteriores quedan en un disco diferencial, fuera del disco base. No la borro ni la restauro (ver Parte 2).

## Parte 1 · Adquisición en vivo (31/10/2026)

### Arranque y marca

- 08:58 · Arranco la VM. 08:59 · Inicio sesión como `Administrador`.
- 08:59 · Aparece sola la ventana **"You must restart your computer to apply these changes"**. En el escritorio hay dos ficheros `crea_user`, que no abro. Captura desde VirtualBox: [00_Escritorio.png](capturas/00_Escritorio.png)
- 09:00 · Cierro la ventana con la X, sin reiniciar: un reinicio borraría la memoria.
- 09:05–09:07 · Marca: escribo `AFI-LuisBaena-20261030` en el Bloc de notas y la guardo en el escritorio como `marca-luisbaena.txt` (fecha de modificación del fichero: 09:07:50). Captura: [02_marca-luisbaena.png](capturas/02_marca-luisbaena.png)
  -  La fecha es la del día anterior, por error.
  -  Cerré el Bloc de notas en lugar de dejarlo abierto sin guardar.
  -  No la copié al pendrive encontrado: ese paso se añadió al enunciado después. Se copió el 06/11, antes de adquirir el pendrive (ver Parte 2).
- 09:11 · Conecto el USB (Dispositivos → USB) → `D:`.

### 1. Estado inicial

- 09:17 · En el equipo: `echo %date% %time% & tzutil /g` → `09:17:31,54`, zona `Romance Standard Time` (UTC+1). En el equipo del analista: `09:17:37`.
- **Desfase:** el equipo va unos 5,5 s atrasado (unos 9 s respecto al ROA).
- Captura: [01_Estado_Inicial.png](capturas/01_Estado_Inicial.png)

### 2. Memoria RAM (EV01) · DumpIt → WinPmem

- 09:24 · `echo %PROCESSOR_ARCHITECTURE%` → `AMD64`, así que uso la versión x64.
- 09:34–09:56 · DumpIt y WinPmem fallan: el volumen del USB está marcado como "sucio" y el `cmd` no está elevado.
- 10:00 · En el equipo del analista: `chkdsk D: /f`, sin errores.
- 10:05 · Reconecto el USB y abro un `cmd` **como administrador**.
- 10:08 · `"D:\WINDOWS\memoria\Magnet DumpIt\x64\DumpIt.exe" /OUTPUT D:\salida\PC-ADMIN-ram.dmp` → **`Load driver failed with 0xC0000428`**. [06_dumpit_error.png](capturas/06_dumpit_error.png)
- 10:14–10:16 · `D:\WINDOWS\memoria\WinPmem\winpmem_mini_x64_rc2.exe D:\salida\PC-ADMIN-ram.raw` → 1.073.676.288 bytes. [07_winpmem_fin.png](capturas/07_winpmem_fin.png)
- 10:30 · `certutil -hashfile D:\salida\PC-ADMIN-ram.raw SHA256` → `871ce15b1b64bdfb6084d2100f5e20e82416296a40541c9e18915382e0779653`. Coincide con el calculado después en el equipo del analista.

**Por qué WinPmem:** DumpIt usa un driver firmado con SHA-2 que Windows 7 rechaza (`0xC0000428`). WinPmem hace lo mismo, ya estaba verificado en el USB y su driver sí carga en este sistema.

**Marca en la RAM:** aparece 4 veces (comprobado con `strings64.exe -u … | findstr "AFI-LuisBaena"` sobre la copia). [04_marca_en_RAM.png](capturas/04_marca_en_RAM.png)

### 3. Datos volátiles (EV02 y EV02b)

- 10:35 · `$PSVersionTable.PSVersion` → **PowerShell 2.0**.
- 10:38 · `respuesta-en-vivo.ps1` no arranca: usa `-notin`, que no existe hasta PowerShell 3.0.
- 11:04 · Hago una copia en el USB (`D:\respuesta-en-vivo-ps2.ps1`) y cambio solo la línea 88 por `'hashes-sha256.csv', '00-registro.txt' -notcontains $_.Name`. El original no se toca.
- 11:11 · `powershell -ExecutionPolicy Bypass -File D:\respuesta-en-vivo-ps2.ps1 -Caso ALMADRABA-2026-01 -Evidencia EV02-volatiles -Destino D:\salida` → 22 ficheros. Fallan 14 módulos por cmdlets de PowerShell 3.0 o posterior. ([registro](hashes/EV02-respuesta-en-vivo-registro.txt))
- 11:30–11:31 · Completo los módulos que fallaron con `recogida-win7.bat`, que pasé al USB desde el equipo del analista: `netstat -anob`, `tasklist /v`, `wmic process`, `sc queryex`, `wmic startup`, `reg query …\Run`, `net user`, `net localgroup administrators`, `systeminfo`, etc. → 31 ficheros. ([registro](hashes/EV02b-recogida-win7-registro.txt))

**Por qué:** el script es la herramienta que pide el enunciado, así que lo adapté lo mínimo. Lo que no pudo recoger lo saqué con comandos nativos de Windows, que no instalan nada. El portapapeles no se pudo recoger.

### 4. Tráfico de red (EV03)

En el equipo del analista:

```powershell
VBoxManage controlvm "FORENSIC_10" nictracefile1 "C:\Users\lbaen\Desktop\caso\red.pcap"
VBoxManage controlvm "FORENSIC_10" nictrace1 on     # 11:35
VBoxManage controlvm "FORENSIC_10" nictrace1 off    # 11:42
```

- `red.pcap` · 14.641 bytes · unos 6 minutos. Ya había tráfico en el primer segundo.
- SHA-256: `0ae6934dc360cc094b8090bfdc0c6b14f9e4800def7cfc7686c886d0ef147ea2`

**Por qué:** se captura desde fuera de la máquina, como un TAP, así que no se instala ni se ejecuta nada en el equipo.

### 5. Triaje (EV04) · KAPE → comandos nativos + RawCopy

- 11:44 · `dir C:\Windows\Microsoft.NET\Framework64` → solo hay .NET 4.0 (la carpeta tiene fecha del arranque de hoy).
- 11:45 · En el USB solo está el **instalador** `.msi` de Velociraptor, no un recolector offline.
- 11:49 · `kape.exe --tsource C: --tdest D:\salida\ALMADRABA-2026-01_EV04-triaje --target KapeTriage` → `System.NotSupportedException` y se cierra. [08_kape_error.png](capturas/08_kape_error.png)
- 11:59–12:32 · `triaje-win7.bat` (pasado al USB desde el equipo del analista):
  - `RawCopy64.exe /FileNamePath:C:0 /OutputPath:…\mft` → `$MFT`
  - `reg save` de SYSTEM, SOFTWARE, SAM, SECURITY y HKCU
  - `wevtutil epl` de los registros de eventos
  - `xcopy` de Prefetch, Tasks, Startup, Temp, Recent, el escritorio y `hosts`
  - → 313 ficheros. ([registro](hashes/EV04-triaje-win7-registro.txt))
- 12:04–12:28 · El listado `dir C:\ /s` va muy lento y lo interrumpo con `Ctrl+C`. `$MFT` ya contiene esos datos.

**Por qué:** KAPE necesita .NET 4.5 o superior, e instalar Velociraptor crearía un servicio en el equipo. RawCopy es portable, no necesita .NET y copia `$MFT`, que Windows no deja copiar.

**Marca en el triaje:** en `$MFT` aparecen `marca-luisbaena.txt` y `marca-luisbaena.lnk`. [05_marca_en_MFT.png](capturas/05_marca_en_MFT.png)

### 6. Cifrado

- 12:35 · `manage-bde -status` → **C:** *Fully Decrypted*, *Protection Off*, sin protectores.
- 12:39 · `findstr /i "truecrypt veracrypt"` sobre la lista de procesos y servicios recogida → sin resultados.
- Captura: [03_Cifrado.png](capturas/03_Cifrado.png)

**Qué hice:** nada, porque no hay ningún volumen cifrado. No hace falta sacar la clave ni hacer una copia lógica antes de apagar.

### 7. Apagado

- 12:45 · Expulso el USB de forma segura dentro del equipo.
- 12:47 · VirtualBox → Cerrar → **Apagar la máquina**, sin restaurar la instantánea. [09_apagado.png](capturas/09_apagado.png)

**Por qué corte de corriente y no apagado ordenado:**

- Había un reinicio pendiente que nadie había pedido. Un apagado ordenado lo aplicaría y ejecutaría lo programado para el cierre, que podría ser del atacante.
- El apagado ordenado escribe en el disco; el corte de corriente lo deja tal como estaba.
- Lo volátil ya estaba adquirido y no hay cifrado.
- Riesgo asumido: una inconsistencia leve del sistema de ficheros, que no impide hacer la imagen.

### 8. Hashes y copia de seguridad

- 12:50–13:00 · En el equipo del analista calculo el hash de los 368 ficheros de evidencia ([hashes-evidencias.sha256](hashes/hashes-evidencias.sha256)) y hago una copia verificada en `Desktop\caso\evidencias`: **368 correctos, 0 errores**.
- 12:52 · Windows Defender pone en cuarentena tres `.vbs` del triaje (`HoVgcPUXNBk.vbs`, `jnQGgUrmj.vbs`, `JOEvfoml.vbs`), detectados como `Trojan:VBS/Obfuse.NFE!MTB` y `TrojanDropper:VBS/Ploty.A`. Excluyo las carpetas de evidencias del antivirus y los restauro con `MpCmdRun -Restore`. Los hashes coinciden.

## Parte 2 · Adquisición post-mortem (06/11/2026)

### Disco de la máquina

**Documentación de la máquina (07:48):**

- Estado: *powered off* desde el 31/10/2026 a las 12:47:56 (11:47:56 UTC), que coincide con el corte de corriente.
- Carpeta: `C:\Users\lbaen\VirtualBox VMs\FORENSIC_10`.
- Disco base: `FORENSIC_10_disk0.vdi` (VDI, 32.768 MB). Al reimportar se convirtió a VDI, no se quedó en `.vmdk`.
- Instantánea: "Instalacion No hay arranque". La máquina usa el disco diferencial `Snapshots\{f8dd2a95-a32b-49a7-aca6-5643062bacab}.vdi` (1,16 GB), hijo del disco base. Ahí están todos los cambios de la sesión en vivo, incluida la marca.
- Comandos: `VBoxManage showvminfo "FORENSIC_10"`, `VBoxManage snapshot "FORENSIC_10" list --details`, `VBoxManage list hdds`.

**Hashes de la carpeta antes de la adquisición (07:50–07:51):**

| Fichero | SHA-256 |
|---|---|
| `FORENSIC_10_disk0.vdi` | `7430c40df5d0d68f90d8c5c1567864dcf9f4b9365b2f1f75b7aa4a4b50fabf3c` |
| `Snapshots\{f8dd2a95-…}.vdi` | `c3652a3fe1d50994fabd1d7782c108f4905097f7262ce110fafc52b9d97f5887` |
| `FORENSIC_10.vbox` | `f649f8d97ed9b400d6d32e637a0be4a8cf31094e55bc37fddcd0e861b2227c17` |
| `FORENSIC_10.vbox-prev` | `704b1888a20a5d57e32d39171d8b5066e850bea35e4fa1b8d00794fdb3789779` |
| `Logs\VBox.log` | `ad87d1319759b55b3f1ce24c6de8ddaf360065917c36b0fea8aabdef04c16605` |
| `Logs\VBoxHardening.log` | `8fd94621391115c57abb9e766915e7c26e7ca352bd1de84b55baf583ae4a17e7` |

**Imagen RAW del estado del disco (07:58–07:59):**

`VBoxManage clonemedium disk "…\Snapshots\{f8dd2a95-a32b-49a7-aca6-5643062bacab}.vdi" "…\caso\parte2\PC-ADMIN-disco.raw" --format RAW` → `Clone medium created in format 'RAW'`. Tamaño: 34.359.738.368 bytes (32.768 MB, la capacidad del disco). Después, `VBoxManage closemedium disk` para quitar la RAW del registro de VirtualBox sin borrar el fichero.

**Por qué:** FTK Imager no admite discos `.vdi` ni cadenas con disco diferencial. El estado del disco al apagar está repartido entre `FORENSIC_10_disk0.vdi` y el diferencial de la instantánea. `clonemedium` sobre el diferencial recorre la cadena y saca el disco completo, leyendo los originales sin modificarlos (se comprueba con su hash antes y después). `qemu-img` no sirve porque no entiende los diferenciales de VirtualBox.

**Hashes de la carpeta después de la imagen (08:01):** los 6 ficheros, incluidos `FORENSIC_10_disk0.vdi` y el diferencial, tienen **el mismo hash que antes** (`Compare-Object` → IGUAL en todos). El original no se ha modificado. Ficheros: `hashes-vm-antes.csv` y `hashes-vm-despues.csv`.

**SHA-256 de la RAW (`PC-ADMIN-disco.raw`, 08:01–08:03):** `8aee6f5436e4da5201f511dc7fbcf81c1ab241a5e188637ea5e62cbabc410327`

**Imagen E01 con FTK Imager 8.3.0.27 (08:26–08:31):**

*File → Create Disk Image → Image File* → `PC-ADMIN-disco.raw` → **E01**. Case Number `ALMADRABA-2026-01`, Evidence Number `EV05`, Examiner `Luis Baena`, fragment size 0 (un solo fichero), compression 6, sin cifrado y **Verify images after they are created** marcado. Destino: carpeta vacía `caso\parte2\E01-disco`. La primera vez lo apunté a la misma carpeta que la RAW, FTK avisó de que podía sobrescribir un fichero y cambié de carpeta.

- Adquisición: 08:26:32 – 08:28:49 · Verificación: 08:28:49 – 08:31:15.
- `PC-ADMIN-disco.E01`: 5.005.541.505 bytes · 67.108.864 sectores de 512 bytes (32.768 MB).
- MD5 `2d61e1880d37a2f05c878b378c53f502` y SHA1 `7acba6a3007d28292ac0574d15cab6daabc0f24d`: **verified** en el hash calculado, el guardado en la imagen y el del informe.
- SHA-256 del fichero `PC-ADMIN-disco.E01` (para la copia de trabajo y la entrega): `93d9d2be149e6e308685a1d17edfc4e72bc27ed2d516e128642c0862138afe66`
- Registro: [EV05-disco-ftk.E01.txt](hashes/EV05-disco-ftk.E01.txt) · Captura: [10_ftk_verificacion_disco.png](capturas/10_ftk_verificacion_disco.png)

**Por qué E01 con FTK Imager:** es el formato forense que propone el enunciado. Guarda dentro de la imagen los metadatos del caso y los hashes, va comprimido y FTK lo verifica él solo al terminar.

**Configuración y registros (08:33):** copiados a `caso\parte2\config-vm`: `FORENSIC_10.vbox`, `FORENSIC_10.vbox-prev`, `Logs\VBox.log` y `Logs\VBoxHardening.log`. Los 4 tienen el mismo hash que en los originales.

### Pendrive encontrado

**Marca en el pendrive (08:40):** copio `marca-luisbaena.txt` a mi pendrive, que hace de pendrive encontrado:

`Copy-Item "…\caso\evidencias\ALMADRABA-2026-01_EV04-triaje-win7\escritorio-Administrador\marca-luisbaena.txt" E:\` → contenido `AFI-LuisBaena-20261030`. El fichero conserva su fecha de modificación original, 31/10/2026 09:07:50, la de la marca en el equipo.

**Por qué ahora:** el enunciado pide copiar la marca al pendrive antes de empezar, pero ese paso se añadió después de la adquisición en vivo (31/10). Lo hago antes de tratar el pendrive como evidencia, es decir, antes de fotografiarlo y de hacer su imagen, usando la copia del fichero del triaje para no volver a arrancar el equipo. Después lo expulso de forma segura y lo desconecto: desde aquí no se vuelve a conectar sin el bloqueo de escritura activo.

**Descripción del pendrive (10:11), antes de conectarlo:** pendrive USB-A negro, de unos 6,5 cm de largo, **sin marca, modelo, capacidad ni número de serie visibles** en el exterior. Foto sobre plancha milimetrada: [11_pendrive_foto.jpg](capturas/11_pendrive_foto.jpg). Los datos del dispositivo se obtienen ya con el bloqueo activo.

**Bloqueo de escritura con `bloqueo-usb.ps1` (10:15–10:19):**

- **Error de orden:** conecté el pendrive **antes** de activar el bloqueo. Al conectarlo, Windows lo montó (E:) con permiso de escritura durante unos segundos. No escribí nada en él, pero el sistema pudo hacerlo.
- 10:15:23 · Activo la política global (`WriteProtect=1`) y, a las 10:15:34, marco el disco como solo lectura: `Disco 1 (Sony Storage Media, serie 2A080922H6837) -> SOLO LECTURA`.
- 10:16–10:17 · Pulso **Comprobar** → `no deja escribir -> PROTEGIDO`.
- 10:19:25 · Para corregirlo, lo desconecto y lo vuelvo a conectar con la política ya activa (hora de llegada del dispositivo según Windows, `DEVPKEY_Device_LastArrivalDate`). 10:19:57 · **Comprobar** → PROTEGIDO. La imagen se hace desde esta conexión. Captura: [12_bloqueo_protegido.png](capturas/12_bloqueo_protegido.png)
- Datos del pendrive (según el sistema): **Sony Storage Media**, **7,5 GB**, n.º de serie **2A080922H6837**.
- SHA-256 de `bloqueo-usb.ps1` (ejecutado desde `Desktop\bloqueo-usb.ps1`): `150adbc3bee0ee8f8662bee8758e4791fb090c20a48807ac24ae9bea531eef5a` (coincide con el de la preparación). Registro: [bloqueo-usb.log](hashes/bloqueo-usb.log); las dos primeras líneas (03/11, 08:55) son una prueba del script anterior al caso.

**Por qué el bloqueo:** el pendrive es una evidencia. Sin bloqueo, Windows puede escribir en él al conectarlo (metadatos, indexación, papelera…). El script impide escribir en cualquier USB que se conecte (política `WriteProtect`), y **Comprobar** confirma que no se puede escribir antes de hacer la imagen.

**Imagen E01 con FTK Imager 8.3.0.27 (10:33–10:41):**

*File → Create Disk Image → Physical Drive* → `Sony Storage Media USB Device` (7.648 MB) → **E01**. Case Number `ALMADRABA-2026-01`, Evidence Number `EV06`, Examiner `Luis Baena`, fragment size 0, compression 6, sin cifrado y **Verify images after they are created** marcado. Destino: `caso\parte2\E01-pendrive`.

- Adquisición: 10:33:30 – 10:41:00 · Verificación: 10:41:00 – 10:41:21.
- Origen según FTK: modelo `Sony Storage Media USB Device`, n.º de serie `2A080922H6837`, interfaz USB, extraíble, 15.663.104 sectores de 512 bytes (7.648 MB).
- `pendrive-encontrado.E01`: 4.318.806.712 bytes.
- MD5 `deffac3913d254624312e9b105bae016` y SHA1 `89fa203a0d59b66ec169ac76b8d287a993c66afd`: **verified** en los tres hashes; *Match*; sin sectores defectuosos.
- SHA-256 del fichero `pendrive-encontrado.E01`: `0a10b259264e58cd3a26291af10a7e42dec528219a3ba5ab95ffe308fcbcb393`
- Registro: [EV06-pendrive-ftk.E01.txt](hashes/EV06-pendrive-ftk.E01.txt) · Captura: [13_ftk_verificacion_pendrive.png](capturas/13_ftk_verificacion_pendrive.png)

**Cierre (10:46:50):** desactivo la política de bloqueo (`WriteProtect=0`), con la imagen ya terminada y verificada (10:41:21).

**Observación:** el pendrive es de uso personal y la imagen incluye otras carpetas además de la marca (`DOCUMENTOS`, `MUSICA`, `VIDEOS`…).

## Parte 3 · Busca tu marca (06/11/2026)

Desde la VM **CAINE 14** (Linux), con la carpeta `caso` compartida **en solo lectura** (`VBoxManage sharedfolder add … --readonly`; en CAINE, `sudo mount -t vboxsf -o ro caso /mnt/caso`). Las búsquedas las hace el script `buscar-marca.sh` (SHA-256 `f34c707a729d87300d11b3a8531c173779e8dda2e99355ef869ee45f958fda1f`), que guarda su registro en una segunda carpeta compartida con escritura: [parte3-busca-marca.log](hashes/parte3-busca-marca.log).

**Por qué un script:** en la VM no se puede pegar texto desde Windows, y así cada comando queda escrito tal cual en el registro.

| Hora | Evidencia | Comando | Resultado |
|---|---|---|---|
| 12:11 | EV01 · RAM | `strings -el PC-ADMIN-ram.raw \| grep "AFI-"` | `AFI-LuisBaena-20261030` × 4 · [14](capturas/14_marca_RAM_linux.png) |
| 12:16 | EV04 · triaje | `strings -el '$MFT' \| grep "marca-"` | `marca-luisbaena.txt`, `marca-luisbaena.lnk` y `…\Desktop\marca-luisbaena.txt` · [15](capturas/15_marca_MFT_linux.png) |
| 12:17 | EV05 · imagen del disco | `ewfmount` → `mmls` (partición NTFS en el sector 206848) → `mount -t ntfs-3g -o ro,noexec,loop,offset=105906176` | `Users/Administrador/Desktop/marca-luisbaena.txt` → `AFI-LuisBaena-20261030` · [16](capturas/16_marca_imagen_disco.png) |
| 12:19 | EV06 · imagen del pendrive | `ewfmount` → `mmls` (GPT, partición en el sector 2048) → `mount -t ntfs-3g -o ro,noexec,loop,offset=1048576` | `/marca-luisbaena.txt` (volumen NTFS `LB8GB`) → `AFI-LuisBaena-20261030` · [17](capturas/17_marca_imagen_pendrive.png) |

- SHA-256 del fichero de la marca, **igual en el disco y en el pendrive**: `682107edfa7bc8dcae1e6b57932ec222db478b103203f276f34a228ca36c4886`.
- Las imágenes se montan **en solo lectura** (`ro`) y sin ejecución (`noexec`); al terminar, el script las desmonta.

**Incidencia:** la primera ejecución del script (11:59–12:05) falló en dos búsquedas: bash interpretaba `$MFT` como variable, y no detectaba la partición GPT del pendrive. Las corregí y volví a lanzarlo; los resultados y las capturas son de la segunda ejecución.

## Incidencias

- USB no reconocido con los controladores 3.0 y 1.1 → reimportación y USB 2.0.
- Instantánea hecha antes de conocer el aviso del enunciado.
- Marca: fecha equivocada, Bloc de notas cerrado y copia al pendrive hecha el 06/11 en lugar de antes de empezar.
- Volumen del USB "sucio" → `chkdsk` en el equipo del analista. No anoté el resultado de la segunda verificación de herramientas.
- DumpIt, `respuesta-en-vivo.ps1` y KAPE no son compatibles con este Windows 7 → WinPmem, copia adaptada y comandos nativos con RawCopy.
- Escrituras en el equipo además de la marca: drivers temporales de WinPmem y DumpIt, el informe de error de KAPE y la conexión del USB.
- Defender puso en cuarentena las evidencias `.vbs` → se restauraron con su hash intacto.
- Parte 2: el disco era `.vdi` con instantánea → imagen del estado completo con `VBoxManage clonemedium` y E01 con FTK Imager; hashes del original iguales antes y después.
- Parte 2: conecté el pendrive antes de activar el bloqueo de escritura → lo reconecté con el bloqueo activo y comprobado antes de hacer la imagen.