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
- 14:20 · El hash del `.ova` coincide. 14:22 · Lo reimporto. 14:26 · Pongo **USB 2.0**, antes del primer arranque.

**31/10/2026**

- 08:45 · Hago una instantánea de la VM apagada ("Instalacion No hay arranque"). ⚠️ **Error:** el enunciado dice que no se hagan instantáneas, porque los cambios posteriores quedan fuera del `.vmdk`. No la borro ni la restauro.

## Arranque y marca

- 08:58 · Arranco la VM. 08:59 · Inicio sesión como `Administrador`.
- 08:59 · Aparece sola la ventana **"You must restart your computer to apply these changes"**. En el escritorio hay dos ficheros `crea_user`, que no abro. Captura desde VirtualBox: [00_Escritorio.png](capturas/00_Escritorio.png)
- 09:00 · Cierro la ventana con la X, sin reiniciar: un reinicio borraría la memoria.
- 09:05–09:07 · Marca: escribo `AFI-LuisBaena-20261030` en el Bloc de notas y la guardo en el escritorio como `marca-luisbaena.txt`. Captura: [02_marca-luisbaena.png](capturas/02_marca-luisbaena.png)
  - ⚠️ La fecha es la del día anterior, por error.
  - ⚠️ Cerré el Bloc de notas en lugar de dejarlo abierto sin guardar.
  - ⚠️ No la copié al pendrive encontrado: ese paso se añadió al enunciado después.
- 09:11 · Conecto el USB (Dispositivos → USB) → `D:`.

## 1. Estado inicial

- 09:17 · En el equipo: `echo %date% %time% & tzutil /g` → `09:17:31,54`, zona `Romance Standard Time` (UTC+1). En el equipo del analista: `09:17:37`.
- **Desfase:** el equipo va unos 5,5 s atrasado (unos 9 s respecto al ROA).
- Captura: [01_Estado_Inicial.png](capturas/01_Estado_Inicial.png)

## 2. Memoria RAM (EV01) · DumpIt → WinPmem

- 09:24 · `echo %PROCESSOR_ARCHITECTURE%` → `AMD64`, así que uso la versión x64.
- 09:34–09:56 · DumpIt y WinPmem fallan: el volumen del USB está marcado como "sucio" y el `cmd` no está elevado.
- 10:00 · En el equipo del analista: `chkdsk D: /f`, sin errores.
- 10:05 · Reconecto el USB y abro un `cmd` **como administrador**.
- 10:08 · `"D:\WINDOWS\memoria\Magnet DumpIt\x64\DumpIt.exe" /OUTPUT D:\salida\PC-ADMIN-ram.dmp` → **`Load driver failed with 0xC0000428`**. [06_dumpit_error.png](capturas/06_dumpit_error.png)
- 10:14–10:16 · `D:\WINDOWS\memoria\WinPmem\winpmem_mini_x64_rc2.exe D:\salida\PC-ADMIN-ram.raw` → 1.073.676.288 bytes. [07_winpmem_fin.png](capturas/07_winpmem_fin.png)
- 10:30 · `certutil -hashfile D:\salida\PC-ADMIN-ram.raw SHA256` → `871ce15b1b64bdfb6084d2100f5e20e82416296a40541c9e18915382e0779653`. Coincide con el calculado después en el equipo del analista.

**Por qué WinPmem:** DumpIt usa un driver firmado con SHA-2 que Windows 7 rechaza (`0xC0000428`). WinPmem hace lo mismo, ya estaba verificado en el USB y su driver sí carga en este sistema.

**Marca en la RAM:** aparece 4 veces (comprobado con `strings64.exe -u … | findstr "AFI-LuisBaena"` sobre la copia). [04_marca_en_RAM.png](capturas/04_marca_en_RAM.png)

## 3. Datos volátiles (EV02 y EV02b)

- 10:35 · `$PSVersionTable.PSVersion` → **PowerShell 2.0**.
- 10:38 · `respuesta-en-vivo.ps1` no arranca: usa `-notin`, que no existe hasta PowerShell 3.0.
- 11:04 · Hago una copia en el USB (`D:\respuesta-en-vivo-ps2.ps1`) y cambio solo la línea 88 por `'hashes-sha256.csv', '00-registro.txt' -notcontains $_.Name`. El original no se toca.
- 11:11 · `powershell -ExecutionPolicy Bypass -File D:\respuesta-en-vivo-ps2.ps1 -Caso ALMADRABA-2026-01 -Evidencia EV02-volatiles -Destino D:\salida` → 22 ficheros. Fallan 14 módulos por cmdlets de PowerShell 3.0 o posterior. ([registro](hashes/EV02-respuesta-en-vivo-registro.txt))
- 11:30–11:31 · Completo los módulos que fallaron con `recogida-win7.bat`, que pasé al USB desde el equipo del analista: `netstat -anob`, `tasklist /v`, `wmic process`, `sc queryex`, `wmic startup`, `reg query …\Run`, `net user`, `net localgroup administrators`, `systeminfo`, etc. → 31 ficheros. ([registro](hashes/EV02b-recogida-win7-registro.txt))

