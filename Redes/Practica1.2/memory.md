# Ampliación Redes: Práctica 1.2

## Estados de una conexión TCP

***Ejercicio 1.*** Consultar las páginas de manual para `nc` y `netstat`. Probar algunas de las opciones para ambos programas para familiarizarse con su comportamiento.


    root@frontend:~# man nc
    root@frontend:~# man netstat

***Ejercicio 2.*** (`LISTEN`) Abrir un servidor TCP en el puerto 7777 en VM1 usando el comando `nc`. Comprobar el estado de la conexión en el servidor con el comando `netstat`.

    root@frontend:~# nc -l -p 7777
    // Desde otro terminal
    root@frontend:~# netstat -t -l
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    ...
    tcp          0        0   [::]:7777                [::]:*             LISTEN
    ...

***Ejercicio 3.*** (`ESTABLISHED`) En VM2, iniciar una conexión cliente al servidor arrancado en el ejercicio anterior.


- Comprobar el estado de la conexión e identificar los parámetros (dirección IP y puerto) usando el comando `netstat`.

    ```bash
    //Desde VM2
    root@frontend:~# netstat -t -l -a
    // No aparece especificamente, pero podemos deducir que es:
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    ...
    tcp          0        0   0.0.0.0:47206            0.0.0.0:*          LISTEN
    ...
    ```


- Reiniciar el servidor en VM1 usando la opción `-s 192.168.0.1`. Comprobar si es posible la conexión desde VM1 usando como dirección destino `localhost`. Observar la diferencia con el comando anterior (sin opción `-s`) usando `netstat`.

    ```
    // Desde VM1
    root@frontend:~# nc -l -s 192.168.0.1 -p 7777
    // Desde otro terminal
    root@frontend:~# netstat -t -l
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    ...
    tcp          0        0   192.168.0.1:7777         [::]:*             LISTEN
    ...
    
    root@frontend:~# nc localhost 7777    // No es posible conectarse
    root@frontend:~# nc 192.168.0.1 7777  // Conexión establecida
    ```

- Iniciar el servidor e intercambiar un único carácter con el cliente. Con ayuda de `wireshark`, observar los mensajes intercambiados (especialmente los números de secuencia, confirmación y flags TCP) y determinar cuántos bytes (y número de mensajes) han sido necesarios.

    ```
    // Desde VM2
    root@frontend:~# nc 192.168.0.1 7777  // Conectado
    a
    ```

    | **Origen**  | **Destino** | **Protocolo** | **Seq**    | **Ack**    | **Flags TCP** | **Data** |
    | ----------- | ----------- | ------------- | ---------- | ---------- | ------------- | -------- |
    | 192.168.0.2 | 192.168.0.1 | TCP           | 1311808192 | 3017975394 | PSH, ACK      | 2 bytes  |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 3017975394 | 1311808194 | ACK           | no data  |


***Ejercicio 4.*** (`TIMEWAIT`) Cerrar la conexión en el servidor (con Ctl+C) y comprobar el estado de la conexión con `netstat`. Usar la opción -o para determinar el valor del temporizador `TIMEWAIT`.


    // Tras cerrar conexión
    root@frontend:~# netstat -l -t -a -n -o
    Proto  Recv-Q  Send-Q  Local Address      Foreign Address   State     Timer
    ...
    tcp         0       0  192.168.0.2:47207  192.168.0.1:7777  TIME_WAIT timewait (52,75/0/0)
    ...
    // El temporizador espera un minuto

***Ejercicio 5.*** (`SYN-SENT` y `SYN-RCVD`) El comando `iptables` permite filtrar paquetes según los flags TCP del segmento (opción `--tcp-flags`):


- Fijar una regla que permita filtrar las conexiones en el servidor (VM1) de forma que deje al cliente en el estado `SYN-SENT`. Comprobar el resultado con el comando `netstat` en el cliente (VM2). 

    ```
    // Desde VM1
    root@frontend:~# iptables -A INPUT -p tcp --syn -j DROP
    root@frontend:~# nc -l -s 192.168.0.1 -p 7777
    
    // Desde VM2
    root@frontend:~# nc 192.168.0.1 7777
    // Desde otro terminal
    root@frontend:~# netstat -l -t -a -n
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    ...
    tcp          0        1   192.168.0.2:47209        192.168.0.1:7777   SYN_SENT
    ...
    ```

