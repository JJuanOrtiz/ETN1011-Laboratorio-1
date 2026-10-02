# Laboratorio 1 — Redes virtuales con Linux

---

## 1. Datos del entorno

**Asignatura:** ETN1011 — Laboratorio de Sistemas de Comunicación II
**Laboratorio:** Laboratorio 1 — Redes virtuales con Linux
**Estudiante:** Juan Luis Ortiz Aliaga
**Usuario de Linux:** `juan_ortiz`
**Sistema operativo:** Ubuntu 24.04.5 LTS (Noble Numbat)
**Entorno:** WSL2 sobre Windows 11
**Arquitectura:** `x86_64`
**Kernel:** `6.18.33.2-microsoft-standard-WSL2`

### Comandos utilizados

```bash
whoami
uname -a
cat /etc/os-release
```

### Resultados principales

```text
Usuario: juan_ortiz
Sistema: Ubuntu 24.04.5 LTS
Kernel: 6.18.33.2-microsoft-standard-WSL2
Arquitectura: x86_64
```

---

## 2. Objetivo

El objetivo del laboratorio fue **comprender y aplicar conceptos fundamentales de redes en Linux** mediante la creación de una topología de red virtual utilizando *network namespaces* y un enlace **VETH (Virtual Ethernet)**.

Se creó una comunicación directa entre dos hosts virtuales:

* `hostA`
* `hostB`

utilizando la red IPv4 `10.10.1.0/30`.

Durante el laboratorio se verificaron:

* Interfaces de red.
* Direcciones IPv4.
* Rutas directamente conectadas.
* Tabla de vecinos y resolución ARP.
* Conectividad mediante ICMP.
* Tráfico de red mediante `tcpdump`.
* Comportamiento de la red ante una falla controlada.
* Recuperación de la conectividad.

---

## 3. Topología

La topología implementada fue:

```text
                         Linux / WSL2
                              |
                 +------------+------------+
                 |                         |
              hostA                     hostB
           namespace                  namespace
                 |                         |
               vethA                    vethB
                 |                         |
                 +=========================+
                       enlace VETH

                  10.10.1.0/30
```

La conexión entre los dos namespaces se puede representar como:

```text
        hostA                                      hostB
    10.10.1.1/30                              10.10.1.2/30
         |                                         |
       vethA =================================== vethB
                  enlace Ethernet virtual
```

---

## 4. Direccionamiento

Se utilizó la red:

```text
10.10.1.0/30
```

La máscara de red correspondiente es:

```text
255.255.255.252
```

### Tabla de direccionamiento

| Dispositivo | Interfaz | Dirección IPv4 | Prefijo |
| ----------- | -------- | -------------- | ------- |
| `hostA`     | `vethA`  | `10.10.1.1`    | `/30`   |
| `hostB`     | `vethB`  | `10.10.1.2`    | `/30`   |

La distribución de direcciones es:

| Dirección   | Función              |
| ----------- | -------------------- |
| `10.10.1.0` | Dirección de red     |
| `10.10.1.1` | Dirección de `hostA` |
| `10.10.1.2` | Dirección de `hostB` |
| `10.10.1.3` | Broadcast            |

Una red `/30` contiene **4 direcciones**, de las cuales **2 pueden utilizarse para hosts**.

Esto resulta adecuado para un enlace punto a punto entre `hostA` y `hostB`.

---

## 5. Construcción de la red

### 5.1. Creación de los namespaces

Primero se crearon los dos espacios de nombres de red:

```bash
sudo ip netns add hostA
sudo ip netns add hostB
```

Se verificó su existencia mediante:

```bash
ip netns list
```

El resultado permitió comprobar que los namespaces habían sido creados correctamente:

```text
hostB
hostA
```

Los *network namespaces* permiten crear entornos de red aislados dentro de un mismo sistema Linux.

---

### 5.2. Creación del enlace VETH

Se creó un par de interfaces VETH:

```bash
sudo ip link add vethA type veth peer name vethB
```

Un par VETH funciona conceptualmente como un **cable Ethernet virtual**, donde cada extremo pertenece a una interfaz diferente.

Posteriormente se trasladó cada extremo al namespace correspondiente:

```bash
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
```

La topología resultante fue:

```text
hostA                              hostB
  |                                  |
vethA ============================ vethB
          cable virtual VETH
```

---

### 5.3. Asignación de direcciones IP

Se asignó la dirección `10.10.1.1/30` a `vethA`:

```bash
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
```

Se asignó la dirección `10.10.1.2/30` a `vethB`:

```bash
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB
```

Las direcciones fueron verificadas mediante:

```bash
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr
```

---

### 5.4. Activación de las interfaces

Se activaron las interfaces `loopback`:

```bash
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostB ip link set lo up
```

Posteriormente se activaron las interfaces VETH:

```bash
sudo ip netns exec hostA ip link set vethA up
sudo ip netns exec hostB ip link set vethB up
```

De esta manera, las interfaces quedaron disponibles para transmitir y recibir tráfico.

---

## 6. Verificación

### 6.1. Verificación de interfaces

Se verificaron las interfaces de `hostA` y `hostB` mediante:

```bash
sudo ip netns exec hostA ip link
sudo ip netns exec hostB ip link
```

Se comprobó la presencia de:

* `vethA` dentro de `hostA`.
* `vethB` dentro de `hostB`.

Además, las interfaces se encontraban en estado:

```text
UP
```

---

### 6.2. Verificación de direcciones

Las direcciones fueron comprobadas con:

```bash
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr
```

Se confirmó:

```text
hostA → 10.10.1.1/30
hostB → 10.10.1.2/30
```

---

### 6.3. Verificación de rutas

En `hostA` se ejecutó:

```bash
sudo ip netns exec hostA ip route
```

Obteniéndose:

```text
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1
```

En `hostB` se ejecutó:

```bash
sudo ip netns exec hostB ip route
```

Obteniéndose:

```text
10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2
```

Esto demuestra que Linux creó automáticamente las rutas hacia la red directamente conectada `10.10.1.0/30`.

**No fue necesario agregar rutas manualmente.**

---

### 6.4. Verificación de vecinos

Se consultó la tabla de vecinos mediante:

```bash
sudo ip netns exec hostA ip neigh
sudo ip netns exec hostB ip neigh
```

Esta tabla permite observar la relación entre una dirección IP y su correspondiente dirección MAC.

Durante la práctica se observó el estado:

```text
STALE
```

El estado `STALE` **no significa que exista una falla**. Indica que Linux conserva la asociación entre la dirección IP y la dirección MAC, pero la entrada no ha sido confirmada recientemente como activa.

La comunicación continuó funcionando correctamente.

---

### 6.5. Verificación de conectividad

Se comprobó la conectividad desde `hostA` hacia `hostB`:

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

También se comprobó la comunicación inversa:

```bash
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

El resultado de la prueba de conectividad fue:

```text
4 packets transmitted, 4 received, 0% packet loss
```

Esto confirmó que ambos namespaces podían comunicarse correctamente mediante el enlace VETH.

---

## 7. Captura y análisis de tráfico

Para observar el tráfico que circulaba por el enlace virtual se utilizó `tcpdump`.

En `hostA` se ejecutó:

```bash
sudo ip netns exec hostA tcpdump -n -i vethA
```

Mientras `tcpdump` permanecía ejecutándose, desde `hostB` se generó tráfico mediante:

```bash
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

La captura permitió observar tráfico ICMP como:

```text
10.10.1.2 > 10.10.1.1: ICMP echo request
10.10.1.1 > 10.10.1.2: ICMP echo reply
```

Esto demuestra que los paquetes estaban circulando realmente por la interfaz `vethA`.

El proceso de comunicación puede representarse como:

```text
hostB                                      hostA
10.10.1.2                                  10.10.1.1
    |                                           |
    | -------- ICMP Echo Request ------------> |
    |                                           |
    | <--------- ICMP Echo Reply ------------- |
    |                                           |
  vethB                                      vethA
      \_______________________________________/
                 enlace VETH
```

Durante la captura también pudieron observarse mensajes ARP utilizados para determinar las direcciones MAC correspondientes a las direcciones IPv4.

Por lo tanto, `tcpdump` permitió comprobar directamente el intercambio de paquetes entre ambos namespaces.

---

## 8. Falla y diagnóstico

Se realizó una **falla controlada** desactivando la interfaz `vethB` del namespace `hostB`.

Se ejecutó:

```bash
sudo ip netns exec hostB ip link set vethB down
```

Posteriormente se intentó establecer comunicación:

```bash
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

El sistema respondió:

```text
ping: connect: Network is unreachable
```

Esto ocurrió porque la interfaz `vethB` estaba desactivada y `hostB` no disponía de un enlace funcional para alcanzar `hostA`.

### 8.1. Recuperación del enlace

Para recuperar la conectividad se activó nuevamente la interfaz:

```bash
sudo ip netns exec hostB ip link set vethB up
```

Posteriormente se repitió la prueba:

```bash
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

El resultado fue:

```text
4 packets transmitted, 4 received, 0% packet loss
```

Esto confirmó que la conectividad había sido recuperada.

### 8.2. Diagnóstico

La secuencia de diagnóstico fue:

```text
vethB DOWN
     ↓
hostB pierde el enlace
     ↓
ping hacia hostA
     ↓
Network is unreachable
     ↓
vethB UP
     ↓
ping nuevamente
     ↓
4 paquetes recibidos
     ↓
Conectividad recuperada
```

---

## 9. Uso de OpenCode

Durante el laboratorio se consideró el uso de una herramienta de inteligencia artificial como apoyo para el diagnóstico de problemas de red.

El caso analizado fue la pérdida de conectividad después de desactivar `vethB`.

El problema observado fue:

```text
ping: connect: Network is unreachable
```

La revisión del estado de la interfaz permitió identificar que `vethB` se encontraba desactivada.

La solución aplicada fue:

```bash
sudo ip netns exec hostB ip link set vethB up
```

Posteriormente se comprobó nuevamente la conectividad mediante `ping`.

La inteligencia artificial fue utilizada como herramienta de apoyo para comprender el problema y orientar el diagnóstico. La solución fue posteriormente **verificada directamente mediante comandos ejecutados en Linux**.

---

## 10. Conclusiones

* Se crearon correctamente los *network namespaces* `hostA` y `hostB`.

* Se estableció un enlace virtual mediante un par de interfaces **VETH**.

* Se configuró correctamente la red IPv4 `10.10.1.0/30`.

* `hostA` recibió la dirección `10.10.1.1/30`.

* `hostB` recibió la dirección `10.10.1.2/30`.

* Linux generó automáticamente las rutas correspondientes a la red directamente conectada.

* Se verificó la resolución de direcciones mediante la tabla de vecinos y ARP.

* La conectividad fue comprobada mediante **ICMP y `ping`**, obteniendo comunicación sin pérdida de paquetes.

* Mediante **`tcpdump`** se observaron directamente los paquetes `ICMP Echo Request` y `ICMP Echo Reply`.

* Se realizó una falla controlada desactivando `vethB`, provocando la pérdida de conectividad.

* La conectividad fue recuperada al volver a activar `vethB`.

En conclusión, el laboratorio permitió comprender de manera práctica la relación entre los conceptos de **network namespace, interfaz de red, VETH, dirección IP, dirección MAC, ARP, rutas, ICMP y captura de tráfico**.

Estos conceptos constituyen una base fundamental para el trabajo posterior con redes Linux y topologías virtualizadas.

---

## Anexos

### Comandos principales utilizados

```bash
# Crear namespaces
sudo ip netns add hostA
sudo ip netns add hostB

# Ver namespaces
ip netns list

# Crear enlace VETH
sudo ip link add vethA type veth peer name vethB

# Mover las interfaces
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB

# Asignar direcciones IP
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB

# Activar interfaces
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostB ip link set lo up
sudo ip netns exec hostA ip link set vethA up
sudo ip netns exec hostB ip link set vethB up

# Ver rutas
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route

# Ver vecinos
sudo ip netns exec hostA ip neigh
sudo ip netns exec hostB ip neigh

# Comprobar conectividad
sudo ip netns exec hostA ping -c 4 10.10.1.2
sudo ip netns exec hostB ping -c 4 10.10.1.1

# Capturar tráfico
sudo ip netns exec hostA tcpdump -n -i vethA

# Provocar una falla
sudo ip netns exec hostB ip link set vethB down

# Recuperar la interfaz
sudo ip netns exec hostB ip link set vethB up
```

---

**Fin del informe — Laboratorio 1: Redes virtuales con Linux**