**Por qué:** el script es la herramienta que pide el enunciado, así que lo adapté lo mínimo. Lo que no pudo recoger lo saqué con comandos nativos de Windows, que no instalan nada. El portapapeles no se pudo recoger.

## 4. Tráfico de red (EV03)

En el equipo del analista:

```powershell
VBoxManage controlvm "FORENSIC_10" nictracefile1 "C:\Users\lbaen\Desktop\caso\red.pcap"
VBoxManage controlvm "FORENSIC_10" nictrace1 on     # 11:35
VBoxManage controlvm "FORENSIC_10" nictrace1 off    # 11:42
```

- `red.pcap` · 14.641 bytes · unos 6 minutos. Ya había tráfico en el primer segundo.
- SHA-256: `0ae6934dc360cc094b8090bfdc0c6b14f9e4800def7cfc7686c886d0ef147ea2`

**Por qué:** se captura desde fuera de la máquina, como un TAP, así que no se instala ni se ejecuta nada en el equipo.

## 5. Triaje (EV04) · KAPE → comandos nativos + RawCopy

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

## 6. Cifrado

- 12:35 · `manage-bde -status` → **C:** *Fully Decrypted*, *Protection Off*, sin protectores.
- 12:39 · `findstr /i "truecrypt veracrypt"` sobre la lista de procesos y servicios recogida → sin resultados.
- Captura: [03_Cifrado.png](capturas/03_Cifrado.png)

**Qué hice:** nada, porque no hay ningún volumen cifrado. No hace falta sacar la clave ni hacer una copia lógica antes de apagar.

## 7. Apagado

- 12:45 · Expulso el USB de forma segura dentro del equipo.
- 12:47 · VirtualBox → Cerrar → **Apagar la máquina**, sin restaurar la instantánea. [09_apagado.png](capturas/09_apagado.png)

**Por qué corte de corriente y no apagado ordenado:**

- Había un reinicio pendiente que nadie había pedido. Un apagado ordenado lo aplicaría y ejecutaría lo programado para el cierre, que podría ser del atacante.
- El apagado ordenado escribe en el disco; el corte de corriente lo deja tal como estaba.
- Lo volátil ya estaba adquirido y no hay cifrado.
- Riesgo asumido: una inconsistencia leve del sistema de ficheros, que no impide hacer la imagen.

## 8. Hashes y copia de seguridad

- 12:50–13:00 · En el equipo del analista calculo el hash de los 368 ficheros de evidencia ([hashes-evidencias.sha256](hashes/hashes-evidencias.sha256)) y hago una copia verificada en `Desktop\caso\evidencias`: **368 correctos, 0 errores**.
- 12:52 · Windows Defender pone en cuarentena tres `.vbs` del triaje (`HoVgcPUXNBk.vbs`, `jnQGgUrmj.vbs`, `JOEvfoml.vbs`), detectados como `Trojan:VBS/Obfuse.NFE!MTB` y `TrojanDropper:VBS/Ploty.A`. Excluyo las carpetas de evidencias del antivirus y los restauro con `MpCmdRun -Restore`. Los hashes coinciden.

## Incidencias

- USB no reconocido con los controladores 3.0 y 1.1 → reimportación y USB 2.0.
- Instantánea hecha antes de conocer el aviso del enunciado.
- Marca: fecha equivocada, Bloc de notas cerrado y sin copia al pendrive.
- Volumen del USB "sucio" → `chkdsk` en el equipo del analista. No anoté el resultado de la segunda verificación de herramientas.
- DumpIt, `respuesta-en-vivo.ps1` y KAPE no son compatibles con este Windows 7 → WinPmem, copia adaptada y comandos nativos con RawCopy.
- Escrituras en el equipo además de la marca: drivers temporales de WinPmem y DumpIt, el informe de error de KAPE y la conexión del USB.
- Defender puso en cuarentena las evidencias `.vbs` → se restauraron con su hash intacto.
