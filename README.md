# Adquirir un equipo comprometido

**Alumno:** Luis Baena
**Módulo:** Análisis Forense Informático · UD1
**Caso:** Almadraba Diseño · equipo PC-ADMIN (FORENSE-06, Windows 7 x64) · ALMADRABA-2026-01

## Contenido

- [Acta de adquisición](acta-adquisicion.pdf) (firmada)
- [Registro de cadena de custodia](registro-custodia.pdf) (una ficha por evidencia, firmado por los dos)
- [Hashes y registros de las herramientas](hashes/)
- [Notas de trabajo](notas-de-trabajo.md)
- [Capturas](capturas/)

## Antes de empezar · Marca

Marca `AFI-LuisBaena-20261030`, escrita con el Bloc de notas y guardada en el escritorio como `marca-luisbaena.txt`. El 06/11 se copió al pendrive encontrado. Captura: [02_marca-luisbaena.png](capturas/02_marca-luisbaena.png)

> La marca se escribió con la fecha del día anterior (20261030) por error; la adquisición se realizó el 31/10/2026. No se corrige en el equipo porque forma parte de la evidencia.



## Parte 1 · Adquisición en vivo 

### 1. Estado inicial

- Captura de la pantalla: [01_Estado_Inicial.png](capturas/01_Estado_Inicial.png) · al iniciar sesión: [00_Escritorio.png](capturas/00_Escritorio.png) (aviso de reinicio pendiente)
- Hora del sistema: 09:17:31,54 · hora real (equipo del analista): 09:17:37 · zona horaria del equipo: Romance Standard Time (UTC+1)
- Desfase: el equipo va unos 5,5 s atrasado (unos 9 s respecto a la hora oficial del ROA)

### 2. Memoria RAM

- **EV01** · `PC-ADMIN-ram.raw` (1.073.676.288 bytes) · WinPmem `winpmem_mini_x64_rc2`
- SHA-256: `871ce15b1b64bdfb6084d2100f5e20e82416296a40541c9e18915382e0779653` ([EV01-ram.sha256](hashes/EV01-ram.sha256))
- DumpIt 3.6 no funciona en este Windows 7 (error `0xC0000428`: el sistema no acepta su driver firmado con SHA-2). Se usó WinPmem, incluido en el USB de respuesta.
- Marca encontrada en la RAM (`strings -u` + `findstr`): [04_marca_en_RAM.png](capturas/04_marca_en_RAM.png)

### 3. Datos volátiles

- **EV02** · `respuesta-en-vivo.ps1`, en copia adaptada a PowerShell 2.0 (el original no se modificó) · [EV02-volatiles.sha256](hashes/EV02-volatiles.sha256) · registro: [EV02-respuesta-en-vivo-registro.txt](hashes/EV02-respuesta-en-vivo-registro.txt)
- **EV02b** · los módulos que fallaron por PowerShell 2.0 se completaron con comandos nativos (`recogida-win7.bat`) · [EV02b-volatiles-win7.sha256](hashes/EV02b-volatiles-win7.sha256) · registro: [EV02b-recogida-win7-registro.txt](hashes/EV02b-recogida-win7-registro.txt)

### 4. Tráfico de red

- **EV03** · `red.pcap` (14.641 bytes, unos 6 minutos), capturado desde fuera con `VBoxManage controlvm "FORENSIC_10" nictrace1 on/off`
- SHA-256: `0ae6934dc360cc094b8090bfdc0c6b14f9e4800def7cfc7686c886d0ef147ea2` ([EV03-red.sha256](hashes/EV03-red.sha256))

### 5. Triaje

- **EV04** · 313 ficheros, entre ellos `$MFT`, las colmenas del registro, los registros de eventos, Prefetch y las tareas · `triaje-win7.bat` + RawCopy64 v1.0.0.22 · [EV04-triaje.sha256](hashes/EV04-triaje.sha256) · registro: [EV04-triaje-win7-registro.txt](hashes/EV04-triaje-win7-registro.txt)
- KAPE falla al iniciar porque el equipo solo tiene .NET 4.0. En el USB solo había el instalador de Velociraptor, no un recolector offline.
- Marca encontrada en `$MFT` (`strings -u` + `findstr`): [05_marca_en_MFT.png](capturas/05_marca_en_MFT.png)

