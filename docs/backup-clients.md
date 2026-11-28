# Configuración de Clientes de Copia de Seguridad 💾🛡️

Esta guía detalla la configuración óptima para realizar copias de seguridad automáticas hacia Movistar / O2 Cloud mediante el proxy WebDAV con **Synology HyperBackup**, **Duplicati** y **rclone**.

---

## 📑 Índice de Contenidos

1. 🗄️ [Synology HyperBackup](#-1-synology-hyperbackup)
2. 📦 [Duplicati](#-2-duplicati)
3. ☁️ [Rclone (Sync y Backup)](#-3-rclone-sync-y-backup)
   * 🔏 [Configuración avanzada: archivos de más de 4 GB (`chunker`) y cifrado (`crypt`)](#-configuración-avanzada-archivos-de-más-de-4-gb-chunker-y-cifrado-crypt)

---

## 🗄️ 1. Synology Hyper Backup

[Hyper Backup](https://kb.synology.com/es-es/DSM/help/HyperBackup/BackupApp_desc?version=7) de Synology NAS es 100 % compatible con el proxy WebDAV de Movistar/O2 Cloud gracias a la implementación de las cuotas virtuales RFC 4331 y el soporte de repositorios `.hbk`.

### Pasos de configuración

1. En tu NAS Synology (DSM), abre **Hyper Backup**.
2. Pulsa en el icono **`+`** para [crear una nueva tarea de copia de seguridad de datos](https://kb.synology.com/es-es/DSM/help/HyperBackup/data_backup_create?version=7#b_5).
3. Selecciona el tipo de destino de copia de seguridad **Carpetas y paquetes** (para respaldar tanto los datos de los paquetes de DSM como las carpetas compartidas) y pulsa **Siguiente**.
4. En la lista de destinos, selecciona **WebDAV** dentro de la categoría *Servidor de archivos* y pulsa **Siguiente**.
5. Configura el tipo de tarea y los parámetros de conexión:
   * **Crear tarea de copia de seguridad:** para iniciar una copia completamente nueva.
   * **Volver a vincular a tarea existente:** para reutilizar o restaurar los datos de un repositorio `.hbk` que ya exista en la nube. Tendrás que introducir la clave de cifrado en caso de tenerlo.
   * **Dirección del servidor:**
     * **Contenedor en el mismo NAS (recomendado):** `http://127.0.0.1:5050` o `http://IP_LOCAL_DEL_NAS:5050`. Al no salir el tráfico WebDAV del propio NAS, usar HTTP local es seguro y evita sobrecarga o límites de *timeout* del proxy inverso.
     * **Servidor externo / remoto:** `https://dominio.com` (a través de [proxy inverso](configuration.md#-6-topologías-de-red-docker-y-proxies-inversos)).
   * **Nombre de usuario:** El usuario configurado en `WEBDAV_USER` (ej. `admin`).
   * **Contraseña:** La contraseña configurada en `WEBDAV_PASSWORD`.
   * **Carpeta:** Escribe o selecciona una carpeta dedicada (ej. `HyperBackup_NAS`).
   * **Directorio:** Nombre del repositorio `.hbk` que creará Hyper Backup (ej. escribe `NAS_Backup_1`, que se guardará como `NAS_Backup_1.hbk`).
   * Pulsa **Siguiente**.
6. Selecciona las carpetas compartidas de tu NAS que deseas incluir en la copia (opcionalmente puedes aplicar filtros de archivos de inclusión o exclusión por nombre o extensión) y pulsa **Siguiente**.
7. Elige las aplicaciones compatibles de DSM que quieras respaldar (ej. *Synology Drive Server*, *Synology Photos*, etc.) y pulsa **Siguiente**.
8. Configura las opciones generales de la tarea:
   * **Nombre de la tarea:** Escribe un nombre identificativo para la copia.
   * **Programación de copia de seguridad:** Habilita esta opción y elige los días (frecuencia) y la hora a la que deseas que se ejecute la copia de forma automática.
   * **Programación de verificación de integridad:** Habilita esta opción y selecciona las fechas y horas en las que DSM comprobará periódicamente que los datos del repositorio no estén dañados.
   * **Compresión:** Viene marcada por defecto para reducir el tamaño de los datos antes de subirlos.
   * **Cifrado del lado cliente (muy recomendado):** Marca la casilla **Habilitar cifrado del lado cliente**. Al igual que ocurre en otras nubes comerciales (como Google Drive o Dropbox), en Movistar/O2 Cloud el transporte va cifrado pero los archivos no están cifrados con una clave privada del usuario. Al activar esta opción, Hyper Backup cifra todos los datos **en el propio NAS antes de subirlos** utilizando una contraseña que solo tú conoces, evitando que el proveedor o terceros puedan leer tu información.
   * ⚠️ **Importante:** Introduce una contraseña única y compleja y guarda a buen recaudo (en diferentes ubicaciones) **tanto la contraseña como el archivo de clave `.pem`** que se descargará al finalizar; si los pierdes, **no podrás descifrar ni restaurar la copia de seguridad** bajo ningún concepto.
   * Pulsa **Siguiente**.
9. Ajusta la política de rotación de versiones según tus preferencias y pulsa **Siguiente**.
10. Revisa los datos en la pantalla de resumen y pulsa **Finalizar** para guardar la tarea. Cuando te pregunte si deseas realizar la copia de seguridad ahora, pulsa **Sí**.

> [!TIP]
> Mantén siempre una carpeta dedicada y exclusiva para Hyper Backup (ej. `/HyperBackup_NAS/`) para aislar sus índices y bloques de otras copias. **Nunca sincronices esa carpeta** si tienes instalado el cliente oficial de escritorio de Movistar/O2 Cloud, ya que podría alterar o bloquear los archivos internos del repositorio `.hbk`.


---

## 📦 2. Duplicati

Duplicati es una solución multiplataforma de copias de seguridad encriptadas e incrementales compatible con Windows, Linux, macOS y Docker.

### Pasos de Configuración:

1. Abre la interfaz web de **Duplicati**.
2. Crea una nueva tarea de copia de seguridad (*Añadir copia de seguridad* > *Configurar una nueva copia*).
3. Asigna un nombre a la copia y una contraseña de encriptación fuerte (AES-256).
4. En el paso **Destino de la copia**:
   * **Tipo de almacenamiento:** `WebDAV`.
   * **Servidor y puerto:** La IP de tu servidor (ej: `192.168.1.46`) y puerto `5050` (o el asignado a `8080` de Docker).
   * **Ruta en el servidor:** Nombre de la carpeta de destino en la nube (ejemplo: `Duplicati_Servidor` o `/Duplicati_Servidor/`).
   * **Conexión integrada (SSL):** Desactivado (a menos que uses proxy inverso HTTPS).
   * **Nombre de usuario:** El usuario definido en `WEBDAV_USER` (ej. `admin`).
   * **Contraseña:** La contraseña definida en `WEBDAV_PASSWORD`.
5. Pulsa el botón **Probar conexión**. Duplicati verificará el acceso e indicará que la conexión ha tenido éxito (si la carpeta no existía, preguntará si deseas crearla).
6. Selecciona los datos locales a respaldar y finaliza el asistente.

---

## ☁️ 3. Rclone (Sync y Backup)

Rclone permite sincronizar, copiar y respaldar directorios locales o de otros proveedores directamente contra Movistar Cloud.

### Configuración en `rclone.conf`:

```ini
[movistar]
type = webdav
url = http://TU_IP_SERVIDOR:5050
# owncloud envía X-OC-Mtime: se conserva la fecha de modificación original
vendor = owncloud
user = admin
pass = TuContraseñaOfuscadaConRcloneObscure
```

### Comandos útiles:

* **Listar carpetas:**
  ```bash
  rclone lsd movistar:
  ```
* **Copiar un directorio local a la nube:**
  ```bash
  rclone copy /ruta/a/mis/fotos movistar:Fotos --transfers 3 --progress
  ```
* **Sincronizar espejo:**
  ```bash
  rclone sync /ruta/datos movistar:Backup_Datos --transfers 3 --progress
  ```

> [!NOTE]
> Se recomienda limitar `--transfers 3` para alinearse con el semáforo interno del proxy y aprovechar el máximo ancho de banda sin saturar la API de Telefónica.

> [!TIP]
> **Archivos grandes y cortes de red (perfil `desktop`):** si la conexión con O2 se corta a mitad de una subida, el proxy la reanuda donde se quedó, como el cliente oficial. Pero O2 recibe las subidas reanudadas a solo 1–2 MB/s, y rclone da la subida por fallida si no recibe respuesta en 5 minutos (`--timeout`). En ese caso la reintenta entera, y el proxy retira lo que había subido a medias. Si subes archivos de varios GB con una conexión inestable, sube el límite, por ejemplo con `--timeout 30m`, para que la reanudación pueda terminar.

---

### 🔏 Configuración avanzada: archivos de más de 4 GB (chunker) y cifrado (crypt)

La API de Movistar / O2 Cloud rechaza cualquier archivo individual superior a **4 GB**. Para subir archivos de mayor tamaño hay que trocearlos con [`chunker`](https://rclone.org/chunker/), y opcionalmente cifrarlos con [`crypt`](https://rclone.org/crypt/). Esta configuración está probada con varios TB de datos:

```ini
[movistar]
type = webdav
url = http://TU_IP_SERVIDOR:5050
# owncloud envía X-OC-Mtime: se conserva la fecha de modificación original
vendor = owncloud
user = admin
pass = TuContraseñaOfuscadaConRcloneObscure

[movistar-chunk]
type = chunker
remote = movistar:datos
chunk_size = 1Gi
# Sin punto: el formato por defecto (*.rclone_chunk.###) cambia la extensión al
# renombrar, SAPI no lo permite (MED-1050) y el proxy tiene que volver a subir el trozo
name_format = *-chunk###
# Cada trozo se sube directamente con su nombre definitivo: sin MOVE ni 423 Locked
transactions = norename
hash_type = none

# Opcional: cifrado de contenido y nombres
[movistar-crypt]
type = crypt
remote = movistar-chunk:
```

* **Sin comentarios al final de la línea:** `rclone` no los reconoce y el comentario pasa a formar parte del valor (`name_format = *-chunk###  # …` deja un formato inválido). Ponlos siempre en una línea aparte, como arriba.
* **Úsalo en un remoto nuevo:** `name_format` y `transactions` cambian cómo se llaman los trozos en la nube; no los modifiques en un remoto que ya contenga datos subidos.
* **Nombres de hasta 255 caracteres:** Movistar Cloud recorta los más largos sin avisar y el proxy los rechaza con `400`. Con `crypt` (codificación `base32`) eso ocurre con nombres originales de unos 140 caracteres o más; acórtalos o usa `filename_encoding = base64` en un remoto `crypt` nuevo.
