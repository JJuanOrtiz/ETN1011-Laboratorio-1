# Laboratorio 2 --- FRRouting en WSL2/Ubuntu y equivalencias con Cisco IOS

## 1. Datos del entorno

**Universidad:** Universidad Mayor de San Andrés (UMSA)\
**Carrera:** Ingeniería Electrónica\
**Asignatura:** ETN1011 --- Laboratorio de Sistemas de Comunicación II\
**Laboratorio:** Laboratorio 2 --- FRRouting en WSL2/Ubuntu y
equivalencias con Cisco IOS\
**Estudiante:** Juan Luis Ortiz Aliaga\
**Usuario de Linux:** `juan_ortiz`\
**Sistema operativo:** Ubuntu 24.04.5 LTS (Noble Numbat)\
**Entorno:** WSL2 sobre Windows 11\
**Arquitectura:** `x86_64`\
**Kernel:** `6.18.33.2-microsoft-standard-WSL2`\
**Versión de FRRouting:** 8.4.4

### Comandos utilizados para identificar el entorno

``` bash
whoami
uname -a
cat /etc/os-release
sudo vtysh -c 'show version'
```

Los datos del sistema operativo, usuario, kernel y versión de FRR se
obtuvieron durante las comprobaciones del entorno y del servicio.

## 2. Objetivo

Configurar y explorar un router FRRouting (FRR) en Ubuntu sobre WSL2,
identificar la relación entre FRR y la tabla de rutas del kernel Linux,
practicar rutas conectadas y rutas estáticas de descarte, comprobar la
persistencia de la configuración y realizar tareas básicas de
diagnóstico. Finalmente, comparar los comandos principales con sus
equivalentes aproximados en Cisco IOS.

## 3. Topología lógica

``` text
Windows
└── WSL2 / Ubuntu
    └── R1: Linux + FRR
        ├── eth0       Interfaz de WSL; se conserva la ruta por defecto
        └── dummy0     Interfaz virtual de laboratorio
             └── 10.10.10.1/24
```

`dummy0` es una interfaz virtual local. Sirve para practicar
configuración de interfaces y rutas conectadas, pero no representa una
LAN externa real.

## 4. Configuración realizada

### 4.1. Verificación del entorno y del servicio

Se comprobó el sistema Ubuntu y la versión del kernel. También se
verificó el servicio FRR mediante `systemctl`, la versión de FRR y el
estado de sus procesos.

La versión observada fue FRR 8.4.4. Durante el diagnóstico, el servicio
figuraba como `active (running)` y sus procesos principales (`watchfrr`,
`zebra`, `ospfd` y `staticd`) aparecían activos.

Comandos de consulta utilizados:

``` bash
cat /etc/os-release
uname -r
sudo systemctl status frr --no-pager
sudo vtysh -c 'show version'
```

### 4.2. Interfaz virtual de laboratorio

Se configuró la interfaz `dummy0` con la dirección `10.10.10.1/24` y se
activó. La interfaz apareció en estado `UP`, con protocolo de línea
`up`.

Comandos de referencia:

``` bash
sudo ip link add dummy0 type dummy
sudo ip addr add 10.10.10.1/24 dev dummy0
sudo ip link set dummy0 up
ip -br addr show dummy0
ip -4 route show 10.10.10.0/24
```

Si la interfaz ya existe, no se debe volver a crearla sin comprobar su
estado. En WSL2, una interfaz virtual creada manualmente puede no
persistir después de reiniciar el entorno.

### 4.3. Ruta conectada

La dirección `10.10.10.1/24` hace que Linux instale una ruta conectada
para `10.10.10.0/24`. Se consultaron las rutas desde Linux y desde FRR:

``` bash
ip -4 route show dev dummy0
sudo vtysh -c 'show ip route connected'
```

La tabla de Linux y la tabla mostrada por FRR tienen perspectivas
relacionadas, pero no idénticas: Linux mantiene la tabla de rutas del
kernel, mientras que FRR administra la información de enrutamiento y
coordina su instalación mediante `zebra`.

### 4.4. Ruta estática de descarte (blackhole)

Se configuró en FRR la ruta:

``` text
ip route 198.51.100.0/24 blackhole
```

El prefijo `198.51.100.0/24` está reservado para documentación. Una ruta
`blackhole` descarta los paquetes dirigidos a ese prefijo en lugar de
reenviarlos a una siguiente dirección.

La ruta fue visible desde FRR con esta consulta:

``` bash
sudo vtysh -c 'show ip route 198.51.100.0/24'
```

La salida obtenida indicaba:

``` text
Routing entry for 198.51.100.0/24
  Known via "static", distance 1, metric 0, best
  * unreachable (blackhole), weight 1
```

También se verificó desde Linux:

``` bash
ip -4 route show 198.51.100.0/24
```

