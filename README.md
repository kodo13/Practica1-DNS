# Práctica 1- DNS

## Requisitos de configuración 

### Apartados

1. <b>Volumen por separado de la configuración</b>

Primero tenemos que crear dos carpetas, conf y zonas, en la que tendremos las configuraciones de estas.
Creamos un archivo named.conf y en este llamamos a las configuraciones haciendo un include.

```
include "/etc/bind/named.conf.options"; #Llamada al fichero que contiene la configuración 
#global de opciones del DNS.
include "/etc/bind/named.conf.local"; #Llamada al fichero que contiene la configuración 
#de nuestras zonas.
```

Para tener el volumen por separado, tenemos que declarar el volumen en el archivo docker-compose.yml. Lo mapeamos de la carpeta donde tengamos el contenedor con la configuración de este.

2. <b>Red propia interna para todos los contenedores</b>

Para tener una red propia interna, tenemos que crear nosotros una red, que lo haremos con el siguiente comando: 
`docker network create --subnet 10.0.3.0/24 --gateway 10.0.3.1 p1_subnet`

3. <b>Ip fija en el servidor</b>

Para añadir Ip fija, añadimos la línea de ipv4_address y la IP.
`ipv4_address: 10.0.3.254`

5. <b>Configurar Forwarders</b>

Los forwarders los utilizamos para poder resolver cuando nuestro servidor no es capaz de hacerlo.

```
 forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        forward only;
```

6. <b>Crear Zona propia </b>
    1. Registros a configurar: NS, A, CNAME, TXT, SOA

```
$TTL 38400	; 10 hours 40 minutes
@		IN SOA	ns.ionut.int. root.email.com. (
				1910202201   ; serial
				10800      ; refresh (3 hours)
				3600       ; retry (1 hour)
				604800     ; expire (1 week)
				38400      ; minimum (10 hours 40 minutes)
				)
@		IN NS	ns.ionut.int.
ns		IN A 	10.0.3.254
test	IN A	10.0.3.15
alias	IN A    10.11.12.13
alias	IN TXT   mensaje
ionu    IN CNAME 10.0.3.115
```
7. <b>Cliente con herramientas de red</b>

El cliente ya tiene herramienta de ping para poder hacer la comprobacion.

## Creación de servicios (contenedor)
Para crear un contenedor, tenemos que añadir un nuevo servicio, en este caso, bind9.
Le damos un nombre a dicho contenedor y añadimos la imagen. Muy importante añadir la versión si no es muy probable que nos salte un error.
Creamos los volúmenes y asignamos la network que habíamos creado anteriormente, dándole una Ip fija de esta red.
```
bind9:
    container_name: p1_serverDNS #nombre del contenedor
    image: internetsystemsconsortium/bind9:9.16 #Ojo, a la imagen hay que ponerle la versión
    volumes:
      - ./conf:/etc/bind
      - ./zonas:/var/lib/bind
    networks:
      p1_subnet:
        ipv4_address: 10.0.3.254
```

## Líneas docker-compose explicadas

```
version: "3.9"
services:
  bind9:
    container_name: p1_serverDNS #nombre del contenedor
    image: internetsystemsconsortium/bind9:9.16 #Ojo, a la imagen hay que ponerle la versión
    volumes:
      - ./conf:/etc/bind 
      - ./zonas:/var/lib/bind
    networks:
      p1_subnet: #Red propia interna creada anteriormente.
        ipv4_address: 10.0.3.254 #Dirección IP fija de nuestro servidor DNS.
  p1_cliente:  #Creación del cliente
    container_name: p1_cliente
    image: alpine
    networks: 
      - p1_subnet
    stdin_open: true      # Línea equivalente a docker run -i 
    tty: true             # docker run -t , opciones para que se nos abra un nuevo ter
    dns: 
      - 10.0.3.254        #el contenedor dns server
networks:
 p1_subnet: 
  external: true

```

## Modificación de la configuración, arranque y parada de servicio bind9

Para arrancar el servicio, tenemos que ejecutar desde la misma carpeta en la que tenemos toda la configuración, el comando docker-compose up para levantar el servicio.
Se nos crean los contenedores, los volumenes y se arranca el servicio.

Si hacemos alguna modificación, es muy recomendable variar el serial del archivo de las zonas para luego poder ver si se ejecutaron los cambios correctamente.

Para la parada del servicio, ejecutaremos `docker-compose down -v`. El -v es para que se eliminen los contenedores asociados al servicio.

## Configuración zona y comprobar que funciona.

Para la configuración de la zona, tenemos que crear un archivo db.nuestrodominio.ejemplo, db.ionut.int en este caso.
Aquí configuraremos todos los registros, tales como ns, que es el dominio de nuestro servidor, un alias si queremos, un cliente de correo electrónico MX...etc.

Para comprobar que funciona, una vez arrancado los servicios, tenemos que ir al cliente y abrir una terminal.
![(Imagen)](https://github.com/kodo13/Practica1-DNS/blob/master/Capturas/Captura%20desde%202022-10-26%2000-05-19.png?raw=true)
> Apertura de shell del cliente
De esta manera, podremos hacer un ping al servidor y ver si obtenemos respuesta.
![(Imagen)](https://github.com/kodo13/Practica1-DNS/blob/master/Capturas/Captura%20desde%202022-10-26%2000-03-11.png?raw=true)
> Comprobación de ping a la Ip del servidor.

también podemos hacer ping al dominio que tenemos y ver si nos resuelve con la IP.
![(Imagen)](https://github.com/kodo13/Practica1-DNS/blob/master/Capturas/Captura%20desde%202022-10-26%2000-06-49.png?raw=true)
> Vemos que haciendo ping al dominio, nos resuleve a la Ip fija del servidor.