### 6. Cifrado

`manage-bde -status`: C: *Fully Decrypted*, *Protection Off*, sin protectores. Sin rastro de TrueCrypt ni VeraCrypt en procesos y servicios. No hay volúmenes cifrados, así que no hace falta recuperar claves ni hacer una copia lógica antes de apagar.
   Captura: [Cifrado](capturas/03_Cifrado.png)

### 7. Apagado

#### ¿Cómo se ha apagado el equipo?

El equipo se apagó por corte de corriente, con la opción "Apagar la máquina" de VirtualBox, que equivale a desenchufarlo. Antes se expulsó de forma segura el USB de respuesta, para que todo lo escrito en él quedara guardado. Además, se dejó desmarcada la casilla de restaurar la instantánea, para no deshacer el estado del disco ni borrar la marca.

#### ¿Por qué se ha elegido este método?

La alternativa era un apagado ordenado desde Windows, pero se descartó por varios motivos. Al iniciar sesión apareció, sin que nadie lo pidiera, el aviso "You must restart your computer to apply these changes": había un reinicio pendiente, y un apagado ordenado habría aplicado esos cambios y ejecutado lo que estuviera programado para el cierre, posiblemente obra del atacante, alterando el disco. Además, un apagado ordenado escribe en el disco al cerrar programas, vaciar cachés y guardar registros, mientras que el corte de corriente lo deja exactamente como estaba. Lo que se pierde al cortar la corriente, como la memoria, los procesos, las conexiones y el tráfico de red, ya se había adquirido en los pasos anteriores. Tampoco había volúmenes cifrados que pudieran quedar bloqueados al apagar, así que no hacía falta recuperar claves ni hacer una copia lógica antes.

#### ¿Qué riesgo se asume?

El único riesgo es que quede alguna inconsistencia leve en el sistema de ficheros, por datos que estaban en caché y no llegaron a escribirse. Esto no impide hacer después la imagen forense del disco.

## Parte 2 · Adquisición post-mortem

### Disco

- La máquina usa un disco VDI con una instantánea, así que el estado del disco está repartido entre el disco base y el diferencial. `VBoxManage clonemedium --format RAW` los une en una imagen RAW de 34.359.738.368 bytes.
- Hashes de los ficheros de la VM **antes y después: iguales** ([hashes-vm-antes.csv](hashes/hashes-vm-antes.csv), [hashes-vm-despues.csv](hashes/hashes-vm-despues.csv)). El original no se modificó.
- **EV05** · `PC-ADMIN-disco.E01` · FTK Imager 8.3.0.27, con verificación · MD5 `2d61e1880d37a2f05c878b378c53f502` · SHA1 `7acba6a3007d28292ac0574d15cab6daabc0f24d` · **verified**
- SHA-256 del fichero E01: `93d9d2be149e6e308685a1d17edfc4e72bc27ed2d516e128642c0862138afe66` · registro: [EV05-disco-ftk.E01.txt](hashes/EV05-disco-ftk.E01.txt) · captura: [10_ftk_verificacion_disco.png](capturas/10_ftk_verificacion_disco.png)

### Pendrive encontrado