La salida mostraba:

``` text
blackhole 198.51.100.0/24 proto static scope global metric 20
```

Estas consultas confirmaron que la ruta de descarte figuraba tanto en la
configuración de enrutamiento de FRR como en la tabla del kernel.

### 4.5. Configuración activa y persistencia

Se consultó la configuración activa de FRR con:

``` bash
sudo vtysh -c 'show running-config'
```

La configuración observada incluía, entre otros elementos:

``` text
frr version 8.4.4
frr defaults traditional
hostname DESKTOP-UJGJH52
service integrated-vtysh-config
ip route 198.51.100.0/24 blackhole
router ospf
 network 10.10.10.0/24 area 0
```

La ruta se guardó en el archivo de configuración de FRR. Se comprobó
posteriormente con:

``` bash
sudo grep -n "198.51.100.0" /etc/frr/frr.conf
```

Resultado obtenido:

``` text
10:ip route 198.51.100.0/24 blackhole
```

Esto confirma que la ruta aparece guardada en `/etc/frr/frr.conf`.
`write memory` guarda la configuración de FRR, pero no garantiza que una
interfaz virtual creada manualmente con `ip link` se vuelva a crear
automáticamente después de reiniciar WSL2.

## 5. Diagnóstico del servicio

Se consultaron el estado de FRR, los registros del servicio y la
información de versión/configuración:

``` bash
sudo systemctl status frr --no-pager
sudo journalctl -u frr -n 50 --no-pager
sudo vtysh -c 'show version'
sudo vtysh -c 'show running-config'
sudo vtysh -c 'show ip route'
```

Durante el arranque aparecieron mensajes iniciales de conexión fallida,
seguidos por mensajes que indicaban que los daemons habían establecido
conexión y estaban activos. En las comprobaciones posteriores, el
servicio figuraba como activo. No se identificó evidencia de un fallo
persistente del servicio.

## 6. Comprobación de interfaces y contadores

Se consultaron los contadores de la interfaz mediante:

``` bash
ip -s link show dummy0
```

También se consultó el estado de la interfaz desde FRR:

``` text
show interface dummy0
```

La interfaz aparecía `UP,BROADCAST,RUNNING,NOARP`, con dirección
`10.10.10.1/24` y MTU de 1500. En una de las lecturas de contadores, RX
era 0 y TX era mayor que 0, sin errores ni descartes registrados en esa
salida. Estos contadores por sí solos no demuestran que la prueba de
descarte de la ruta `blackhole` haya funcionado correctamente.

## 7. OSPF con un solo router

Se habilitó `ospfd` en la configuración de daemons y se configuró OSPF
para anunciar la red de laboratorio en el área 0:

``` text
router ospf
 network 10.10.10.0/24 area 0
```

Se consultaron los comandos:

``` text
show ip ospf interface
show ip ospf neighbor
```

La interfaz `dummy0` apareció asociada al área 0. No se encontraron
vecinos OSPF. Este resultado es el esperado en una topología con un
único router y sin un segundo proceso OSPF conectado a la misma red. Por
lo tanto, la ausencia de vecinos no se considera un error de
configuración por sí sola.

## 8. Prueba de descarte de la ruta blackhole

La guía propone comprobar el comportamiento de la ruta mediante los siguientes comandos:

``` text 
ip route get 198.51.100.10
ping -c 2 198.51.100.10
```

### 8.1. Verificación de la configuración

Se realizaron las siguientes comprobaciones:

a) Tabla de rutas de FRR

``` text
Routing entry for 198.51.100.0/24
  Known via "static", distance 1, metric 0, best
  Last update 00:00:28 ago
  * unreachable (blackhole), weight 1
```

FRR reconoce la ruta estática como una ruta de tipo blackhole.

b) Tabla de rutas del kernel de Linux

``` text
blackhole 198.51.100.0/24 proto static metric 20
```

La salida confirma que la ruta también está presente en la tabla de rutas IPv4 del kernel.

c) Configuración persistente de FRR

``` text
10:ip route 198.51.100.0/24 blackhole
```

La ruta aparece en /etc/frr/frr.conf, por lo que está guardada en el archivo de configuración.

### 8.2. Resultado de la prueba de tráfico

Los comandos de prueba devolvieron los siguientes mensajes:

``` text

RTNETLINK answers: Invalid argument

ping: connect: Invalid argument
```

Por tanto, las consultas de ruta y la prueba con ping no produjeron una salida normal en este entorno WSL2.

### 8.3. Conclusión

Se verificó que la ruta `blackhole` está configurada en FRR, instalada en el kernel de Linux y guardada en el archivo de configuración persistente. Sin embargo, la prueba con `ip route get` y ping devolvió `Invalid argument`, por lo que no se obtuvo una demostración directa mediante esos comandos del descarte de paquetes.

