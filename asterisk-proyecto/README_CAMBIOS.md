# Cambios realizados para el funcionamiento del proyecto Asterisk

Este documento describe los cambios y configuraciones clave que se realizaron para lograr que el proyecto funcione correctamente. Se abordan los principales problemas encontrados y cómo se resolvieron.

## 1. Agregado de `modules.conf`

**Problema:**
El archivo `modules.conf` no estaba presente en la configuración original. Sin este archivo, Asterisk no puede cargar los módulos necesarios para operar, especialmente los relacionados con PJSIP (el stack SIP moderno de Asterisk).

**Solución:**
Se creó el archivo `asterisk/conf/modules.conf` con la siguiente configuración:

```ini
[modules]
autoload=yes

; Módulos esenciales para PJSIP
load => res_pjsip.so
load => res_pjsip_session.so
load => res_pjsip_registrar.so
load => res_pjsip_authenticator_digest.so
load => chan_pjsip.so
load => res_pjsip_endpoint_identifier_user.so
load => res_pjsip_outbound_registration.so
load => res_pjsip_nat.so
load => res_pjsip_transport_websocket.so

; Módulos core
load => pbx_config.so
load => chan_bridge_media.so
load => res_musiconhold.so
```

**Importancia:**
Este archivo indica a Asterisk qué módulos cargar al iniciar. Sin los módulos de PJSIP, no es posible registrar ni autenticar extensiones SIP modernas.

## 2. Agregado de `asterisk.conf`

**Problema:**
No existía el archivo `asterisk.conf`, lo que provocaba que el contenedor no generara logs y mostrara errores indicando que no encontraba este archivo.

**Solución:**
Se creó el archivo `asterisk/conf/asterisk.conf` con las rutas necesarias para los directorios de configuración, módulos, logs, etc.

```ini
[directories]
astetcdir => /etc/asterisk
astmoddir => /usr/lib/asterisk/modules
astvarlibdir => /var/lib/asterisk
astdbdir => /var/lib/asterisk
astkeydir => /var/lib/asterisk
astdatadir => /var/lib/asterisk
astagidir => /var/lib/asterisk/agi-bin
astspooldir => /var/spool/asterisk
astrundir => /var/run/asterisk
astlogdir => /var/log/asterisk

[options]
;verbose = 3
;debug = 3
```

**Importancia:**
Este archivo define las rutas de los directorios que Asterisk utiliza. Sin él, el sistema no puede ubicar sus archivos de configuración ni generar logs, dificultando la depuración.

## 3. Configuración de `pjsip.conf` para autenticación

**Problema:**
La configuración original de `pjsip.conf` no era adecuada para la autenticación de usuarios. Al intentar hacer login desde un cliente SIP, se recibía el error 401 (Unauthorized), aunque el puerto 5060 estaba correctamente expuesto en la máquina host.

**Solución:**
Se reescribió el archivo `asterisk/conf/pjsip.conf` para definir endpoints, autenticaciones y aors correctamente. Se aseguraron los siguientes puntos:
- Cada extensión tiene su propio bloque de autenticación (`auth`), endpoint y aor.
- Las credenciales de usuario y contraseña son correctas y coinciden con las configuraciones del cliente SIP.
- Se especifica el contexto adecuado (`from-internal`).
- Se definen los códecs permitidos y se habilita la autenticación por usuario.

**Ejemplo de configuración:**
```ini
; Transporte UDP
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; Extensión 100
[100]
type=endpoint
context=from-internal
disallow=all
allow=ulaw
allow=h264
allow=vp8
allow=vp9
auth=auth100
aors=100
identify_by=auth_username

[100]
type=aor
max_contacts=5
remove_existing=yes

[auth100]
type=auth
auth_type=userpass
username=100
password=clave100
realm=asterisk

; Extensión 101
[101]
type=endpoint
context=from-internal
disallow=all
allow=ulaw
allow=h264
allow=vp8
allow=vp9
auth=auth101
aors=101
identify_by=auth_username

[101]
type=aor
max_contacts=5
remove_existing=yes

[auth101]
type=auth
auth_type=userpass
username=101
password=clave101
realm=asterisk
```

**Importancia:**
Esta configuración permite que los usuarios SIP se autentiquen correctamente contra el servidor Asterisk, resolviendo el error 401 y permitiendo el registro y uso de las extensiones.

## 4. Ajustes en `docker-compose.yml`

Se modificó el archivo para montar los directorios de configuración, datos, spool y logs de Asterisk desde la estructura local, asegurando la persistencia y correcta lectura de los archivos.

---

**Resumen:**
- Se agregaron los archivos `modules.conf` y `asterisk.conf` esenciales para el funcionamiento y depuración de Asterisk.
- Se corrigió la configuración de `pjsip.conf` para permitir la autenticación de usuarios SIP.
- Se ajustó el `docker-compose.yml` para montar los directorios necesarios.

Con estos cambios, el sistema Asterisk funciona correctamente y permite el registro y autenticación de extensiones SIP.