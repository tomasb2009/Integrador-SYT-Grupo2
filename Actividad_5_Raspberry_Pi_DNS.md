# Acceso a Internet por Bluetooth y Enrutamiento (NAT) en una Raspberry Pi

## Índice

- [Acceso a Internet por Bluetooth y Enrutamiento (NAT) en una Raspberry Pi](#acceso-a-internet-por-bluetooth-y-enrutamiento-nat-en-una-raspberry-pi)
  - [Índice](#índice)
  - [1. Introducción](#1-introducción)
  - [2. Objetivo del informe](#2-objetivo-del-informe)
  - [3. Desarrollo teórico](#3-desarrollo-teórico)
    - [3.1 Raspberry Pi](#31-raspberry-pi)
    - [3.2 Bluetooth y el perfil PAN](#32-bluetooth-y-el-perfil-pan)
    - [3.3 Anclaje de red (tethering)](#33-anclaje-de-red-tethering)
    - [3.4 Enrutamiento e IP forwarding](#34-enrutamiento-e-ip-forwarding)
    - [3.5 NAT y _masquerade_](#35-nat-y-masquerade)
    - [3.6 nftables](#36-nftables)
    - [3.7 DHCP y DNS en un router](#37-dhcp-y-dns-en-un-router)
  - [4. Topología utilizada](#4-topología-utilizada)
  - [5. Requisitos previos](#5-requisitos-previos)
  - [6. Desarrollo práctico](#6-desarrollo-práctico)
    - [6.1 Verificar el estado inicial (DHCP y eth0)](#61-verificar-el-estado-inicial-dhcp-y-eth0)
    - [6.2 Instalar los paquetes necesarios](#62-instalar-los-paquetes-necesarios)
    - [6.3 Activar el anclaje Bluetooth en el celular Android](#63-activar-el-anclaje-bluetooth-en-el-celular-android)
    - [6.4 Emparejar la Raspberry Pi con el celular](#64-emparejar-la-raspberry-pi-con-el-celular)
    - [6.5 Crear la conexión Bluetooth PAN](#65-crear-la-conexión-bluetooth-pan)
    - [6.6 Verificar que la Raspberry Pi tiene Internet](#66-verificar-que-la-raspberry-pi-tiene-internet)
    - [6.7 Habilitar el reenvío de paquetes (IP forwarding)](#67-habilitar-el-reenvío-de-paquetes-ip-forwarding)
    - [6.8 Configurar NAT con nftables](#68-configurar-nat-con-nftables)
    - [6.9 Verificar el servidor DHCP (gateway y DNS)](#69-verificar-el-servidor-dhcp-gateway-y-dns)
    - [6.10 Pruebas de funcionamiento](#610-pruebas-de-funcionamiento)
  - [7. Conclusión](#7-conclusión)


---

## 1. Introducción

Un router hogareño cumple, en esencia, tres funciones: **obtiene acceso a Internet** por una interfaz (la _WAN_), **reparte direcciones IP** a los dispositivos de la red local (_DHCP_) y **reenvía el tráfico** de la red local hacia Internet traduciendo las direcciones (_NAT_).

En este trabajo la Raspberry Pi cumple ese rol, pero con una particularidad: su "WAN" no es un cable ni un WiFi, sino una conexión **Bluetooth** con un celular Android que comparte su acceso a Internet. La Raspberry Pi luego redirige ese tráfico hacia la interfaz **eth0**, que está conectada a un switch donde se encuentran los demás equipos de la red.

Se mantiene activa la configuración del servidor DHCP realizada en la entrega anterior.

## 2. Objetivo del informe

- Conectar la Raspberry Pi a Internet a través de Bluetooth usando el celular Android como _punto de acceso_ (anclaje Bluetooth).
- Habilitar el reenvío de paquetes y NAT para que los equipos conectados a `eth0` salgan a Internet a través de la Raspberry Pi.
- Verificar que los clientes de la red local obtienen dirección, puerta de enlace y DNS del servidor DHCP, y que pueden navegar.

> **Alcance:** en esta etapa **no** se crea una red WiFi propia en la Raspberry Pi. Solo se trabaja con Bluetooth (entrada) y Ethernet (salida hacia la red local).

## 3. Desarrollo teórico

### 3.1 Raspberry Pi

Es una computadora de placa única (SBC) de bajo costo que corre Linux (Raspberry Pi OS, basado en Debian). Incluye interfaz Ethernet, WiFi y Bluetooth, lo que la hace apta para funcionar como router casero.

### 3.2 Bluetooth y el perfil PAN

**Bluetooth** es una tecnología inalámbrica de corto alcance (aprox. 10 m) en la banda de 2,4 GHz. Se organiza en _perfiles_, que definen para qué se usa la conexión. El perfil que nos interesa es **PAN (Personal Area Networking)**, que permite transportar tráfico IP sobre Bluetooth, como si fuera una red Ethernet.

En PAN hay dos roles:

| Rol | Descripción | Quién lo cumple |
|-----|-------------|-----------------|
| **NAP** (_Network Access Point_) | Ofrece acceso a una red/Internet a los demás | El **celular Android** |
| **PANU** (_PAN User_) | Cliente que consume ese acceso | La **Raspberry Pi** |

Cuando la conexión se establece, Linux crea una interfaz de red virtual llamada **`bnep0`** (Bluetooth Network Encapsulation Protocol). A partir de ahí `bnep0` se comporta como cualquier otra interfaz: recibe una IP por DHCP (del celular), tiene una ruta por defecto, etc.

### 3.3 Anclaje de red (tethering)

Es la función del celular de compartir su conexión a Internet (datos móviles o WiFi) con otros dispositivos, ya sea por WiFi, USB o Bluetooth. En Android el anclaje Bluetooth entrega direcciones en la red `192.168.44.0/24` en la mayoría de los dispositivos (puede variar según marca y versión).

### 3.4 Enrutamiento e IP forwarding

Por defecto, un equipo Linux **descarta** los paquetes que le llegan por una interfaz y que no están dirigidos a él. Para que actúe como router, hay que activar el **reenvío de paquetes** (`net.ipv4.ip_forward = 1`), de modo que pueda pasar tráfico de `eth0` hacia `bnep0` y viceversa.

### 3.5 NAT y _masquerade_

Las IP de la red local (por ejemplo `192.168.2.0/24`) son **privadas** y no se enrutan en Internet. El **NAT** (_Network Address Translation_) reemplaza la IP de origen de los paquetes salientes por la IP de la interfaz de salida (`bnep0`), y deshace la traducción en las respuestas. La variante **masquerade** hace esto usando automáticamente la IP actual de la interfaz, ideal cuando la IP es dinámica, como en este caso (la asigna el celular).

### 3.6 nftables

Es el framework de filtrado de paquetes del kernel Linux que reemplaza a `iptables`. Organiza las reglas en _tablas_ → _cadenas_ → _reglas_. Es el que viene por defecto en las versiones recientes de Raspberry Pi OS.

### 3.7 DHCP y DNS en un router

- **DHCP:** reparte IP a los clientes e informa _qué puerta de enlace_ (gateway) y _qué servidores DNS_ usar.
- **DNS:** traduce nombres de dominio (`google.com`) a direcciones IP. Si el DNS está mal configurado, hay conectividad pero "no navega".

Para que el esquema funcione, el DHCP debe indicar a los clientes que su **gateway es la IP de eth0 de la Raspberry Pi**.

## 4. Topología utilizada

```text
   Internet
      │
┌─────┴──────┐   Bluetooth (PAN)
│  Celular   │◄──────────────────┐
│  Android   │      (NAP)        │
│ 192.168.44.1                   │ bnep0 (PANU)
└────────────┘            ┌──────┴────────────┐
                          │   Raspberry Pi    │
                          │  - Cliente BT PAN │
                          │  - NAT + forward  │
                          │  - Servidor DHCP  │
                          └──────┬────────────┘
                                 │ eth0 = 192.168.2.1/24
                          ┌──────┴──────┐
                          │   Switch    │
                          └──┬───────┬──┘
                             │       │
                          PC cliente  WAP (acceso WiFi)
```

## 5. Requisitos previos

- Raspberry Pi con Raspberry Pi OS y acceso por SSH o teclado/monitor.
- `eth0` con IP estática y servidor DHCP funcionando (entrega anterior).
- Celular Android con datos móviles o WiFi con Internet.
- Un equipo cliente conectado al switch para las pruebas.

## 6. Desarrollo práctico

### 6.1 Verificar el estado inicial (DHCP y eth0)

Antes de tocar nada confirmamos que lo que ya funcionaba sigue en pie.

```bash
ip -br address show eth0
```

- `ip`: herramienta moderna para administrar la red en Linux.
- `-br`: modo _brief_, muestra una salida resumida en una línea por interfaz.
- `address show eth0`: muestra las direcciones IP de la interfaz `eth0`.

Debe verse la IP estática del grupo (ejemplo `192.168.2.1/24`).

```bash
systemctl status isc-dhcp-server
```

- `systemctl`: controla servicios en sistemas que usan _systemd_.
- `status`: muestra si el servicio está activo y sus últimos mensajes.
- `isc-dhcp-server`: nombre del servicio DHCP configurado en la entrega anterior.

### 6.2 Instalar los paquetes necesarios

```bash
sudo apt update
sudo apt install -y bluez nftables
```

- `sudo`: ejecuta el comando con privilegios de administrador.
- `apt update`: actualiza la lista de paquetes disponibles en los repositorios.
- `apt install -y`: instala paquetes; `-y` responde "sí" automáticamente a las confirmaciones.
- `bluez`: pila Bluetooth oficial de Linux (incluye el servicio `bluetooth` y la herramienta `bluetoothctl`).
- `nftables`: framework de reglas de firewall/NAT.

Activamos el servicio Bluetooth:

```bash
sudo systemctl enable --now bluetooth
```

- `enable`: hace que el servicio arranque automáticamente al iniciar el sistema.
- `--now`: además lo inicia en este momento.
- `bluetooth`: servicio que administra el adaptador Bluetooth.

### 6.3 Activar el anclaje Bluetooth en el celular Android

1. Activar **Bluetooth** en el celular.
2. Ir a **Ajustes → Conexiones (o Red e Internet) → Zona WiFi y anclaje de red** (el nombre varía según la marca).
3. Activar **Anclaje Bluetooth** (_Bluetooth tethering_).
4. Dejar la pantalla de Bluetooth del celular abierta y visible durante el emparejamiento.

### 6.4 Emparejar la Raspberry Pi con el celular

```bash
bluetoothctl
```

- `bluetoothctl`: consola interactiva para administrar dispositivos Bluetooth. A partir de acá los comandos se escriben dentro de esa consola.

Dentro de `bluetoothctl`:

```text
power on
agent on
default-agent
scan on
```

- `power on`: enciende el adaptador Bluetooth de la Raspberry.
- `agent on` / `default-agent`: registra el agente que gestiona la confirmación de códigos de emparejamiento.
- `scan on`: busca dispositivos cercanos. Aparecerá el celular con su dirección MAC (formato `AA:BB:CC:DD:EE:FF`).

Cuando aparezca el celular (anotar su MAC), detener la búsqueda y emparejar:

```text
scan off
pair AA:BB:CC:DD:EE:FF
trust AA:BB:CC:DD:EE:FF
exit
```

- `scan off`: detiene la búsqueda.
- `pair <MAC>`: inicia el emparejamiento. Hay que **aceptar el código en el celular**.
- `trust <MAC>`: marca al dispositivo como confiable, para que se pueda reconectar automáticamente sin pedir confirmación.
- `exit`: sale de `bluetoothctl`.

### 6.5 Crear la conexión Bluetooth PAN

La Raspberry actúa como **PANU** y se conecta al **NAP** del celular.

```bash
sudo nmcli connection add type bluetooth con-name bt-android \
  autoconnect yes bluetooth.bdaddr AA:BB:CC:DD:EE:FF bluetooth.type panu
```

- `nmcli`: cliente de línea de comandos de NetworkManager.
- `connection add`: crea un nuevo perfil de conexión.
- `type bluetooth`: indica que el perfil es de tipo Bluetooth.
- `con-name bt-android`: nombre que le damos al perfil.
- `autoconnect yes`: intenta reconectarse solo cuando el celular esté disponible.
- `bluetooth.bdaddr`: dirección MAC del celular (reemplazar por la real).
- `bluetooth.type panu`: la Raspberry se conecta como cliente PAN (_PANU_).

Activar la conexión:

```bash
sudo nmcli connection up bt-android
```

- `connection up`: activa el perfil. Si todo sale bien, el celular le entrega una IP por DHCP a la Raspberry.

Luego se le pide IP al celular con NetworkManager.

```bash
sudo nmcli device set bnep0 managed yes
sudo nmcli device connect bnep0
```

- `device set bnep0 managed yes`: indica a NetworkManager que administre la interfaz `bnep0`.
- `device connect bnep0`: crea un perfil automático con DHCP y solicita una dirección al celular.

Para cortar el enlace: `sudo busctl call org.bluez /org/bluez/hci0/dev_AA_BB_CC_DD_EE_FF org.bluez.Network1 Disconnect`.

### 6.6 Verificar que la Raspberry Pi tiene Internet

```bash
ip -br address show bnep0
```

- Muestra la IP asignada a `bnep0` (típicamente `192.168.44.x/24`).

```bash
ip route
```

- `ip route`: muestra la tabla de rutas. Debe aparecer una línea `default via 192.168.44.1 dev bnep0`, es decir, la salida a Internet es el celular.

Para estar seguros de que el acceso sale por Bluetooth y no por otra interfaz, apagamos el WiFi de la Raspberry durante las pruebas:

```bash
sudo nmcli radio wifi off
```

- `radio wifi off`: apaga la radio WiFi de la Raspberry. **Ojo:** si se está trabajando por SSH a través del WiFi, se corta la sesión; en ese caso entrar por `eth0` desde la notebook con IP fija temporal (`192.168.2.10/24`) con `ssh admin@192.168.2.1`.

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

- `ping`: envía paquetes ICMP para comprobar conectividad.
- `-c 4`: envía solo 4 paquetes.
- `8.8.8.8`: DNS público de Google; prueba la conectividad **sin** depender de la resolución de nombres.
- `google.com`: prueba conectividad **y** resolución DNS.

### 6.7 Habilitar el reenvío de paquetes (IP forwarding)

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
```

- `echo "..."`: imprime el texto indicado.
- `|` (pipe): envía esa salida como entrada del siguiente comando.
- `sudo tee <archivo>`: escribe la entrada en el archivo con privilegios de administrador (se usa `tee` porque una redirección `>` con `sudo` no elevaría los permisos de escritura).
- `/etc/sysctl.d/99-ip-forward.conf`: archivo de configuración del kernel que se aplica en cada arranque, por lo que el cambio es **permanente**.
- `sysctl --system`: recarga todos los archivos de configuración del kernel, aplicando el cambio ahora.

Verificación:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

- `cat`: muestra el contenido de un archivo. Debe devolver `1`.

### 6.8 Configurar NAT con nftables

Editamos el archivo de configuración de nftables:

```bash
sudo nano /etc/nftables.conf
```

- `nano`: editor de texto simple de terminal.

Contenido del archivo:

```nft
#!/usr/sbin/nft -f

flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "bnep0" masquerade
    }
}

table ip filter {
    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        iifname "eth0" oifname "bnep0" accept
    }
}
```

Explicación línea por línea:

- `flush ruleset`: borra todas las reglas previas, para partir de una configuración limpia.
- `table ip nat`: tabla para IPv4 dedicada a traducción de direcciones.
- `chain postrouting`: cadena que procesa los paquetes **justo antes de salir** del equipo.
- `type nat hook postrouting priority srcnat; policy accept;`: cadena de tipo NAT enganchada en _postrouting_, que acepta por defecto.
- `oifname "bnep0" masquerade`: a todo paquete que sale por `bnep0` se le cambia la IP de origen por la IP de `bnep0` (la del celular).
- `table ip filter` / `chain forward`: cadena que procesa el tráfico **que atraviesa** la Raspberry (no el destinado a ella).
- `policy drop`: por defecto se descarta el tráfico reenviado (política restrictiva).
- `ct state established,related accept`: permite las **respuestas** a conexiones ya iniciadas desde la red local.
- `iifname "eth0" oifname "bnep0" accept`: permite iniciar conexiones desde `eth0` hacia `bnep0` (red local → Internet).

Aplicar y hacer permanente:

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo nft list ruleset
```

- `nft -f <archivo>`: carga las reglas desde el archivo.
- `systemctl enable --now nftables`: el servicio carga `/etc/nftables.conf` en cada arranque.
- `nft list ruleset`: muestra las reglas activas, para comprobar que se cargaron.

### 6.9 Verificar el servidor DHCP (gateway y DNS)

La configuración del servidor DHCP de la entrega anterior **ya entrega todo lo necesario**, por lo que no hace falta modificarla. Las líneas relevantes de `/etc/dhcp/dhcpd.conf` son:

```text
option domain-name-servers 8.8.8.8, 1.1.1.1;

subnet 192.168.2.0 netmask 255.255.255.0 {
    range 192.168.2.100 192.168.2.200;
    option routers 192.168.2.1;
    option broadcast-address 192.168.2.255;
}
```

- `option routers 192.168.2.1;`: los clientes reciben como puerta de enlace la IP de `eth0` de la Raspberry Pi. Hasta ahora esa puerta de enlace no llevaba a ningún lado; a partir de este trabajo la Raspberry realmente reenvía el tráfico hacia Internet.
- `option domain-name-servers 8.8.8.8, 1.1.1.1;`: DNS públicos de Google y Cloudflare. Los clientes consultan directamente a esos servidores, y las consultas viajan por el NAT de la Raspberry, por lo que los nombres externos se resuelven correctamente.

### 6.10 Pruebas de funcionamiento

**Desde un equipo cliente conectado al switch:**

Renovar la dirección y revisar lo recibido.

En Windows (CMD): `ipconfig /release`, `ipconfig /renew` y `ipconfig /all`.

Pruebas de conectividad:

```bash
ping -c 4 192.168.2.1
ping -c 4 8.8.8.8
ping -c 4 google.com
traceroute 8.8.8.8
```

- `traceroute`: lista los routers por los que pasa el paquete. El primer salto debe ser la Raspberry y el segundo el celular (`192.168.44.1`).

## 7. Conclusión

Se logró que la Raspberry Pi obtenga acceso a Internet a través de Bluetooth, comportándose como cliente PANU de un celular Android que actúa como NAP, y se la configuró como router: se habilitó el reenvío de paquetes, se aplicó NAT (masquerade) sobre `bnep0` con reglas nftables persistentes y se ajustó el servidor DHCP para que los clientes de `eth0` reciban la Raspberry como gateway y DNS válidos.

Entre las lecciones aprendidas: la interfaz `bnep0` es dinámica, por lo que conviene usar _masquerade_; la ruta por defecto debe ser única (no debe haber gateway en `eth0`); y la falta de navegación suele deberse al DNS o al forwarding, no al enlace. El rendimiento está limitado por el ancho de banda de Bluetooth, mucho menor que el de un enlace Ethernet o WiFi.
