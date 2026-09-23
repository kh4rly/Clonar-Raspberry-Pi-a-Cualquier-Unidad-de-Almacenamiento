[Clonar Raspberry Pi a Cualquier Unidad de Almacenamiento.md](https://github.com/user-attachments/files/32571021/Clonar.Raspberry.Pi.a.Cualquier.Unidad.de.Almacenamiento.md)
# 🍓 Guía Universal: Clonar Raspberry Pi a Cualquier Unidad de Almacenamiento

Guía paso a paso para migrar todo el sistema (SO, archivos, bots, configuraciones) desde la tarjeta SD a **cualquier unidad de almacenamiento** (SSD NVMe, SSD SATA, disco USB, otra SD, etc.) sin perder absolutamente nada.

El proceso es **universal**: solo cambia el nombre del dispositivo destino. Todo lo demás es idéntico.

---

## 📋 Requisitos Previos

- Raspberry Pi 5 (probado en modelo de 8 GB; funciona también en Pi 4).
- Unidad de destino correctamente conectada (SSD NVMe en HAT M.2, SSD SATA por USB, disco externo, etc.).
- Sistema operativo Raspberry Pi OS funcionando desde la tarjeta SD.
- Conexión a internet.
- Acceso a terminal (usuario con permisos `sudo`).

---

## 🎯 Lo Que Es Universal y Lo Que Cambia

### ✅ Universal (siempre igual)

- Instalación de `rpi-clone` con el mismo comando.
- Flags `-f -v` para forzar clonado y ver progreso.
- Respuestas a las preguntas del proceso (`yes`, `yes`, `yes`, Enter, Enter).
- Configuración del arranque con `sudo raspi-config` → **Advanced Options** → **Boot Order**.
- Verificación final con `lsblk` y `ls ~`.

### 🔄 Lo Que Cambia Según la Unidad

**Solo el nombre del dispositivo destino.** Nada más.

| Tipo de unidad | Nombre típico | Ejemplo de comando |
| :--- | :--- | :--- |
| **SSD NVMe (M.2 PCIe)** | `/dev/nvme0n1` | `sudo rpi-clone -f -v /dev/nvme0n1` |
| **SSD/HDD SATA por USB** | `/dev/sda` | `sudo rpi-clone -f -v /dev/sda` |
| **Disco USB (segundo)** | `/dev/sdb` | `sudo rpi-clone -f -v /dev/sdb` |
| **Otra tarjeta SD** | `/dev/mmcblk1` | `sudo rpi-clone -f -v /dev/mmcblk1` |
| **Adaptador NVMe por USB** | `/dev/sda` o `/dev/nvme0n1` (según adaptador) | Verificar con `lsblk` |

> **Regla de oro**: **Siempre ejecuta `lsblk` antes de clonar** para identificar exactamente cómo se llama la unidad que quieres usar como destino.

---

## 🛠️ Fase 1: Preparación del Sistema

### 1.1 — Actualizar el sistema

Es importante tener todo actualizado antes de clonar para evitar conflictos.

```bash
sudo apt update && sudo apt full-upgrade -y
```

### 1.2 — Instalar `rpi-clone`

Se usa la versión mantenida por **Jeff Geerling**, que es la que mejor funciona en Raspberry Pi 4 y 5.

```bash
curl https://raw.githubusercontent.com/geerlingguy/rpi-clone/master/install | sudo bash
```

### 1.3 — Identificar la unidad de destino

```bash
lsblk
```

Localiza tu unidad de destino. Ejemplos según el tipo:

**SSD NVMe:**
```
nvme0n1     259:0    0 931,5G  0 disk 
├─nvme0n1p1 259:1    0    16M  0 part 
└─nvme0n1p2 259:2    0 931,5G  0 part 
```

**SSD SATA por USB:**
```
sda           8:0    0 465,8G  0 disk 
├─sda1        8:1    0   512M  0 part 
└─sda2        8:2    0 465,3G  0 part 
```

**Otra SD:**
```
mmcblk1     179:0    0  53,7G  0 disk 
├─mmcblk1p1 179:1    0   512M  0 part 
└─mmcblk1p2 179:2    0  53,2G  0 part 
```

Apunta el nombre que corresponda a tu unidad. Se usará en todos los comandos siguientes.

---

## 🧹 Fase 2: Limpiar la Unidad de Destino (si venía de otro sistema)

Si la unidad se usó antes en Windows, Linux, macOS u otro sistema, conviene borrar sus firmas de sistema de archivos para evitar conflictos.

### 2.1 — Borrar firmas

Sustituye `NOMBRE_UNIDAD` por el que identificaste en la Fase 1.

```bash
# Ejemplo para SSD NVMe:
sudo wipefs -a /dev/nvme0n1

# Ejemplo para SSD SATA por USB:
sudo wipefs -a /dev/sda

# Ejemplo para otra SD:
sudo wipefs -a /dev/mmcblk1
```

### 2.2 — Verificar

```bash
lsblk
```

Ahora la unidad debería aparecer **sin particiones hijas** (`p1`, `p2`).

> **Nota**: Si la unidad ya está limpia (nunca se ha usado o ya se hizo wipefs antes), puedes saltarte esta fase. `rpi-clone` con `-f` borrará de todas formas cualquier resto.

---

## 💾 Fase 3: Clonar la SD a la Unidad de Destino

### 3.1 — Ejecutar la clonación

Sustituye `NOMBRE_UNIDAD` por el nombre real de tu unidad.

```bash
# Ejemplo para SSD NVMe:
sudo rpi-clone -f -v /dev/nvme0n1

# Ejemplo para SSD SATA por USB:
sudo rpi-clone -f -v /dev/sda

# Ejemplo para otra SD:
sudo rpi-clone -f -v /dev/mmcblk1
```

- `-f` → fuerza clonado completo desde cero.
- `-v` → muestra progreso detallado.

> **Nota**: Algunas versiones de `rpi-clone` esperan el nombre **sin** `/dev/` delante. Si te da error de "Cannot find...", prueba sin el prefijo:
> ```bash
> sudo rpi-clone -f -v nvme0n1
> ```

### 3.2 — Preguntas del proceso y respuestas

| Pregunta | Respuesta |
| :--- | :--- |
| `Do you want to continue? (yes/no)` | `yes` |
| `Do you want to change the UUIDs? (yes/no)` | `yes` |
| `Do you want to resize the filesystem? (yes/no)` | `yes` |
| `Optional destination ext type file system label (16 chars max)` | Pulsar `Enter` (dejar vacío) |
| `Hit Enter when ready to unmount...` | Pulsar `Enter` |

> **Importante**:
> - Cambiar los UUIDs evita conflictos entre la SD y la unidad destino al arrancar.
> - Aceptar el resize expande la partición raíz para ocupar **todo el espacio** de la unidad destino, no solo el tamaño de la SD.

### 3.3 — Tiempo estimado

- **SSD NVMe rápido**: 3-5 minutos.
- **SSD SATA por USB**: 5-15 minutos.
- **Disco HDD por USB**: 15-40 minutos.
- **Otra SD**: 10-20 minutos.

El tiempo varía según la cantidad de datos y la velocidad de la unidad.

---

## ⚙️ Fase 4: Configurar el Arranque desde la Unidad de Destino

### 4.1 — Abrir configuración

```bash
sudo raspi-config
```

### 4.2 — Navegar por el menú

1. **Advanced Options** (Opciones Avanzadas)
2. **Boot Order** (Orden de arranque)
3. Seleccionar **NVMe/USB Boot** (sirve para NVMe, USB y SATA por USB)
4. **Finish** (Finalizar)
5. Cuando pregunte si reiniciar → elegir **`No`**

> **Nota**: La opción se llama "NVMe/USB Boot" pero cubre también SSD SATA conectados por USB. Para el caso de clonar a otra tarjeta SD (menos común), la Pi 5 siempre intentará arrancar primero desde la SD insertada, así que simplemente retirando la SD original arrancará desde la nueva si está en el slot.

---

## 🔌 Fase 5: Apagar y Arrancar desde la Nueva Unidad

### 5.1 — Apagar

```bash
sudo shutdown now
```

### 5.2 — Cambio físico

1. Esperar a que la Pi se apague por completo (LED verde deja de parpadear).
2. **Retirar la tarjeta SD** de su ranura (salvo que el destino sea la SD original, en cuyo caso no aplica).
3. **Encender** la Raspberry Pi.

> La primera vez puede tardar unos segundos más de lo normal. Es normal.

---

## ✅ Fase 6: Verificación

### 6.1 — Confirmar el dispositivo de arranque

```bash
lsblk
```

Salida esperada (ejemplo para SSD NVMe):

```
nvme0n1     259:0    0 931,5G  0 disk 
├─nvme0n1p1 259:1    0   512M  0 part /boot/firmware
└─nvme0n1p2 259:2    0   931G  0 part /
```

Salida esperada (ejemplo para SSD SATA por USB):

```
sda           8:0    0 465,8G  0 disk 
├─sda1        8:1    0   512M  0 part /boot/firmware
└─sda2        8:2    0 465,3G  0 part /
```

Fíjate en los **MOUNTPOINTS**: `/` y `/boot/firmware` deben apuntar a la nueva unidad, no a la SD (`mmcblk0`).

### 6.2 — Confirmar que los archivos están intactos

```bash
ls ~
```

Deberías ver exactamente las mismas carpetas y archivos que tenías en la SD.

### 6.3 — Confirmar que tus servicios funcionan

```bash
systemctl --user status    # si los tienes como servicios de usuario
# o
sudo systemctl status      # si los tienes como servicios del sistema
```

---

## 🚀 Ajuste Opcional: Forzar PCIe Gen 3.0 (solo para NVMe en HAT M.2)

Por defecto, la Raspberry Pi 5 usa PCIe Gen 2.0 para el SSD NVMe. Puedes forzar Gen 3.0 para obtener el doble de velocidad (10 GT/s).

> **Este ajuste NO aplica a unidades USB o SATA.** Solo a NVMe conectado directamente por PCIe.

### Editar la configuración

```bash
sudo nano /boot/firmware/config.txt
```

Añadir al final:

```
dtparam=pciex1_gen=3
```

Guardar (`Ctrl+O`, `Enter`) y salir (`Ctrl+X`). Reiniciar:

```bash
sudo reboot
```

> **Nota**: Si notas inestabilidad (reinicios, errores de disco), comenta la línea añadiendo `#` al principio y reinicia.

---

## 💡 Consejos y Notas Útiles

- **Copia de seguridad**: Guarda la SD original sin borrarla durante unos días. Si algo falla, puedes volver a arrancar desde ella.
- **SSD con disipador**: Si tu SSD ya viene con disipador (como muchos NVMe modernos), no necesitas nada más. Ayuda a evitar el *thermal throttling*.
- **Durabilidad del SSD**: Un SSD NVMe de 1 TB soporta cientos de TB escritos (600-1200 TBW típico). Para uso de bots 24/7, la duración es de **décadas**.
- **Log2ram**: Ya no es necesario si arrancas desde SSD, pero sigue siendo útil si en algún momento vuelves a la SD.
- **Actualizaciones del sistema**: Puedes seguir usando `sudo apt update && sudo apt upgrade` con normalidad.

---

## 🔧 Solución de Problemas

### La Pi no arranca desde la nueva unidad tras sacar la SD

1. Vuelve a insertar la SD y enciende.
2. Verifica el Boot Order:
   ```bash
   sudo raspi-config
   ```
   → **Advanced Options** → **Boot Order** → **NVMe/USB Boot**
3. Actualiza la EEPROM del firmware:
   ```bash
   sudo rpi-eeprom-update -a
   sudo reboot
   ```

### El SSD NVMe no es detectado por el sistema

1. Comprueba que el HAT M.2 esté bien conectado al puerto PCIe.
2. Verifica que el cable plano del HAT esté bien insertado (sin forzar).
3. Comprueba que el sistema ve el dispositivo:
   ```bash
   lspci
   ```
4. Si no aparece, revisa el archivo `/boot/firmware/config.txt` para asegurarte de que el PCIe está habilitado.

### La unidad USB no arranca (problema conocido en Pi 5)

Algunos adaptadores USB-SATA/NVMe tienen incompatibilidades con el arranque en Raspberry Pi 5.

**Adaptadores que funcionan bien**: chips **ASMedia** y **JMicron**.

**Solución si no arranca**: añadir un `quirk` en `cmdline.txt`.

1. Edita el archivo:
   ```bash
   sudo nano /boot/firmware/cmdline.txt
   ```
2. Añade al final (todo en la misma línea):
   ```
   usb-storage.quirks=XXXX:XXXX:u
   ```
   Sustituye `XXXX:XXXX` por el ID de tu adaptador. Lo puedes encontrar con:
   ```bash
   lsusb
   ```
3. Guarda y reinicia.

### Error "Cannot find 'NOMBRE' in the partition table"

- Verifica el nombre exacto de la unidad con `lsblk`.
- Asegúrate de escribir el nombre completo correcto (por ejemplo `nvme0n1`, no `nvme0n2`).
- Algunas versiones de `rpi-clone` esperan el nombre sin `/dev/` delante. Prueba:
  ```bash
  sudo rpi-clone -f -v nvme0n1
  ```

### La unidad de destino es más pequeña que la SD actual

`rpi-clone` no encoge particiones automáticamente. Si los datos no caben en el destino, abortará el proceso. Para solucionarlo:

1. Redimensiona las particiones de la SD antes de clonar con `gparted` (desde entorno gráfico).
2. O clona primero a una unidad igual o mayor, y luego ajusta con `gparted` desde la unidad clonada.

---

## 📝 Resumen Universal (para copiar/pegar)

Sustituye `NOMBRE_UNIDAD` por el nombre real de tu unidad (ej: `nvme0n1`, `sda`, `mmcblk1`).

```bash
sudo apt update && sudo apt full-upgrade -y && \
curl https://raw.githubusercontent.com/geerlingguy/rpi-clone/master/install | sudo bash && \
lsblk && \
sudo wipefs -a /dev/NOMBRE_UNIDAD && \
sudo rpi-clone -f -v /dev/NOMBRE_UNIDAD && \
sudo raspi-config && \
sudo shutdown now
```

---

## 📌 Ejemplos Rápidos por Tipo de Unidad

### SSD NVMe (M.2 PCIe)
```bash
sudo wipefs -a /dev/nvme0n1
sudo rpi-clone -f -v /dev/nvme0n1
```

### SSD/HDD SATA por USB
```bash
sudo wipefs -a /dev/sda
sudo rpi-clone -f -v /dev/sda
```

### Otra tarjeta SD
```bash
sudo wipefs -a /dev/mmcblk1
sudo rpi-clone -f -v /dev/mmcblk1
```

---

**Autor**: Downdragon  
**Fecha**: 2026  
**Hardware base**: Raspberry Pi 5 (8 GB) + SSD NVMe M.2 1 TB  
**Sistema**: Raspberry Pi OS (64-bit)  
**Compatibilidad**: Universal (NVMe, SATA, USB, SD)
