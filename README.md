# Informe-Infraestructura-de-Redes-II
## Informe acerca del _docker-compose.yaml_ utilizado en la práctica de Grafana.


# **Servicios**

### MySQL

El primer servicio visible dentro del archivo YAML es MySql:

```
mysql:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: mysecretpassword
    networks:
      - mynetwork
```
MySQL es un sistema desarrollado por Oracle Corporation utilizado en la gestión de bases de datos relacionales. Permite almacenar datos en tablas y modificarlas mediante "Querys".

En la sintaxis del docker-compose se indica a través del mapeo distintos parámetros, como que imagen de MySQL utilizar (en este caso la más actual) o la red a la que debe conectarse usando una lista.

El fragmento ```restart: always``` le indica a Docker que el servicio debe iniciarse siempre después de reiniciar el programa, el dispositivo o en caso de algún error como un crash.

Además, se puede apreciar que la contraseña es introducida mediante una variable de entorno, las cuales son utilizadas para otorgar diversos valores de configuración a las aplicaciones o servicios que correrán dentro del container.

En este caso, la variable ```MYSQL_ROOT_PASSWORD: mysecretpassword``` es utilizada para establecer como contraseña root por defecto "mysecretpassword". Esto puede ser útil para mantener la consistencia entre containers y realizar preparativos iniciales más rápido.

Según la documentación de Docker, no es una buena práctica, debido a que la contraseña puede ser obtenida fácilmente por cualquier otro usuario usando un comando como ```cat docker-compose.yml```.

En su lugar, podría utilizarse la función de secrets de Docker.

```
environment:
MYSQL_ROOT_PASSWORD: /run/secrets/my_secret
secrets:
- my_secret

secrets:
my_secret:
file: ./my_secret.txt
```
Los "secretos" son creados en un archivo guardado en el dispositivo al que se le pueden aplicar otras medidas de seguridad.

https://dev.mysql.com/blog-archive/docker-secrets-and-mysql-password-management/

### DBExporter

DBExporter o MySQLDExporter es un servicio utilizado para exportar datos de métricas de servidores MySQL extraidos con Prometheus.

La sintaxis YAML es similar a la de MySQL con la siguiente adición:

```
 volumes:
      - ./my.cnf:/.my.cnf:ro
    command:
      - --config.my-cnf=/.my.cnf
    depends_on:
      - mysql
```

Se distingue el valor "volumes" o volúmenes, función utilizada y gestionada por los containers para almacenar datos. 

Los volúmenes se alojan en un directorio especificado por el usuario y son utilizados cuando se generan datos persistentes en el container.
Pueden ser útiles para almacenamiento de dichos datos debido a que no incrementan el uso de espacio de los containers y son rápidos.

https://docs.docker.com/engine/storage/volumes/#when-to-use-volumes

También se puede destacar que en este caso el archivo ```my.cnf``` esta configurado como solo lectura, lo cual evita que pueda ser modificado al montarse. 

Montarlo como solo lectura previene modificaciones indeseadas que pueden comprometer el funcionamiento del servicio. Para algunas aplicaciones es necesario escribir dentro del volumen, mientras que otras no lo requieren, leer los volúmenes es suficiente. 


Por último, la línea ```depends_on:``` del código establece el orden en el que los servicios deben arrancar o detenerse, en esta ocasión, se indica que DBExporter iniciará unicamente después de que MySQL esté en funcionamiento.

### CAdvisor

CAdvisor, por su nombre completo Container Advisor, es un servicio que provee a los usuarios una vista de la utilización de recursos de un container, extrayendo la información y exponiéndola con estadísticas.
. También posee una interfaz web que puede ser accedida mediante un navegador.

En la sintaxis se puede distinguir que las rutas están configuradas como como solo lectura para que una vez activado "descubra" la jerarquía de los grupos de control (cgroups) de  los demás containers y luego comience a recolectar la información sin realizar modificaciones. 
```
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
```