La configuración de la ruta quedó verificada, pero el comportamiento de la prueba de tráfico se registra como una incidencia observada en el entorno utilizado, sin atribuir una causa definitiva que no se haya demostrado.
 
## 9. Equivalencias principales con Cisco IOS

  -------------------------------------------------------------------------------------
  Operación          FRR / Linux                 Cisco IOS clásico
  ------------------ --------------------------- --------------------------------------
  Ver interfaces     `ip -br addr` /             `show ip interface brief`
                     `show interface`            

  Ver rutas          `ip -4 route` /             `show ip route`
                     `show ip route`             

  Entrar a la CLI    `sudo vtysh`                `enable`

  Entrar al modo de  `configure terminal`        `configure terminal`
  configuración                                  

  Configurar IP de   `ip addr add ... dev ...`   `interface ...` y `ip address ...`
  interfaz           en Linux                    

  Activar interfaz   `ip link set ... up`        `no shutdown`

  Ruta de descarte   `ip route ... blackhole` en `ip route RED MÁSCARA Null0`
                     FRR                         

  Ver configuración  `show running-config`       `show running-config`

  Guardar            `write memory`              `copy running-config startup-config`
  configuración                                  

  Consultar          `journalctl -u frr` en      `show logging`
  registros          Linux                       

  Ver vecinos OSPF   `show ip ospf neighbor`     `show ip ospf neighbor`
  -------------------------------------------------------------------------------------

Las equivalencias son aproximadas. FRR trabaja sobre Linux, por lo que
algunas tareas se ejecutan en Bash y otras en `vtysh`; en un router
Cisco IOS, la administración de interfaces y del sistema se realiza
mediante su propia CLI.

## 10. Conclusiones

1.  Se verificó la instalación y el funcionamiento del servicio FRR
    8.4.4 en Ubuntu sobre WSL2.
2.  Se utilizó la interfaz virtual `dummy0` con la dirección
    `10.10.10.1/24` para practicar interfaces y rutas conectadas.
3.  Se configuró la ruta `198.51.100.0/24` de tipo `blackhole` y se
    confirmó su presencia en FRR y en la tabla de rutas del kernel.
4.  Se comprobó que la ruta está guardada en `/etc/frr/frr.conf`.
5.  Se revisaron el estado del servicio, los registros, las interfaces y
    la configuración activa.
6.  Se preparó OSPF para la red de laboratorio. La ausencia de vecinos
    es coherente con el uso de un solo router.
7.  La prueba de `ip route get` y `ping` para el destino de la ruta
    `blackhole` devolvió `Invalid argument`. Por ello, esa prueba se
    documenta como incidencia pendiente y no como una demostración
    exitosa del descarte.
8.  Se compararon los comandos más importantes de FRR/Linux con sus
    equivalentes aproximados en Cisco IOS.

## 11. Respuestas breves para la defensa oral

**1. ¿Qué diferencia hay entre FRR y el kernel Linux?**\
FRR calcula y administra rutas mediante sus procesos de enrutamiento;
Linux mantiene la tabla de rutas del kernel y realiza el reenvío de
paquetes.

**2. ¿Qué función cumple `zebra`?**\
Coordina la información de interfaces y rutas entre los daemons de FRR y
el kernel Linux.

**3. ¿Qué diferencia hay entre `show ip route` e `ip -4 route`?**\
`show ip route` muestra la perspectiva de enrutamiento de FRR;
`ip -4 route` consulta la tabla de rutas IPv4 del kernel Linux.

**4. ¿Qué conserva `write memory`?**\
Guarda la configuración de FRR. No crea persistencia automática para una
interfaz temporal añadida con `ip link`.

**5. ¿Por qué no aparecen vecinos OSPF?**\
Porque el laboratorio tiene un solo router y no hay otro proceso OSPF
conectado a la misma red.

**6. ¿Qué hace una ruta `blackhole`?**\
Descarta los paquetes destinados al prefijo configurado.

**7. ¿Cuál es la equivalencia de `write memory` en Cisco IOS?**\
`copy running-config startup-config`.

**8. ¿Por qué se configura la IP de la interfaz fuera de `vtysh` en este
laboratorio?**\
Porque FRR administra protocolos y rutas, mientras que Linux administra
directamente las interfaces y sus direcciones en esta configuración.

## 12. Referencias

-   Documentación oficial de FRRouting:
    https://docs.frrouting.org/en/latest/
-   Configuración de FRR:
    https://docs.frrouting.org/en/latest/setup.html
-   `vtysh`: https://docs.frrouting.org/en/latest/vtysh.html
-   Rutas estáticas de FRR:
    https://docs.frrouting.org/en/latest/static.html
-   Instalación de WSL:
    https://learn.microsoft.com/en-us/windows/wsl/install
