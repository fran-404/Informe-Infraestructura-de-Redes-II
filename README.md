# Informe Infraestructura de Redes II

## Introducción
En este documento se realiza un informe en base a fragmentos del archivo ``setup.sh`` de la práctica publicada el día **23/09/2026**.  
En el desarrollo del mismo se responderán a las consignas y se explicará el funcionamiento de algunas de las funciones del archivo ``docker-compose.yml``.


## **Servicios dentro del docker-compose**

### MySQL

El primer servicio visible dentro del archivo YAML es MySQL:

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

En la sintaxis del docker-compose se indican a través del mapeo distintos parámetros, como que imagen de MySQL utilizar (en este caso la más actual) o las redes a las que debe conectarse usando una lista.  
El valor ```restart: always``` le indica a Docker que el servicio debe iniciarse sin excepción después de reiniciar el programa, el dispositivo o en caso de algún error como un crash.

Además, se puede apreciar una variable de entorno, las cuales son utilizadas para otorgar diversos valores de configuración a las aplicaciones o servicios que correrán dentro del container.  
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

[información sobre esta función](https://dev.mysql.com/blog-archive/docker-secrets-and-mysql-password-management/)

---

### DBExporter

DBExporter o MySQLDExporter es un servicio utilizado para exportar datos de métricas de servidores MySQL extraidos con Prometheus.

La sintaxis YAML es similar a la de MySQL, con la siguiente adición:

```
 volumes:
      - ./my.cnf:/.my.cnf:ro
    command:
      - --config.my-cnf=/.my.cnf
    depends_on:
      - mysql
```

Se distingue el valor "volumes" o volúmenes, función utilizada y gestionada por los containers para almacenar datos. 

Los volúmenes se alojan en un directorio especificado en el dispositivo del host y son utilizados cuando se generan datos persistentes del container.
Pueden ser útiles para almacenamiento de dichos datos debido a que no incrementan el uso de espacio de los containers y son veloces.

[información sobre vólumenes](https://docs.docker.com/engine/storage/volumes/#when-to-use-volumes)

También se puede destacar que en este caso el archivo ```my.cnf``` esta configurado como solo lectura.  
Montarlo como solo lectura previene modificaciones indeseadas que pueden comprometer el funcionamiento del servicio.  

Para algunas aplicaciones es necesario escribir dentro del volumen, mientras que otras no lo requieren, leer los volúmenes es suficiente. 


Por último, la línea ```depends_on:``` del código establece el orden en el que los servicios deben arrancar o detenerse, en esta ocasión, se indica que DBExporter iniciará unicamente después de que inicie MySQL (esto no significa que esté funcionando).

---

### CAdvisor
<img src="https://github.com/user-attachments/assets/b810d1f4-7752-4f67-95fe-8b5f053831d2" alt="Vista de CAdvisor" width="600" height="300">

CAdvisor, por su nombre completo Container Advisor, es un servicio que provee a los usuarios una vista de la utilización de recursos de un container, extrayendo la información y exponiéndola con estadísticas.  
También posee una interfaz web que puede ser accedida mediante un navegador.

Sintaxis:
```
     cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    restart: always
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    networks:
      - mynetwork
```


En la sintaxis YAML se puede distinguir que las rutas están configuradas como solo lectura, para que una vez activado el servicio "descubra" la jerarquía de los grupos de control (cgroups) de  los demás containers y comience a recolectar la información sin realizar modificaciones. 


---

### Prometheus
Creado originalmente en 2012 por la empresa de retransmisión de música SoundCloud, Prometheus es un sistema que recolecta métricas mediante un "scraping" de endpoints HTTPS en distintos servicios. 

Prometheus se encuentra tanto en la configuración del docker-compose.yml como por fuera, con un archivo ``prometheus.yml`` propio, este último se ubica en el dispositivo del host. 


#### *docker-compose.yml*
```
 prometheus:
    image: prom/prometheus:latest
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - mynetwork
```

En los volúmenes se puede observar que se monta el archivo ``prometheus.yml`` mencionado, esto es para cargar las configuraciones de Prometheus preestablecidas.  
Además, en esta configuración se puede observar la línea ```extra_hosts```, la cual es utilizada para que un container pueda "alcanzar" servicios que están corriendo en la computadora del host, en este caso la aplicación Flask.

#### *prometheus.yml*
```
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'mysql'
    static_configs:
      - targets: ['dbexporter:9104']

  - job_name: 'crud-app'
    static_configs:
      - targets: ['host.docker.internal:8888']
```


En el código se puede distinguir ```scrape_configs:``` que estipula las configuraciones para cada servicio objetivo (MySQL, DBExporter, etc) indicando el puerto al que debe prestar atención. 

En la línea ```scrape_interval: 5s``` se establece que recolectará métricas de los servicios en intervalos de 5 segundos.

Por defecto, Prometheus escucha el puerto 9090 para conexiones entrantes. Este puerto se utiliza para acceder a la web y es donde otros servicios se conectan para "scrapear" métricas.  
De esta forma, también se puede acceder a Prometheus desde el sistema operativo del host, por fuera de Docker.

El puerto puede ser cambiado si se desea, para evitar conflictos con otros servicios u otros motivos.


---

### Grafana
<img src="https://github.com/user-attachments/assets/3611b28a-6b1d-42f6-913e-9b0cd5cf4b16" alt="Vista de Grafana" width="600" height="300">

Grafana es un software libre que permite la visualización y el formato de datos métricos con gráficos y cuadros de mando.  
Es utilizado para monitorear en vivo el estado de las aplicaciones de un servidor.  
Por si solo, Grafana no recolecta los datos, por lo que depende de otras fuentes como Prometheus para leer las métricas recolectadas y mostrarlas.  

```
grafana:
    image: grafana/grafana:13.2.2
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    networks:
      - mynetwork
```
Analizando el código se puede destacar que usa una imagen especifica (para mantener la consistencia en la práctica) y una variable de entorno para establecer la contraseña por defecto como "admin" , esto es para poder iniciar sesión en la aplicación web con esa credencial.  
Luego de iniciar sesión, Grafana le sugiere al usuario cambiar la contraseña, pero no es obligatorio.  


También es visible nuevamente el atributo ```depends_on``` indicando que debe iniciar luego de Prometheus.


## Acerca de la red.

En todos los containers se utiliza la misma red, "mynetwork" y se comunican entre sí por sus nombres, esto es gracias a que a partir de Docker 1.10 se incluye un DNS integrado, el cual permite que los containers conectados a la misma red se puedan referenciar de esta forma, en lugar de usar sus direcciones IP.

[Fuente](https://stackoverflow.com/questions/31149501/how-to-reach-docker-containers-by-name-instead-of-ip-address)

---
**Fin del informe.**
