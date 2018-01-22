# Ampliación Redes: Práctica 1.1

## Configuración Estática

**Ejercicio 1 [VM1].** Determinar los interfaces de red que tiene la máquina y las direcciones IP y/o MAC que tienen asignadas. Utilizar el comando `ip`.


    root@frontend:~# ip addr

**Ejercicio 2 [VM1, VM2, Router].** Activar los interfaces  `eth0` en las máquinas VM1, VM2 y Router, y asignar una dirección de red adecuada. La configuración debe realizarse con la utilidad `ip`, en particular los comandos `ip address` e `ip link`.


    root@frontend:~# ip link set eth0 up
    // La dirección IP cambia en cada VM
    root@frontend:~# ip addr add 10.0.0.1/24 dev eth0  // VM1

**Ejercicio 3 [VM1, VM2].** Arrancar la herramienta **wireshark** y activar la captura en el interfaz de red. Comprobar la conectividad entre VM1 y VM2 con la orden `ping`. Observar el tráfico generado, especialmente los protocolos encapsulados en cada datagrama y las direcciones origen y destino.

| **MAC Origen**    | **MAC Destino**   | **Protocolo** | **IP Origen** | **IP Destino** | **Tipo Mensaje**                 |
| ----------------- | ----------------- | ------------- | ------------- | -------------- | -------------------------------- |
| 02:00:00:00:01:00 | 00:00:00:00:00:00 | ARP           | 10.0.0.1      | 10.0.0.2       | Who has 10.0.0.2? Tell 10.0.0.1  |
| 02:00:00:00:02:00 | 02:00:00:00:01:00 | ARP           | 10.0.0.2      | 10.0.0.1       | 10.0.0.2 is at 02:00:00:00:02:00 |
| 02:00:00:00:01:00 | 02:00:00:00:02:00 | ICMP          | 10.0.0.1      | 10.0.0.2       | ECHO_REQUEST                     |
| 02:00:00:00:02:00 | 02:00:00:00:01:00 | ICMP          | 10.0.0.2      | 10.0.0.1       | ECHO_REPLY                       |


**Ejercicio 4 [VM1, VM2].** Ejecutar de nuevo la orden `ping` entre VM1 y VM2, y a continuación
comprobar el estado de la **tabla ARP** en VM1 y VM2 usando el comando `ip neigh`. El significado del estado de cada entrada de la tabla se puede consultar en la página de manual del comando.

***VM1:***

    root@frontend:~# ping 10.0.0.2
    root@frontend:~# ip neigh
    10.0.0.2 dev eth0 lladdr 02:00:00:00:02:00 REACHABLE

**Ejercicio 5 [Router, VM4].** Repetir la configuración de red para el segmento `192.168.0.0/24`.
Comprobar la conectividad entre Router y VM4; y entre Router, VM1 y VM2.

***VM3:***

    root@frontend:~# ip link set eth1 up
    root@frontend:~# ip addr add 192.168.0.3/24 dev eth1
    root@frontend:~# ping 10.0.0.1  // REACHABLE
    root@frontend:~# ping 10.0.0.2  // REACHABLE

***VM4:***

    root@frontend:~# ip link set eth0 up
    root@frontend:~# ip addr add 192.168.0.4/24 dev eth0
    root@frontend:~# ping 192.168.0.3  // REACHABLE
    root@frontend:~# ping 10.0.0.3  // UNREACHABLE
## Encaminamiento Estático

**Ejercicio 1 [Router].** Activar el reenvío de paquetes *(forwarding)* en Router para que efectivamente pueda funcionar como encaminador entre las redes `10.0.0.0/24` y `192.168.0.0/24`. Ejecutar el comando `# sysctl net.ipv4.ip_forward=1`.

***VM3:***

    root@frontend:~# sysctl net.ipv4.ip_forward=1

**Ejercicio 2 [VM1, VM2].** Añadir la máquina Router como router por defecto para VM1 y VM2. Usar el comando `ip route`.


    root@frontend:~# ip route add 192.168.0.0/24 via 10.0.0.3 dev eth0

**Ejercicio 3 [VM4].** Aunque la configuración adecuada para la tabla de rutas de hosts en redes como las consideradas en esta práctica consiste en añadir una ruta por defecto; es posible incluir rutas para redes concretas. Añadir a la tabla de rutas de VM4 una ruta a la red `10.0.0.0/24` vía Router.


    root@frontend:~# ip route add 10.0.0.0/24 via 192.168.0.3 dev eth0

