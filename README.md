# Informe grupal — Infraestructura de Redes
## Etapa 6: Monitoreo con Docker Compose, Prometheus y Grafana

## Introducción

El objetivo de este informe es documentar y explicar la infraestructura de monitoreo utilizada en la Etapa 6, tomando como referencia la configuración real del archivo `docker-compose.yml` y los archivos auxiliares del proyecto.

La infraestructura está compuesta por cinco servicios principales ejecutados mediante Docker Compose: MySQL, MySQL Exporter (`dbexporter`), cAdvisor, Prometheus y Grafana. Además, la aplicación CRUD desarrollada en Flask se ejecuta en el host y expone métricas para que Prometheus pueda recolectarlas.

Cada servicio cumple una función específica. MySQL administra los datos de la aplicación; MySQL Exporter obtiene métricas relacionadas con la base de datos; cAdvisor recopila información sobre los contenedores; Prometheus almacena las métricas recolectadas; y Grafana permite consultarlas y representarlas visualmente.

El informe se organiza en tres partes: configuración de MySQL y `dbexporter`, recolección de métricas con cAdvisor y Prometheus, y visualización con Grafana y la red común. En cada sección se explican las funciones de los servicios y los elementos de sintaxis YAML utilizados. Finalmente, se relacionan los componentes para comprender cómo funciona la infraestructura en conjunto.

## Parte 1 — Persistencia de datos: MySQL y dbexporter

### 1.1. Servicio MySQL

El servicio `mysql` utiliza la imagen `mysql:latest` y tiene configurada la política `restart: always`. Esta política indica que Docker debe intentar reiniciar el contenedor si se detiene, de acuerdo con el comportamiento de la política de reinicio.

La configuración incluye la variable de entorno `MYSQL_ROOT_PASSWORD`, que establece la contraseña del usuario administrador de MySQL. En el archivo utilizado para esta práctica, el valor configurado es `mysecretpassword`.

La sección `networks` conecta el servicio a la red `mynetwork`. De esta manera, los demás servicios conectados a esa misma red pueden comunicarse con MySQL utilizando el nombre del servicio como dirección de destino.

MySQL no tiene un puerto publicado en el host dentro del archivo `docker-compose.yml` analizado. Por lo tanto, la base de datos está destinada a ser utilizada desde los servicios que se comunican a través de la red interna de Docker.

### 1.2. Configuración de la base de datos

El script de instalación prepara la base de datos `alumnos` y crea una tabla del mismo nombre con los campos `id`, `nombre`, `apellido` y `fecha_nacimiento`. También carga registros iniciales para disponer de datos con los que probar la aplicación.

Aunque esta parte se relaciona con la persistencia de datos, es importante distinguir entre almacenar información en MySQL y configurar un mecanismo explícito para conservarla. En el `docker-compose.yml` analizado no se declara un volumen específico para los datos de MySQL. Por ese motivo, no corresponde afirmar que existe persistencia garantizada mediante volúmenes en esta configuración.

### 1.3. Servicio dbexporter

El servicio `dbexporter` utiliza la imagen `prom/mysqld-exporter:latest`. Su función es consultar información de MySQL y exponer métricas que Prometheus puede recolectar.

La directiva `depends_on` establece que Docker Compose debe iniciar el servicio `mysql` antes de iniciar `dbexporter`. Esto expresa una dependencia de inicio, aunque no garantiza por sí solo que MySQL ya esté preparado para aceptar conexiones.

El servicio utiliza el archivo `my.cnf`, montado mediante la siguiente configuración:

`./my.cnf:/.my.cnf:ro`

La ruta de la izquierda corresponde al archivo del proyecto en el host y la de la derecha indica dónde estará disponible dentro del contenedor. El sufijo `ro` significa *read-only* o solo lectura. De este modo, el contenedor puede leer la configuración sin modificar el archivo original.

El archivo `my.cnf` contiene los parámetros de conexión utilizados por el exportador: usuario, contraseña, host y puerto. En esta práctica, el host configurado es `mysql`, que coincide con el nombre del servicio en la red compartida.

La opción `command` indica al exportador que utilice ese archivo de configuración:

`--config.my-cnf=/.my.cnf`

Así, `dbexporter` dispone de los datos necesarios para conectarse a MySQL y publicar sus métricas.

### 1.4. Sintaxis YAML aplicada