- Fijar una regla que permita filtrar las conexiones en el cliente (VM2) de forma que deje al servidor en el estado `SYN-RCVD`. Además esta regla debe dejar al servidor también en el estado `LAST-ACK` después de cerrar la conexión (con `Ctrl+C`) en el cliente.

    ```
    // Desde VM1
    root@frontend:~# iptables -D INPUT 1  // Eliminar regla anterior
    root@frontend:~# nc -l 7777
    
    // Desde VM2
    root@frontend:~# iptables -A OUTPUT -p tcp --tcp-flags ALL ACK -j DROP
    root@frontend:~# nc 192.168.0.1 7777
    
    // Desde VM1
    root@frontend:~# netstat -t
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    tcp     0        0        192.168.0.1:7777         192.168.0.2:59523  SYN_RECV
    // Después de Ctrl+C en VM2:
    Proto   Recv-Q   Send-Q   Local Address            Foreign Address    State
    tcp     0        0        192.168.0.1:7777         192.168.0.2:59523  LAST_ACK
    ```

- Usando el programa `wireshark`, estudiar los mensajes intercambiados especialmente los números de secuencia , confirmación y flags TCP.

    | **Origen**  | **Destino** | **Protocolo** | **Seq**    | **Ack**    | **Flags TCP** |
    | ----------- | ----------- | ------------- | ---------- | ---------- | ------------- |
    | 192.168.0.2 | 192.168.0.1 | TCP           | 1485526947 |            | SYN           |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913081 | 1485526948 | SYN, ACK      |
    | 192.168.0.2 | 192.168.0.1 | TCP           | 1485526975 | 1373913082 | FIN, ACK      |
    | 192.168.0.1 | 192.168.0.2 | TCP           | 1373913082 |            | RST           |



- Con ayuda de `netstat` (usando la opción `-o`) determinar cuántas retransmisiones se realizan y con qué frecuencia.


    Se realizan 6 retransmisiones, con frecuencia ascendente.
  

**Nota:** La regla debe ser lo suficientemente restrictiva para afectar sólo a las conexiones al servidor. Después de cada ejercicio eliminar las reglas de filtrado.

***Ejercicio 6.*** Finalmente, intentar una conexión a un puerto cerrado del servidor (ej. 7778) y, con ayuda de la herramienta `wireshark`, observar los mensajes TCP intercambiados, especialmente los flags TCP.

| **Origen**  | **Destino** | **Protocolo** | **Seq**    | **Ack**    | **Flags TCP** |
| ----------- | ----------- | ------------- | ---------- | ---------- | ------------- |
| 192.168.0.2 | 192.168.0.1 | TCP           | 345076499  |            | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 0          | 345076500  | RST, ACK      |
| 192.168.0.1 | 192.168.0.2 | TCP           | 2368399191 | 3181240654 | FIN, ACK      |
| 192.168.0.1 | 192.168.0.2 | TCP           | 2368399191 | 3181240654 | FIN, ACK      |
| 192.168.0.1 | 192.168.0.2 | TCP           | 2368399191 | 3181240654 | FIN, ACK      |



## Introducción a la seguridad en el protocolo TCP

***Ejercicio 1.*** El ataque SYN *flood* consiste en saturar un servidor por el envío masivo de mensajes con el *flag* SYN. 

- (Cliente VM2) Para evitar que el atacante responda al mensaje SYN+ACK del servidor con un mensaje RST que liberaría los recursos, bloquearemos los mensajes SYN+ACK en el atacante con `iptables`.

    ```
    root@frontend:~# iptables -A INPUT -p tcp --tcp-flags ALL SYN,ACK -j DROP
    ```


- (Cliente VM2) Para enviar paquetes TCP con los datos de interés usaremos el comando `hping3` (estudiar la página de manual). En este caso, enviaremos mensajes SYN al puerto 23 del servidor (`telnet`) lo más rápido posible (*flood*).

    ```
    root@frontend:~# hping3 --flood --syn -p 23 192.168.0.1
    ```


