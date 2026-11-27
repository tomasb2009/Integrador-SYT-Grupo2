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
    - [3.5 NAT y masquerade](#35-nat-y-masquerade)
    - [3.6 nftables](#36-nftables)
    - [3.7 DHCP y DNS en un router](#37-dhcp-y-dns-en-un-router)
    - [3.8 Recorrido de un paquete](#38-recorrido-de-un-paquete)
  - [4. Topología utilizada](#4-topología-utilizada)
  - [5. Desarrollo práctico](#5-desarrollo-práctico)
    - [5.1 Verificar el estado inicial](#51-verificar-el-estado-inicial)
    - [5.2 Emparejar la Raspberry Pi con el celular](#52-emparejar-la-raspberry-pi-con-el-celular)
    - [5.3 Establecer la conexión Bluetooth PAN](#53-establecer-la-conexión-bluetooth-pan)
    - [5.4 Verificar que la Raspberry Pi tiene Internet](#54-verificar-que-la-raspberry-pi-tiene-internet)
    - [5.5 Habilitar el reenvío de paquetes](#55-habilitar-el-reenvío-de-paquetes)
    - [5.6 Configurar NAT con nftables](#56-configurar-nat-con-nftables)
    - [5.7 Verificar el servidor DHCP](#57-verificar-el-servidor-dhcp)
    - [5.8 Pruebas de funcionamiento](#58-pruebas-de-funcionamiento)
  - [6. Conclusión](#6-conclusión)

---

## 1. Introducción

Un router hogareño cumple tres funciones: **obtiene Internet** por una interfaz (la _WAN_), **reparte direcciones IP** a la red local (_DHCP_) y **reenvía el tráfico** de la red local hacia Internet traduciendo las direcciones (_NAT_).

En este trabajo la Raspberry Pi cumple ese rol con una particularidad: su "WAN" es una conexión **Bluetooth** con un celular Android que comparte su Internet. La Raspberry redirige ese tráfico hacia **eth0**, conectada al switch del grupo. Se mantiene activa la configuración del servidor DHCP de la entrega anterior.

## 2. Objetivo del informe

- Conectar la Raspberry Pi a Internet por Bluetooth usando el anclaje del celular Android.
- Habilitar reenvío de paquetes y NAT para que los equipos conectados a `eth0` salgan a Internet a través de la Raspberry.
- Verificar que los clientes reciben dirección, gateway y DNS del servidor DHCP y pueden navegar.

> **Alcance:** en esta etapa no se crea una red WiFi propia en la Raspberry. Solo se usa Bluetooth (entrada) y Ethernet (salida hacia la red local).

## 3. Desarrollo teórico

### 3.1 Raspberry Pi

Computadora de placa única (SBC) de bajo costo que corre Linux (Raspberry Pi OS, basado en Debian). Incluye Ethernet, WiFi y Bluetooth, lo que la hace apta para funcionar como router.

### 3.2 Bluetooth y el perfil PAN

**Bluetooth** es una tecnología inalámbrica de corto alcance (aprox. 10 m) en la banda de 2,4 GHz. Se organiza en **perfiles**, que definen para qué se usa cada conexión. El perfil que nos interesa es **PAN (Personal Area Networking)**, que transporta **tráfico IP** sobre Bluetooth como si fuera una red Ethernet.

PAN define dos roles:

| Rol | Descripción | Quién lo cumple |
|-----|-------------|-----------------|
| **NAP** (_Network Access Point_) | Ofrece acceso a una red o a Internet a los demás | El **celular Android** |
| **PANU** (_PAN User_) | Cliente que consume ese acceso | La **Raspberry Pi** |

**Emparejamiento (_pairing_) y confianza (_trust_):** antes de usar un perfil, ambos dispositivos deben vincularse e intercambiar claves. `trust` permite que se reconecten sin pedir confirmación.

Al establecerse la conexión, Linux crea la interfaz virtual **`bnep0`** (_Bluetooth Network Encapsulation Protocol_), que encapsula paquetes IP dentro de Bluetooth. Desde ese momento `bnep0` se comporta como cualquier interfaz de red: recibe una IP por DHCP del celular y tiene una ruta por defecto.

**BlueZ** es la pila Bluetooth oficial de Linux. Expone sus funciones por **D-Bus** (un bus de mensajes entre programas del sistema), y es lo que usamos para pedir la conexión PAN.

### 3.3 Anclaje de red (tethering)

Función del celular que comparte su conexión a Internet (datos móviles o WiFi) con otros dispositivos por WiFi, USB o Bluetooth. Con el anclaje Bluetooth activo, el celular ofrece el servicio NAP y actúa como servidor DHCP de ese enlace: en nuestro caso entregó la red `10.73.154.0/24`, con el celular como gateway (`10.73.154.19`). La red puede variar según el celular y la operadora.

### 3.4 Enrutamiento e IP forwarding

Un equipo Linux **descarta** por defecto los paquetes que le llegan y no están dirigidos a él. Para que actúe como router hay que activar el **reenvío de paquetes** (`net.ipv4.ip_forward = 1`), que le permite pasar tráfico de `eth0` hacia `bnep0` y viceversa.

La **tabla de rutas** indica por qué interfaz sale cada destino. La **ruta por defecto** (`default`) es la que se usa cuando no hay una más específica y debe ser **única**: en nuestro caso apunta a `bnep0`, y `eth0` no debe tener gateway.

### 3.5 NAT y masquerade

Las IP de nuestra LAN (`192.168.2.0/24`) son **privadas** y no se enrutan en Internet. El **NAT** (_Network Address Translation_) reemplaza la IP de origen de los paquetes salientes por la de la interfaz de salida (`bnep0`) y deshace la traducción en las respuestas. Para saber a quién devolver cada respuesta, el kernel mantiene una tabla de conexiones (_connection tracking_).

**Masquerade** es la variante de NAT que usa automáticamente la IP actual de la interfaz de salida. Es la adecuada acá, porque la IP de `bnep0` la asigna el celular y puede cambiar.

### 3.6 nftables

Framework de filtrado de paquetes del kernel Linux, sucesor de `iptables` y el que usa por defecto Raspberry Pi OS. Organiza las reglas así:

- **Tabla:** agrupa cadenas de una misma familia (`ip` = IPv4) y propósito.
- **Cadena:** lista de reglas enganchada (_hook_) en un punto del recorrido del paquete: `postrouting` (justo antes de salir) o `forward` (tráfico que atraviesa el equipo).
- **Regla:** condición y acción (`accept`, `drop`, `masquerade`).

### 3.7 DHCP y DNS en un router

- **DHCP:** reparte IP e informa la puerta de enlace y los DNS. Nuestro servidor (`isc-dhcp-server`) entrega a los clientes `192.168.2.1` como gateway y `8.8.8.8`, `1.1.1.1` como DNS.
- **DNS:** traduce nombres de dominio a direcciones IP. Si falla, hay conectividad pero "no navega".

### 3.8 Recorrido de un paquete

Cuando una notebook de la LAN hace `ping 8.8.8.8`:

1. La notebook envía el paquete a su gateway, la Raspberry (`192.168.2.1`), por el switch.
2. La Raspberry lo recibe por `eth0` y, como el reenvío está activo y la regla `forward` lo permite, lo pasa hacia `bnep0`.
3. En `postrouting`, el **masquerade** cambia la IP de origen `192.168.2.x` por la de `bnep0` y registra la conexión.
4. El paquete viaja por Bluetooth al celular, que lo reenvía a Internet (haciendo su propio NAT).
5. La respuesta vuelve por el mismo camino: la Raspberry consulta su tabla de conexiones, restaura la IP de destino original y la envía por `eth0` a la notebook.

## 4. Topología utilizada

```text
   Internet
      │
┌─────┴──────┐   Bluetooth (PAN)
│  Celular   │◄──────────────────┐
│  Android   │      (NAP)        │
│ gateway: 10.73.154.19          │ bnep0 (PANU)
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

## 5. Desarrollo práctico

### 5.1 Verificar el estado inicial

Se confirma que lo configurado en la entrega anterior sigue funcionando.

```bash
ip -br address show eth0
systemctl status isc-dhcp-server
```

- `ip -br address show eth0`: muestra de forma resumida (`-br`) la IP de `eth0`. Debe ser `192.168.2.1/24`.
- `systemctl status isc-dhcp-server`: muestra si el servidor DHCP está activo (`active (running)`).

Si hace falta instalar paquetes, se hace antes de seguir (requiere Internet temporal, por ejemplo con el hotspot del celular, porque la red del switch no tiene salida):

```bash
sudo apt update
sudo apt install -y bluez nftables
sudo systemctl enable --now bluetooth
```

- `apt update`: actualiza la lista de paquetes disponibles.
- `apt install -y bluez nftables`: instala la pila Bluetooth (`bluez`, que incluye `bluetoothctl`) y el firewall `nftables`; `-y` acepta las confirmaciones.
- `systemctl enable --now bluetooth`: inicia el servicio Bluetooth ahora (`--now`) y en cada arranque (`enable`).

### 5.2 Emparejar la Raspberry Pi con el celular

En el celular, abrir **Ajustes → Bluetooth** y dejar la pantalla visible. Luego en la Raspberry:

```bash
bluetoothctl
```

- `bluetoothctl`: consola interactiva para administrar Bluetooth. Los siguientes comandos se escriben dentro de ella.

```text
power on
agent on
default-agent
scan on
```

- `power on`: enciende el adaptador Bluetooth.
- `agent on` / `default-agent`: registra el agente que gestiona la confirmación del código de emparejamiento.
- `scan on`: busca dispositivos cercanos. Aparece el celular con su MAC (formato `AA:BB:CC:DD:EE:FF`).

```text
scan off
pair AA:BB:CC:DD:EE:FF
trust AA:BB:CC:DD:EE:FF
exit
```

- `scan off`: detiene la búsqueda.
- `pair <MAC>`: inicia el emparejamiento. Hay que aceptar el código en el celular (y escribir `yes` en la Raspberry si lo pide).
- `trust <MAC>`: marca al celular como confiable.
- `exit`: sale de `bluetoothctl`.

Luego, en el celular, se activa **Ajustes → Zona WiFi y anclaje → Anclaje Bluetooth**. Se verifica con:

```bash
bluetoothctl info AA:BB:CC:DD:EE:FF | grep -E 'Paired|Trusted|NAP'
```

- `bluetoothctl info <MAC>`: muestra los datos del dispositivo.
- `grep -E 'Paired|Trusted|NAP'`: filtra solo esas líneas. Deben aparecer `Paired: yes`, `Trusted: yes` y `UUID: NAP`, que indica que el celular ofrece el servicio de punto de acceso.

### 5.3 Establecer la conexión Bluetooth PAN

Se le pide a BlueZ que conecte directamente al servicio **NAP** del celular. Para armar el comando, la MAC se escribe con `_` en lugar de `:`.

```bash
sudo busctl call org.bluez /org/bluez/hci0/dev_AA_BB_CC_DD_EE_FF org.bluez.Network1 Connect s "nap"
```

- `busctl call`: invoca un método de un servicio a través de D-Bus.
- `org.bluez`: el servicio de BlueZ.
- `/org/bluez/hci0/dev_AA_BB_CC_DD_EE_FF`: ruta del objeto que representa al celular dentro de BlueZ.
- `org.bluez.Network1 Connect s "nap"`: llama al método `Connect` de la interfaz de red PAN, con el texto (`s`) `"nap"`. Si sale bien, devuelve el nombre de la interfaz creada (`bnep0`).

Después se le pide una IP al celular con NetworkManager:

```bash
sudo nmcli device set bnep0 managed yes
sudo nmcli device connect bnep0
```

- `nmcli`: cliente de línea de comandos de NetworkManager.
- `device set bnep0 managed yes`: le indica a NetworkManager que administre `bnep0`.
- `device connect bnep0`: crea un perfil automático con DHCP y solicita la dirección al celular.

> Esta conexión no se restablece sola después de un reinicio: hay que repetir este paso con el celular desbloqueado y el anclaje activo.

### 5.4 Verificar que la Raspberry Pi tiene Internet

```bash
ip -br address show bnep0
ip route
ping -c 4 8.8.8.8
ping -c 4 google.com
```

- `ip -br address show bnep0`: muestra la IP que dio el celular (en nuestro caso `10.73.154.x/24`).
- `ip route`: muestra la tabla de rutas. Debe haber una línea `default via <IP del celular> dev bnep0` y ninguna ruta `default` por `eth0`.
- `ping -c 4 8.8.8.8`: envía 4 paquetes ICMP (`-c 4`) a un DNS público para probar conectividad **sin** depender de nombres.
- `ping -c 4 google.com`: prueba conectividad **y** resolución DNS.

### 5.5 Habilitar el reenvío de paquetes

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
cat /proc/sys/net/ipv4/ip_forward
```

- `echo "..." | sudo tee <archivo>`: escribe la línea en el archivo con permisos de administrador (se usa `tee` porque una redirección `>` con `sudo` no elevaría los permisos de escritura).
- `/etc/sysctl.d/99-ip-forward.conf`: configuración del kernel que se aplica en cada arranque, por lo que el cambio es **permanente**.
- `sysctl --system`: recarga la configuración del kernel y la aplica ahora.
- `cat /proc/sys/net/ipv4/ip_forward`: debe devolver `1`.

### 5.6 Configurar NAT con nftables

```bash
sudo nano /etc/nftables.conf
```

- `nano`: editor de texto de terminal. Se reemplaza el contenido del archivo por:

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

- `flush ruleset`: borra todas las reglas previas para partir de cero.
- `table ip nat` / `chain postrouting`: tabla y cadena de NAT que procesan los paquetes justo antes de salir.
- `oifname "bnep0" masquerade`: a todo paquete que sale por `bnep0` se le cambia la IP de origen por la de esa interfaz.
- `table ip filter` / `chain forward`: tabla y cadena que procesan el tráfico que **atraviesa** la Raspberry.
- `policy drop`: por defecto se descarta lo reenviado (política restrictiva).
- `ct state established,related accept`: permite las **respuestas** a conexiones ya iniciadas desde la LAN.
- `iifname "eth0" oifname "bnep0" accept`: permite iniciar conexiones desde `eth0` hacia `bnep0` (LAN → Internet).

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo nft list ruleset
```

- `nft -f <archivo>`: carga las reglas desde el archivo.
- `systemctl enable --now nftables`: aplica las reglas ahora y en cada arranque.
- `nft list ruleset`: muestra las reglas activas para comprobar que se cargaron.

### 5.7 Verificar el servidor DHCP

La configuración del servidor DHCP de la entrega anterior **ya entrega lo necesario**, por lo que no se modifica. Las líneas relevantes de `/etc/dhcp/dhcpd.conf` son:

```text
option domain-name-servers 8.8.8.8, 1.1.1.1;

subnet 192.168.2.0 netmask 255.255.255.0 {
    range 192.168.2.100 192.168.2.200;
    option routers 192.168.2.1;
}
```

- `option routers 192.168.2.1;`: los clientes reciben como gateway la IP de `eth0` de la Raspberry. Antes ese gateway no llevaba a ningún lado; ahora la Raspberry realmente reenvía el tráfico a Internet.
- `option domain-name-servers 8.8.8.8, 1.1.1.1;`: DNS públicos que los clientes consultan a través del NAT de la Raspberry.

### 5.8 Pruebas de funcionamiento

Desde una notebook Windows conectada al switch (con el WiFi apagado, para asegurar que sale por el cable):

```powershell
ipconfig /renew
ipconfig /all
ping -n 3 192.168.2.1
ping -n 3 8.8.8.8
ping -n 3 google.com
```

- `ipconfig /renew`: pide una nueva configuración al servidor DHCP.
- `ipconfig /all`: muestra la configuración recibida. Debe verse IP `192.168.2.100–200`, gateway y servidor DHCP `192.168.2.1`, y DNS `8.8.8.8` y `1.1.1.1`.
- `ping -n 3 <destino>`: envía 3 paquetes (en Windows el número se indica con `-n`). El primero prueba el gateway (la Raspberry), el segundo la salida a Internet y el tercero la resolución DNS.

## 6. Conclusión

Se logró que la Raspberry Pi obtenga acceso a Internet por Bluetooth, comportándose como cliente PANU de un celular Android que actúa como NAP, y se la configuró como router: se habilitó el reenvío de paquetes y se aplicó NAT (masquerade) sobre `bnep0` con reglas nftables persistentes. El servidor DHCP de la entrega anterior se mantuvo sin cambios, ya que entrega a los clientes la Raspberry como gateway y DNS válidos.

Entre las lecciones aprendidas: `bnep0` es una interfaz dinámica, por lo que conviene usar _masquerade_; la ruta por defecto debe ser única (sin gateway en `eth0`); y un enlace Bluetooth puede quedar colgado con la interfaz todavía activa, por lo que conviene verificar la conectividad real de la Raspberry antes de buscar fallas en el NAT. La conexión Bluetooth debe restablecerse manualmente tras un reinicio, y el ancho de banda está limitado por Bluetooth, mucho menor que Ethernet o WiFi.