- **`services`** agrupa los contenedores que forman parte de la infraestructura.
- **`image`** indica la imagen desde la que se crea cada contenedor.
- **`environment`** define variables de entorno utilizadas por el servicio.
- **`volumes`** permite montar archivos o directorios del host dentro del contenedor.
- **`depends_on`** declara una dependencia entre servicios.
- **`command`** especifica el comando o los argumentos que utilizará el contenedor.
- **`networks`** indica a qué red de Docker se conecta cada servicio.
- **`restart: always`** establece una política de reinicio del contenedor.

En YAML, la indentación determina la relación entre las claves y sus valores. Por eso es importante mantener los espacios y la estructura jerárquica del archivo.

## Parte 2 — Recolección de métricas: cAdvisor y Prometheus

### 2.1. Servicio cAdvisor

El servicio `cadvisor` utiliza la imagen `gcr.io/cadvisor/cadvisor:v0.47.2`. Su función es recopilar información sobre el uso de recursos y el funcionamiento de los contenedores, como el consumo de CPU y memoria.

Para acceder a la información que necesita del host, tiene configurados los siguientes montajes:

- `/:/rootfs:ro`
- `/var/run:/var/run:ro`
- `/sys:/sys:ro`
- `/var/lib/docker/:/var/lib/docker:ro`
- `/dev/disk/:/dev/disk:ro`

Estos montajes permiten que cAdvisor consulte distintas partes del sistema anfitrión. El sufijo `ro` indica que se montan en modo de solo lectura, lo que evita que cAdvisor modifique esos recursos a través de dichos montajes.

El servicio está conectado a `mynetwork`, por lo que Prometheus puede acceder a sus métricas utilizando el nombre `cadvisor` y el puerto `8080`, que es el puerto de métricas configurado por cAdvisor en esta práctica.

### 2.2. Servicio Prometheus

Prometheus utiliza la imagen `prom/prometheus:latest`. Es el componente encargado de consultar los destinos configurados, recolectar las métricas y almacenarlas para su posterior consulta y visualización.

En `docker-compose.yml` se publica el puerto mediante:

`"9090:9090"`

En esta sintaxis, el primer número corresponde al puerto del host y el segundo al puerto del contenedor. Esto permite acceder a la interfaz de Prometheus desde el host mediante el puerto `9090`.

El archivo de configuración se monta mediante:

`./prometheus.yml:/etc/prometheus/prometheus.yml`

La ruta de la izquierda identifica el archivo ubicado en el proyecto y la de la derecha indica su ubicación dentro del contenedor. Este archivo define la frecuencia de recolección y los destinos que Prometheus consulta.

### 2.3. Configuración de prometheus.yml

El archivo `prometheus.yml` establece un intervalo global de recolección de cinco segundos mediante `scrape_interval: 5s`.

También define cuatro destinos de métricas:

- **prometheus:** `localhost:9090`, para consultar las métricas del propio Prometheus.
- **cadvisor:** `cadvisor:8080`, para recolectar métricas de los contenedores.
- **mysql:** `dbexporter:9104`, para obtener las métricas de MySQL expuestas por el exportador.
- **crud-app:** `host.docker.internal:8888`, para consultar las métricas de la aplicación Flask que se ejecuta en el host.

Cada destino está organizado dentro de un trabajo identificado por `job_name`, con una sección `static_configs` y una lista `targets`. Esta estructura permite agrupar y reconocer las distintas fuentes de información.

### 2.4. ¿Para qué sirve extra_hosts?

En el servicio Prometheus aparece la configuración:

`host.docker.internal:host-gateway`

La directiva `extra_hosts` agrega una entrada al archivo de resolución de nombres del contenedor. En este caso, permite que `host.docker.internal` se resuelva hacia la dirección del host accesible desde Docker.

Esto resulta necesario para que Prometheus, que se ejecuta dentro de un contenedor, pueda consultar la aplicación CRUD que se ejecuta directamente en el host y expone sus métricas en el puerto `8888`.

La configuración de `extra_hosts` no publica por sí sola el puerto de la aplicación: la aplicación debe estar ejecutándose y ser accesible desde el contenedor.

### 2.5. Sintaxis YAML aplicada