**Ejercicio 4 [VM1, VM4].** Usar la orden `ping` entre las máquinas VM1 y VM4. Con ayuda de la
herramienta **wireshark** completar la siguiente tabla para todos los paquetes intercambiados hasta la recepción de la primera respuesta `ECHO_REPLY`.

***Red 10.0.0.0/24 - VM1:***

| **MAC Origen**    | **MAC Destino**   | **Protocolo** | **IP Origen** | **IP Destino** | **Tipo Mensaje** |
| ----------------- | ----------------- | ------------- | ------------- | -------------- | ---------------- |
| 02:00:00:00:01:00 | 02:00:00:00:03:00 | ICMP          | 10.0.0.1      | 192.168.0.4    | ECHO_REQUEST     |
| 02:00:00:00:03:00 | 02:00:00:00:01:00 | ICMP          | 192.168.0.4   | 10.0.0.1       | ECHO_REPLY       |


***Red 192.168.0.0/24 - VM4:***

| **MAC Origen**    | **MAC Destino**   | **Protocolo** | **IP Origen** | **IP Destino** | **Tipo Mensaje** |
| ----------------- | ----------------- | ------------- | ------------- | -------------- | ---------------- |
| 02:00:00:00:03:01 | 02:00:00:00:04:00 | ICMP          | 10.0.0.1      | 192.168.0.4    | ECHO_REQUEST     |
| 02:00:00:00:04:00 | 02:00:00:00:03:01 | ICMP          | 192.168.0.4   | 10.0.0.1       | ECHO_REPLY       |



## Configuración Dinámica de Hosts

**Ejercicio 1 [VM1, VM2, VM4].** Eliminar las direcciones de red de los interfaces `ip addr del`.


    // Cada VM deberá borrar su dirección
    root@frontend:~# ip addr del 10.0.0.1/24 dev eth0

**Ejercicio 2 [Router].** Configurar el **servidor DHCP** para las dos redes:

    # /etc/dhcp/dhcpd.conf
    subnet 10.0.0.0 netmask 255.255.255.0 {
        range 10.0.0.50 10.0.0.100;
        option routers 10.0.0.3;
        option broadcast-address 10.0.0.255;
    }
    subnet 192.168.0.0 netmask 255.255.255.0 {
        range 192.168.0.50 192.168.0.100;
        option routers 192.168.0.3;
        option broadcast-address 192.168.0.255;
    }

***VM3:***

    root@frontend:~# service isc-dhcp-server start.

**Ejercicio 3 [Router, VM1].** Iniciar la captura de paquetes en Router. Arrancar el cliente DHCP con `dhclient -d eth0` en la máquina virtual VM1 y observar el proceso de configuración. Completar la siguiente tabla:

***Mensajes DHCP intercambiados en la configuración***

| **IP Origen** | **IP Destino**  | **Mensaje DHCP** | **Opciones DHCP**        |
| ------------- | --------------- | ---------------- | ------------------------ |
| 0.0.0.0       | 255.255.255.255 | Discover         | 53, 50, 55               |
| 10.0.0.3      | 10.0.0.50       | Offer            | 53, 54, 51, 1, 28, 3, 15 |
| 0.0.0.0       | 255.255.255.255 | Request          | 53, 54, 50, 55           |
| 10.0.0.3      | 10.0.0.50       | ACK              | 53, 54, 51, 1, 28, 3, 15 |



**Ejercicio 4 [Router, VM1].** Observar que después de la configuración hay una solicitud y respuesta de `ECHO`. Determinar quién realiza la solicitud y cuál es su propósito.


  El echo se envía desde el router hacia el cliente para asegurarse de que la nueva dirección IP se ha asignado correctamente.

**Ejercicio 5 [VM4].** Durante el arranque del sistema se pueden configurar automáticamente
determinados interfaces según la información almacenada en el disco del servidor. Añadir al fichero `/etc/network/interfaces` de VM4 una entrada para que el interfaz  `eth0` se configure
automáticamente usando DHCP. Consultar la página de manual `man interfaces`.

| `/etc/network/interfaces`      |
| ------------------------------ |
| auto eth0
iface eth0 inet dhcp |


**Ejercicio 6 [VM4].** Comprobar la configuración automática con las órdenes `ifup` e  `ifdown`. Verificar la conectividad entre todas las máquinas de las dos redes.


    root@frontend:~# ifdown eth0  // Si dice que no está configurada la interfaz ignorarlo
    root@frontend:~# ifup eth0  // Se inicia el protocolo DHCP con éxito

