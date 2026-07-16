### bind9conf

### bind9conf — Configuración básica de un servidor DNS autoritativo con BIND9

Ejemplo completo y comentado de configuración de **BIND9** (`named`) en Debian/Ubuntu para levantar un **servidor DNS autoritativo** de una zona propia, con su correspondiente **zona inversa** (PTR), partiendo de una instalación limpia del paquete.

Pensado como plantilla de referencia rápida para laboratorios, VMs de pruebas o como punto de partida documentado antes de adaptarlo a un entorno de producción.

> Proyecto de [hackingyseguridad.com](http://www.hackingyseguridad.com)

---

### Tabla de contenidos

1. [¿Qué es BIND9 y qué hace esta configuración?](#-qué-es-bind9-y-qué-hace-esta-configuración)
2. [Ficheros incluidos](#-ficheros-incluidos)
3. [Arquitectura de la configuración de BIND9](#-arquitectura-de-la-configuración-de-bind9)
4. [Escenario de este ejemplo](#-escenario-de-este-ejemplo)
5. [Requisitos](#-requisitos)
6. [Guía paso a paso](#-guía-paso-a-paso)
7. [Entendiendo el fichero de zona (SOA y registros)](#-entendiendo-el-fichero-de-zona-soa-y-registros)
8. [named.conf.options explicado](#-namedconfoptions-explicado)
9. [Verificación y pruebas](#-verificación-y-pruebas)
10. [Solución de problemas](#-solución-de-problemas)
11. [ Aviso sobre los ficheros de ejemplo del repositorio](#️-aviso-sobre-los-ficheros-de-ejemplo-del-repositorio)
12. [Buenas prácticas de seguridad](#-buenas-prácticas-de-seguridad)
13. [Licencia](#-licencia)

---

### ¿Qué es BIND9 y qué hace esta configuración?

**BIND** (*Berkeley Internet Name Domain*) es el servidor DNS de código abierto más utilizado del mundo, mantenido por ISC. El demonio que lo implementa se llama `named`.

Este repositorio documenta cómo configurar BIND9 para que actúe como:

- **Servidor autoritativo** de un dominio propio (por ejemplo `hackingyseguridad.es`): responde con autoridad a las consultas de ese dominio, sin necesidad de reenviarlas a otro servidor.
- **Resolver recursivo**: puede resolver también cualquier otro dominio de Internet consultando en cascada a los servidores raíz.
- **Servidor de zona inversa**: traduce direcciones IP a nombres de host (registros PTR), útil para verificación de correo, auditorías y *logging*.

```
                 ┌─────────────────────────────┐
                 │        Cliente DNS          │
                 └──────────────┬──────────────┘
                                │ consulta "hackingyseguridad.es"
                                ▼
                 ┌─────────────────────────────┐
                 │   BIND9 (named) — 192.168.1.252 │
                 │                              │
                 │  ┌────────────┐ ┌──────────┐ │
                 │  │ Zona directa│ │Zona inversa│
                 │  │  db.hacking-│ │  db.192   │ │
                 │  │  yseguridad │ │           │ │
                 │  └────────────┘ └──────────┘ │
                 └─────────────────────────────┘
```

---

### Ficheros incluidos

| Fichero | Rol | ¿Se edita en este tutorial? |
|---|---|---|
| `named.conf` | Fichero principal; incluye (`include`) al resto de ficheros de configuración | No |
| `named.conf.options` | Opciones globales del servidor: directorio de trabajo, forwarders, `allow-query`, IPv6... | ✅ Sí |
| `named.conf.local` | Declaración de las **zonas locales propias** (directa e inversa) | ✅ Sí |
| `named.conf.default-zones` | Zonas por defecto de BIND (`.`, `localhost`, `127.in-addr.arpa`...) | No |
| `zones.rfc1918` | Zonas inversas para todos los rangos de IP privadas (RFC 1918), listas para incluir si se necesitan | No (opcional, vía `include`) |
| `db.local` | Fichero de zona plantilla para `localhost` | No |
| `db.127` | Fichero de zona plantilla para la zona inversa de loopback (`127.in-addr.arpa`) | Se copia como base |
| `db.0` / `db.255` | Ficheros de zona plantilla para direcciones de red/broadcast | No |
| `db.empty` | Plantilla vacía de zona, usada como base para crear zonas nuevas | Se copia como base |
| `db.hackingyseguridad` | Fichero de zona **directa** del dominio de ejemplo (creado a partir de `db.empty`) | ✅ Sí |
| `bind.keys` | Claves de confianza (*trust anchors*) usadas por DNSSEC/`managed-keys` | No |

> El fichero de la **zona inversa** (llamado `db.192` en la guía, siguiendo la convención de nombrarlo por el último octeto de la red) no viene incluido con ese nombre en el repositorio; se crea en el propio proceso a partir de `db.127` (ver [paso 4](#-guía-paso-a-paso) y el [aviso](#️-aviso-sobre-los-ficheros-de-ejemplo-del-repositorio) más abajo).

---

### Arquitectura de la configuración de BIND9

BIND9 divide su configuración en varios ficheros que se incluyen entre sí, en lugar de tener un único fichero monolítico. Esto facilita separar "lo que viene de fábrica" de "lo que es específico de tu servidor":

```
named.conf
   ├── include "named.conf.options"        → opciones globales del servidor
   ├── include "named.conf.local"          → TUS zonas (directa + inversa)
   └── include "named.conf.default-zones"  → zonas de sistema (localhost, root hints...)
```

| Concepto | Explicación |
|---|---|
| **Zona directa (forward zone)** | Traduce nombre → IP (ej. `hackingyseguridad.es` → `192.168.1.252`) |
| **Zona inversa (reverse zone)** | Traduce IP → nombre (ej. `192.168.1.252` → `server.hackingyseguridad.es`), usando el dominio especial `in-addr.arpa` |
| **`type master`** | Este servidor es la fuente autoritativa de la zona (no una copia/`slave`) |
| **`file`** | Ruta al fichero de zona donde están los registros (A, NS, PTR, etc.) |

---

### Escenario de este ejemplo

| Parámetro | Valor de ejemplo |
|---|---|
| Dominio | `hackingyseguridad.es` |
| IP del servidor DNS | `192.168.1.252` (red privada RFC 1918) |
| Zona inversa | `252.1.168.192.in-addr.arpa` |
| Hostname del servidor | `server.hackingyseguridad.es` |
| DNS "upstream" para el propio servidor | `8.8.8.8` / `8.8.4.4` (Google Public DNS) |

>  Sustituye estos valores por los de tu propia red y dominio en cada paso.

---

### Requisitos

- Debian, Ubuntu o derivado con acceso a `apt-get`.
- Acceso `root` o `sudo`.
- Una dirección IP fija asignada a la máquina que hará de servidor DNS.
- Un dominio propio (real o de laboratorio) para configurar como zona.

---

###  paso a paso

###  Instalar BIND9 y hacer copia de seguridad

```bash
apt-get install bind9
cd /etc/bind
mkdir backup
cp *.* backup/
```

> Antes de tocar nada, siempre conviene guardar una copia de los ficheros originales.

### Editar `named.conf.local` — declarar tus zonas

```conf
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

// zona directa
zone "hackingyseguridad.es" {
    type master;
    file "/etc/bind/db.hackingyseguridad";
};

// zona inversa
zone "252.1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
};
```

| Línea | Qué significa |
|---|---|
| `zone "hackingyseguridad.es"` | Declara la zona directa de tu dominio |
| `zone "252.1.168.192.in-addr.arpa"` | Declara la zona inversa; **la IP se escribe al revés** seguida de `.in-addr.arpa` |
| `type master` | Este servidor manda sobre la zona (no replica de otro) |
| `file` | Dónde vive el contenido real de la zona (registros) |

### Crear el fichero de la zona directa

```bash
cp db.empty db.hackingyseguridad
```

Editar `db.hackingyseguridad`:

```dns
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL    86400
@       IN      SOA     hackingyseguridad.es. admin.hackingyseguridad.es. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
@       IN      NS      server.hackingyseguridad.es.
@       IN      A       192.168.1.252
hackingyseguridad.es. IN A 192.168.1.252
```

### Crear el fichero de la zona inversa

```bash
cp db.127 db.192
```

Editar `db.192`:

```dns
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     hackingyseguridad.es. admin.hackingyseguridad.es. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@              IN      NS      server.hackingyseguridad.es.
252.1.168.192.in-addr.arpa. IN PTR server.hackingyseguridad.es.
```

### (Opcional pero recomendado) Revisar `named.conf.options`

Ver la sección [named.conf.options explicado](#-namedconfoptions-explicado) más abajo para entender qué tocar aquí (forwarders, `allow-query`, IPv6...).

###  Configurar el DNS "upstream" del propio servidor

Editar `/etc/resolv.conf`:

```conf
nameserver 8.8.8.8
nameserver 8.8.4.4
```

> Esto es el DNS que usa **el propio sistema operativo** del servidor para resolver nombres (por ejemplo, para sus propias actualizaciones), no tiene que ver con lo que `named` sirve a otros clientes.

### Arrancar el servicio

```bash
service bind9 start
```

###  Comprobar el estado

```bash
service bind9 status
```

###  (Diagnóstico) Ejecutar `named` en primer plano

```bash
named
```

> Lanzar `named` directamente (sin `service`) muestra sus mensajes de arranque y errores de configuración por consola — muy útil si el servicio no arranca. Detén el servicio (`service bind9 stop`) antes de probarlo así, para no tener dos instancias escuchando en el mismo puerto.

---

### Entendiendo el fichero de zona (SOA y registros)

Todo fichero de zona empieza con un registro **SOA** (*Start of Authority*), que define los parámetros de administración de la zona:

| Campo | Significado | Valor de ejemplo | Recomendación |
|---|---|---|---|
| **Serial** | Número de versión de la zona; los servidores esclavos lo usan para saber si hay cambios | `1` | Incrementarlo en **cada cambio** (formato habitual: `AAAAMMDDNN`, ej. `2026071601`) |
| **Refresh** | Cada cuánto un esclavo comprueba si hay una versión nueva | `604800` (7 días) | Bajarlo si tienes esclavos y cambias la zona a menudo |
| **Retry** | Cada cuánto reintenta el esclavo si el maestro no respondió | `86400` (1 día) | Debe ser menor que Refresh |
| **Expire** | Tras cuánto tiempo sin contactar al maestro, el esclavo deja de responder por la zona | `2419200` (28 días) | Margen de seguridad ante caídas largas |
| **Negative Cache TTL** | Cuánto tiempo se cachean las respuestas "no existe" (NXDOMAIN) | `86400` (1 día) | Ajustar según cuán dinámica sea la zona |

### Tipos de registro usados en este ejemplo

| Registro | Significado | Ejemplo en el repo |
|---|---|---|
| **SOA** | Autoridad e información administrativa de la zona | (ver tabla anterior) |
| **NS** | Servidor de nombres autoritativo para la zona | `@ IN NS server.hackingyseguridad.es.` |
| **A** | Nombre → dirección IPv4 | `@ IN A 192.168.1.252` |
| **PTR** | Dirección IP → nombre (solo en zonas inversas) | `...in-addr.arpa. IN PTR server.hackingyseguridad.es.` |

---

### named.conf.options explicado

```conf
options {
    directory "/var/cache/bind";

    // Si hay firewall entre tú y los servidores con los que hablas,
    // puede que necesites abrirlo a varios puertos:
    // http://www.kb.cert.org/vuls/id/800113

    // Si tu ISP te dio una o más IPs de DNS estables, puedes usarlas
    // como forwarders descomentando este bloque:
    // forwarders {
    //     0.0.0.0;
    // };

    // Si BIND se queja de que la clave raíz ha expirado, actualiza tus
    // claves: https://www.isc.org/bind-keys
    // dnssec-validation auto;

    // Permite consultas de cualquier origen:
    // actúa como resolver recursivo para cualquier dominio, y como DNS
    // autoritativo solo para los dominios configurados en named.conf.local
    allow-query {
        any;
    };

    listen-on-v6 { any; };
};
```

| Directiva | Qué hace | Consideración |
|---|---|---|
| `directory` | Carpeta de trabajo donde BIND guarda su caché y ficheros de estado | Normalmente no hace falta tocarla |
| `forwarders { ... }` | Si se define, BIND reenvía las consultas que no sabe resolver a esas IPs en lugar de ir él mismo a los servidores raíz | Útil si tu ISP o red ya tiene resolvers rápidos |
| `dnssec-validation auto` | Activa la validación DNSSEC de las respuestas | Recomendado en producción; puede dar problemas si las claves raíz están desactualizadas |
| `allow-query { any; }` | Permite que **cualquier IP** haga consultas a este servidor | ⚠️ Ver advertencia de seguridad más abajo |
| `listen-on-v6 { any; }` | Escucha en todas las interfaces IPv6 | Quítalo o restríngelo si no usas IPv6 |

---

### Verificación y pruebas

Una vez arrancado el servicio, comprueba que todo funciona **antes** de darlo por bueno:

```bash
# Validar la sintaxis general de la configuración
named-checkconf

# Validar un fichero de zona concreto
named-checkzone hackingyseguridad.es /etc/bind/db.hackingyseguridad
named-checkzone 252.1.168.192.in-addr.arpa /etc/bind/db.192

# Consultar la zona directa contra tu propio servidor
dig @192.168.1.252 hackingyseguridad.es

# Consultar la zona inversa
dig @192.168.1.252 -x 192.168.1.252

# Alternativa con nslookup
nslookup hackingyseguridad.es 192.168.1.252
```

| Herramienta | Para qué sirve |
|---|---|
| `named-checkconf` | Detecta errores de sintaxis en `named.conf` y los ficheros incluidos, **sin** arrancar el servicio |
| `named-checkzone <zona> <fichero>` | Valida que un fichero de zona concreto es correcto (SOA, serial, sintaxis de registros...) |
| `dig @servidor dominio` | Lanza una consulta DNS directamente contra tu servidor, sin depender del resolver del sistema |
| `dig @servidor -x IP` | Consulta inversa (PTR) contra tu servidor |

---

### Solución de problemas

| Síntoma | Causas probables | Qué revisar |
|---|---|---|
| `service bind9 start` falla o el servicio no queda activo | Error de sintaxis en algún fichero de configuración | `named-checkconf`, `journalctl -u bind9 -e`, o lanzar `named` en primer plano |
| `dig` no devuelve respuesta | Firewall bloqueando el puerto 53 (UDP/TCP), o `allow-query` restringido a otro rango | `ss -lntup | grep 53`, revisar `allow-query` en `named.conf.options` |
| Serial no se actualiza en los esclavos | Olvidaste incrementar el `Serial` tras editar la zona | Incrementa el número y reinicia/recarga (`rndc reload`) |
| Error `zone ... not loaded due to errors` en los logs | Sintaxis incorrecta en el fichero de zona (falta un punto final, paréntesis descuadrados...) | `named-checkzone` señala la línea exacta |
| El servidor resuelve dominios externos pero no el propio | La zona no está declarada en `named.conf.local`, o el `file` apunta a una ruta equivocada | Revisar rutas y permisos de los ficheros `db.*` |

---

##  Aviso sobre los ficheros de ejemplo del repositorio

El **README original** describe el escenario con el dominio `hackingyseguridad.es` y la red `192.168.1.252`, pero algunos de los **ficheros ya incluidos** en el repositorio (`named.conf.local`, `db.hackingyseguridad`, `db.217`) contienen en realidad datos de ejemplo de **otro dominio y otra red** (`terra.es`, rango `217.127.199.x`), residuo de una configuración distinta usada como plantilla.

**Antes de usar este repositorio, revisa y sustituye manualmente:**

- El nombre de dominio en `named.conf.local` y en el fichero de zona directa.
- La red y zona inversa (`in-addr.arpa`) según tu propio rango de IP.
- Cualquier dirección IP, `allow-transfer` o referencia a hosts que no sean los tuyos.

No copies estos ficheros "tal cual" en un servidor real sin adaptarlos a tu propio dominio e infraestructura.

---


---

# Configuracion :

/etc/bind

Ejemplo de configuracion basica con bind9. DNS autoritativo hackingyseguridad.es, con direccionamiento privado IP 192.168.1.252 

apt-get install bind9

cd /ect/bind

mkdir backcup

cp *.* backup

1º.- editamos fichero: named.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
//zona directa
zone "hackingyseguridad.es"
  {type master; file "/etc/bind/db.hackingyseguridad";};
//zona inversa
zone "252.1.168.192.in-addr.arpa"
  {type master; file "/etc/bind/db.192";};

2º.- cp db.empty db.hackingyseguridad

3º.- editamos fichero: db.hackingyseguridad

; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL 86400
@       IN SOA hackingyseguridad.es. admin.hackingyseguridad.es. (
              1   ; Serial
          604800   ; Refresh
          86400   ; Retry
          2419200   ; Expire
          86400 ) ; Negative Cache TTL
@       IN NS server.hackingyseguridad.es.
@       IN A 192.168.1.252
hackingyseguridad.es.   IN A 192.168.1.252


4º.- cp db.127 db.192

5º.- editamos fichero: db.192

;
; BIND reverse data file for local loopback interface
;
$TTL 604800
@ IN SOA hackingyseguridad.es. admin.hackingyseguridad.es. (
          1   ; Serial
      604800   ; Refresh
      86400   ; Retry
      2419200   ; Expire
      604800 ) ; Negative Cache TTL
;
@ IN NS server.hackingyseguridad.es.
252.1.168.192.in-addr.arpa. IN PTR server.hackingyseguridad.es.


6º.- editamos fichero: named.conf.options

7º.- editamos fichero: /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4

8º.- service bind9 start

9º.- service bind9 status

10.- named




#
http://www.hackingyseguridad.com/
#