- **`ports`** publica puertos del contenedor en el host mediante la relación `host:contenedor`.
- **`volumes`** permite proporcionar archivos de configuración al contenedor.
- **`extra_hosts`** agrega nombres de host que el contenedor puede resolver.
- **`global`** define opciones generales de Prometheus, como el intervalo de recolección.
- **`scrape_configs`** agrupa los trabajos de recolección de métricas.
- **`job_name`** identifica cada trabajo.
- **`static_configs`** permite declarar destinos de forma estática.
- **`targets`** contiene las direcciones de los servicios o aplicaciones que se consultan.

La indentación de YAML permite representar esta estructura jerárquica. Los guiones indican elementos de una lista, como los trabajos y sus destinos.

## Parte 3 — Visualización y topología: Grafana y mynetwork

### 3.1. Servicio Grafana

El servicio `grafana` utiliza la imagen `grafana/grafana:13.2.2`. La versión está fijada en el archivo de configuración para mantener una versión concreta durante la práctica.

La directiva `ports` publica el puerto `3000` del contenedor en el puerto `3000` del host. Esto permite acceder a la interfaz web de Grafana.

La configuración también establece la variable `GF_SECURITY_ADMIN_PASSWORD` con el valor `admin`, que corresponde a la contraseña de administrador configurada en este entorno de práctica.

La directiva `depends_on` declara una dependencia respecto del servicio `prometheus`. Docker Compose inicia Prometheus antes de Grafana, pero esto no garantiza por sí solo que Prometheus esté completamente preparado para recibir consultas.

### 3.2. Función de Grafana

Grafana permite crear paneles y gráficos a partir de las métricas consultadas en Prometheus. Mientras Prometheus se ocupa de recolectar y almacenar las series temporales, Grafana proporciona una interfaz visual para interpretarlas.

Para utilizar las métricas, Grafana debe configurarse para consultar Prometheus como fuente de datos. La declaración de ambos servicios en el archivo Compose y su conexión a una red común facilita la comunicación entre ellos, aunque la fuente de datos debe estar configurada en Grafana.

### 3.3. Red compartida mynetwork

Al final del archivo `docker-compose.yml` se declara la red `mynetwork`. Los cinco servicios —`mysql`, `dbexporter`, `cadvisor`, `prometheus` y `grafana`— están conectados a ella.

Esta red permite que los contenedores se comuniquen mediante los nombres de servicio, sin tener que configurar manualmente una dirección IP fija para cada uno. Por ejemplo, `dbexporter` puede conectarse a `mysql`, y Prometheus puede consultar a `cadvisor` y `dbexporter` utilizando sus respectivos nombres.

La red común facilita la organización de la infraestructura y permite separar la comunicación interna de los puertos que se publican en el host. En esta configuración, los puertos publicados son `9090` para Prometheus y `3000` para Grafana.

### 3.4. Sintaxis YAML aplicada

- **`networks`** declara las redes utilizadas por los servicios.
- **`mynetwork`** identifica la red compartida por los contenedores.
- **`depends_on`** expresa el orden de inicio entre servicios.
- **`ports`** habilita el acceso desde el host a los puertos publicados.
- **`environment`** define parámetros del servicio, como la contraseña inicial de administrador de Grafana.
- **`image`** permite fijar una versión concreta de Grafana mediante una etiqueta específica.

La combinación de servicios, redes, dependencias y puertos en YAML permite describir la infraestructura en un único archivo, facilitando su lectura y administración.

## Conclusión

El análisis del archivo `docker-compose.yml` y de los archivos auxiliares permite comprender cómo se integran los componentes de una infraestructura de monitoreo. MySQL administra los datos; `dbexporter` expone métricas de la base de datos; cAdvisor recopila información de los contenedores; Prometheus consulta y almacena las métricas; y Grafana permite representarlas visualmente.

También se identificó la importancia de la sintaxis YAML para declarar servicios, variables de entorno, montajes, puertos, redes y dependencias. La configuración de `mynetwork` facilita la comunicación interna entre los contenedores, mientras que `extra_hosts` permite que Prometheus acceda a la aplicación CRUD que se ejecuta en el host.

Por último, el trabajo permite distinguir entre declarar una dependencia de inicio y garantizar que un servicio esté listo, así como entre utilizar una base de datos y configurar explícitamente la persistencia de sus datos mediante volúmenes. Estas diferencias son importantes para interpretar correctamente una infraestructura Docker.

La elaboración conjunta del informe y el uso de commits descriptivos en Git permiten documentar las contribuciones individuales y el trabajo compartido, manteniendo el historial de cambios como evidencia del proceso realizado.