- (Servidor VM1) Estudiar el comportamiento de la máquina, en términos del número de paquetes recibidos. Comprobar si es posible la conexión al servicio `telnet (23)`.


    El volumen de paquetes es tan grande que wireshark deja de funcionar. Si desde otra  máquina (por ejemplo, VM3) intentamos conectarnos a la VM1, no es posible hacerlo.

    ```
    // Desde VM3
    root@frontend:~# telnet 192.168.0.1
    Trying 192.168.0.1 ...
    telnet: Unable to connect to remote host: Connection timmed out.
    ```

- (Servidor VM1) Activar la defensa SYN *cookies* en el servidor con el comando sysctl (parámetro `net.ipv4.tcp_syncookies`). Comprobar si es posible la conexión al servicio `telnet (23)`.

    ```
    // Desde VM1
    root@frontend:~# sysctl net.ipv4.tcp_syncookies=1
    
    // Desde VM3
    root@frontend:~# telnet 192.168.0.1
    Trying 192.168.0.1 ...
    Connected to 192.168.0.1.
    ```

***Ejercicio 2.*** ****(Técnica `CONNECT`) Netcat permite explorar puertos usando la técnica `CONNECT` que intenta establecer una conexión a un puerto determinado. En función de la respuesta (SYN+ACK o RST), es posible determinar si hay un proceso escuchando.

- (Servidor VM1) Abrir un servidor en el puerto 7777.

    ```
    root@frontend:~# nc -l -p 7777
    ```


- (Cliente VM2) Explorar de uno en uno, el rango de puertos 7775-7780 usando nc, en este caso usar las opciones de exploración (`-z`) y de salida detallada (`-v`). **Nota:** La versión de `nc` instalada en la VM no soporta rangos de puertos. Por tanto, se debe hacer manualmente, o bien, incluir la sentencia de exploración de un puerto en un bucle para automatizar el proceso.

    ```
    root@frontend:~# nc -z -v 192.168.0.1 7775  // Misma respuesta con todos menos 7777
    nc: inverse lookup failed for 192.168.0.1: Fallo temporal en la resolución del nombre
    nc: cannot connect to 192.168.0.1 7775: Conexión rehusada
    nc: unable to connect to address 192.168.0.1, service 7775
    root@frontend:~# nc -z -v 192.168.0.1 7777
    nc: inverse lookup failed for 192.168.0.1: Fallo temporal en la resolución del nombre
    nc: 192.168.0.1 7777 open
    ```


- Con ayuda de `wireshark` observar los paquetes intercambiados.

| **Origen**  | **Destino** | **Protocolo** | **Seq** | **Ack** | **Flags TCP** |
| ----------- | ----------- | ------------- | ------- | ------- | ------------- |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 1       | RST, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 1       | RST, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 0       | 1       | SYN, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 1       | 1       | ACK           |
| 192.168.0.2 | 192.168.0.1 | TCP           | 1       | 1       | FIN, ACK      |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 2       | FIN, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 2       | 2       | ACK           |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 1       | RST, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 1       | RST, ACK      |
| 192.168.0.2 | 192.168.0.1 | TCP           | 0       |         | SYN           |
| 192.168.0.1 | 192.168.0.2 | TCP           | 1       | 1       | RST, ACK      |


**Opcional.** La herramienta `nmap` permite realizar diferentes tipos de exploración de puertos, que emplean estrategias más eficientes. Estas estrategias (SYN *stealth*, ACK *stealth*, FIN-ACK *stealth*…) son más rápidas que la anterior y se basan en el funcionamiento del protocolo TCP. Estudiar la página de manual de `nmap` (`PORT SCANNING TECHNIQUES`) y emplearlas para explorar los puertos del servidor.  Comprobar con `wireshark` los mensajes intercambiados.


## Opciones y parámetros TCP

El comportamiento de la conexión TCP se puede controlar con varias opciones que se incluyen en la cabecera en los mensajes SYN y que son configurables en el sistema operativo. La siguiente tabla incluye alguna de las opciones y las variables asociadas del kernel:

| **Opción TCP**         | **Parámetro kernel**          | **Propósito**                                                                                                                          | **Valor por defecto**              |
| ---------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Escalado de la ventana | `net.ipv4.tcp_window_scaling` | Permite aumentar el tamaño de la ventana, si los dos extremos lo soportan.                                                             | Activado                           |
| Marcas de tiempo       | `net.ipv4.tcp_timestamps`     | Permiten identificar, sin apenas coste, cuándo se generó el mensaje.                                                                   | Activado, con offset aleatorio (1) |
| ACKs selectivos        | `net.ipv4.tcp_sack`           | En caso de que paquetes se pierdan, sólo será necesario retransmitir los perdidos, en lugar de toda la secuencia a partir del perdido. | Activado                           |

***Ejercicio 1.*** Con ayuda del comando `sysctl` y la bibliografía recomendada completar la tabla anterior.
***Ejercicio 2.*** Abrir el servidor en el puerto 7777 y realizar una conexión desde la VM cliente. Con ayuda de `wireshark` estudiar el valor de las opciones que se intercambian durante la conexión. Variar algunos de los parámetros anteriores (ej. no usar ACKs selectivos) y observar el resultado en una nueva conexión.

Los siguientes parámetros permiten configurar el temporizador *keepalive*:

| **Parámetro kernel**            | **Propósito**                                                                     | **Valor por defecto**   |
| ------------------------------- | --------------------------------------------------------------------------------- | ----------------------- |
| `net.ipv4.tcp_keepalive_time`   | Tiempo a partir del cual se empiezan a mandar señales keep_alive                  | 7200 segundos (2 horas) |
| `net.ipv4.tcp_keepalive_probes` | Número máximo de señales keep_alive que se enviarán antes de terminar la conexión | 9                       |
| `net.ipv4.tcp_keepalive_intvl`  | Número de segundos entre las señales keep_alive                                   | 75                      |

***Ejercicio 3.*** Con ayuda del comando `sysctl` y la bibliografía recomendada completar la tabla anterior.


## Traducción de direcciones (NAT) y reenvío de puertos (port forwarding)

En esta sección supondremos que la red que conecta Router (VM3) con VM4 es pública y que no puede encaminar el tráfico `192.168.0.0/24`. Además, asumiremos que la IP de Router es dinámica.
***Ejercicio 1.*** Configurar la traducción de direcciones dinámica en Router:

- (VM3 - Router) Configurar Router para que haga SNAT (*masquerade*) sobre la interfaz `eth1` usando el comando `iptables`.

    ```
    root@frontend:~# iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
    ```

- (VM1) Comprobar la conexión entre  VM1 y VM4 con la orden `ping`.

    ```
    root@frontend:~# ping 172.16.0.1
    ```

- (VM4 y VM1) Usando `wireshark`, determinar la IP origen y destino de los ICMP de Echo request y Echo reply en ambas redes. ¿Qué parámetro se utiliza, en lugar del puerto origen, para relacionar las solicitudes de Echo con las respuestas? Comprueba el fichero `/proc/net/nf_conntrack`.


    En el caso de VM1, las direcciones de origen y de destino son las reales, pero para VM4 la dirección de origen y destino de los paquetes es la del router (172.16.0.3).

***Ejercicio 2.*** Acceso a un servidor en la red privada:

- (VM1) Arrancar el servidor con nc en el puerto 7777.

    ```
    root@frontend:~# nc -l -p 7777
    ```

- (VM3 - Router) Usando el comando iptables reenviar las conexiones al puerto 80 de Router al puerto 7777 de VM1.

    ```
    root@frontend:~# iptables -t nat -A PREROUTING -d 172.16.0.3 -p tcp --dport 80 -j DNAT --to 192.168.0.1:7777
    ```

- (VM4) Conectarse al puerto 80 de Router con nc y comprobar el resultado en VM1. Analizar el tráfico intercambiado con wireshark, especialmente los puertos y direcciones IP origen y destino en ambas redes.


    Los mensajes llegan a la VM1. VM4 piensa que está hablando con VM3, pero de nuevo VM1 ve el origen real del mensaje, y ve su dirección como destinatario.