- Sin marcas externas: [11_pendrive_foto.jpg](capturas/11_pendrive_foto.jpg). Según el sistema: Sony Storage Media, 7,5 GB, n.º de serie 2A080922H6837.
- Bloqueo de escritura con `bloqueo-usb.ps1` (política `WriteProtect`), **comprobado antes de la imagen**: [12_bloqueo_protegido.png](capturas/12_bloqueo_protegido.png) · registro: [bloqueo-usb.log](hashes/bloqueo-usb.log)
- **EV06** · `pendrive-encontrado.E01` · FTK Imager 8.3.0.27 (unidad física), con verificación · MD5 `deffac3913d254624312e9b105bae016` · SHA1 `89fa203a0d59b66ec169ac76b8d287a993c66afd` · **verified**
- SHA-256 del fichero E01: `0a10b259264e58cd3a26291af10a7e42dec528219a3ba5ab95ffe308fcbcb393` · registro: [EV06-pendrive-ftk.E01.txt](hashes/EV06-pendrive-ftk.E01.txt) · captura: [13_ftk_verificacion_pendrive.png](capturas/13_ftk_verificacion_pendrive.png)

## Parte 3 · Busca tu marca desde Linux

Desde CAINE 14, con las evidencias en una carpeta compartida **en solo lectura** y las imágenes E01 montadas con `ewfmount` y `mount -o ro,noexec`. Script: `buscar-marca.sh` · registro: [parte3-busca-marca.log](hashes/parte3-busca-marca.log)

| Evidencia | Dónde aparece la marca | Captura |
|---|---|---|
| EV01 · RAM | `AFI-LuisBaena-20261030`, 4 veces (`strings -el`) | [14](capturas/14_marca_RAM_linux.png) |
| EV04 · triaje | `marca-luisbaena.txt` y `.lnk` en `$MFT` | [15](capturas/15_marca_MFT_linux.png) |
| EV05 · disco | `Users/Administrador/Desktop/marca-luisbaena.txt` | [16](capturas/16_marca_imagen_disco.png) |
| EV06 · pendrive | `/marca-luisbaena.txt` | [17](capturas/17_marca_imagen_pendrive.png) |

El fichero de la marca tiene el mismo SHA-256 en el disco y en el pendrive: `682107edfa7bc8dcae1e6b57932ec222db478b103203f276f34a228ca36c4886`.

## Parte 4 · Cadena de custodia

- Copias de trabajo con el mismo hash que las maestras.
- Carpeta de entrega (9,9 GB) con [hashes.sha256](hashes/hashes.sha256): 376 ficheros, comprobado con `sha256sum -c` → **376 OK**. Captura: [18_hashes_entrega.png](capturas/18_hashes_entrega.png)
- SHA-256 de `hashes.sha256`: `c5e23adceb441da87428b02ad4c6cc99c116bc6955e37454d5dd5ab5d644ff98`
- Entrega a Raúl Toledo Procopio (07/11, 13:00): comprueba el hash del listado y `sha256sum -c hashes.sha256` → **376 OK**. Devolución a las 13:25, comprobada de nuevo → 376 OK. Firmado por los dos en el [registro de custodia](registro-custodia.pdf).

## Hashes

- Entrega: [hashes.sha256](hashes/hashes.sha256) (376 ficheros, se comprueba con `sha256sum -c`)
- Parte 1: [hashes-evidencias.sha256](hashes/hashes-evidencias.sha256) (368 ficheros)
- Hashes de las herramientas usadas: [herramientas-usadas.sha256](hashes/herramientas-usadas.sha256)
- Verificación de las herramientas del USB: [verificacion-herramientas-usb.txt](hashes/verificacion-herramientas-usb.txt)

## Incidencias

- **Windows Defender** del equipo del analista puso en cuarentena tres `.vbs` maliciosos del triaje (`Trojan:VBS/Obfuse.NFE!MTB`, `TrojanDropper:VBS/Ploty.A`). Se restauraron y sus hashes coinciden.
- Escrituras en el equipo además de la marca: drivers temporales de WinPmem y DumpIt, el informe de error de KAPE y la conexión del USB.
- La instantánea de VirtualBox se hizo antes del primer arranque, sin conocer el aviso del enunciado. Por eso el disco se unió en RAW antes de hacer la imagen.
- El pendrive se conectó sin bloqueo para copiar la marca. Antes de la imagen se activó el bloqueo, se comprobó y se volvió a conectar.

> Las evidencias no se suben al repositorio por su tamaño; aquí solo se publican sus hashes y los registros de las herramientas.