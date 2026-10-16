# Informe — Infraestructura de Redes
## Etapa 6: Monitoreo con Docker Compose, Prometheus y Grafana

## Introducción

El objetivo de este informe es documentar y explicar la infraestructura de monitoreo utilizada en la Etapa 6, tomando como referencia el archivo `docker-compose.yml` y los archivos auxiliares del proyecto.

La infraestructura está compuesta por cinco servicios principales ejecutados mediante Docker Compose: MySQL, MySQL Exporter (`dbexporter`), cAdvisor, Prometheus y Grafana. Además, la aplicación CRUD desarrollada en Flask se ejecuta directamente en el host y expone métricas que Prometheus puede recolectar.

Cada componente cumple una función específica. MySQL administra los datos de la aplicación; `dbexporter` expone métricas relacionadas con la base de datos; cAdvisor recopila información sobre los recursos utilizados por los contenedores; Prometheus recolecta y almacena las métricas; y Grafana permite consultarlas y representarlas mediante paneles y gráficos.

El informe se divide en tres partes. La primera explica MySQL y `dbexporter`; la segunda aborda cAdvisor, Prometheus y la configuración de recolección; y la tercera describe Grafana y la red compartida `mynetwork`. En cada sección se analizan las funciones de los servicios y los elementos de sintaxis YAML utilizados.

## Parte 1 — Persistencia de datos: MySQL y dbexporter

### 1.1. Servicio MySQL

El servicio `mysql` utiliza la imagen `mysql:latest` y tiene configurada la política `restart: always`. Esta política indica que Docker debe intentar reiniciar el contenedor cuando se detiene.

La sección `environment` define la variable `MYSQL_ROOT_PASSWORD`, que establece la contraseña del usuario administrador de MySQL. En la configuración analizada, el valor utilizado es `mysecretpassword`.

El servicio está conectado a la red `mynetwork`. Esto permite que los demás contenedores conectados a esa red se comuniquen con MySQL utilizando el nombre del servicio como destino, sin necesidad de configurar manualmente una dirección IP.

En el archivo `docker-compose.yml` analizado, MySQL no tiene un puerto publicado en el host. Su acceso se realiza desde los servicios que pueden comunicarse a través de la red interna de Docker.

### 1.2. Base de datos y persistencia

El script de instalación prepara una base de datos llamada `alumnos` y crea una tabla del mismo nombre con los campos `id`, `nombre`, `apellido` y `fecha_nacimiento`. También incorpora registros iniciales para probar la aplicación.

Es importante distinguir entre utilizar una base de datos y configurar la persistencia de sus datos mediante volúmenes. En el archivo `docker-compose.yml` analizado no se declara un volumen específico para almacenar los datos de MySQL fuera del contenedor.

Por lo tanto, la configuración permite trabajar con la base de datos, pero no declara un mecanismo explícito de persistencia mediante volúmenes. Si el contenedor se elimina y se vuelve a crear, no debe suponerse que los datos anteriores se conservarán.

### 1.3. Servicio dbexporter

El servicio `dbexporter` utiliza la imagen `prom/mysqld-exporter:latest`. Su función es consultar información de MySQL y exponer métricas que Prometheus puede recolectar.

La directiva `depends_on` establece que el servicio `mysql` debe iniciarse antes que `dbexporter`. Sin embargo, esta dependencia determina el orden de inicio y no garantiza por sí sola que MySQL esté preparado para aceptar conexiones.

Para conectarse a la base de datos, el exportador utiliza el archivo `my.cnf`, que se monta mediante la configuración:

`./my.cnf:/.my.cnf:ro`

La ruta de la izquierda corresponde al archivo del proyecto en el host, mientras que la ruta de la derecha indica dónde estará disponible dentro del contenedor. El sufijo `ro` significa *read-only*, es decir, solo lectura. Esto permite consultar el archivo sin modificarlo desde el contenedor mediante ese montaje.

El archivo `my.cnf` contiene los parámetros de conexión utilizados en esta práctica: usuario, contraseña, host y puerto. El host configurado es `mysql`, que coincide con el nombre del servicio en la red compartida.

Además, la opción `command` indica al exportador que utilice ese archivo:

`--config.my-cnf=/.my.cnf`

De esta manera, `dbexporter` dispone de la configuración necesaria para conectarse a MySQL y exponer las métricas correspondientes.

### 1.4. Sintaxis YAML aplicada

En esta parte se utilizan distintos elementos de YAML:

- **`services`** agrupa los servicios que forman parte de la infraestructura.
- **`image`** indica la imagen que se utilizará para crear el contenedor.
- **`environment`** define variables de entorno del servicio.
- **`volumes`** permite montar archivos o directorios del host dentro del contenedor.
- **`depends_on`** establece dependencias y orden de inicio entre servicios.
- **`command`** especifica comandos o argumentos que se ejecutarán en el contenedor.
- **`networks`** indica a qué red se conecta cada servicio.
- **`restart: always`** configura una política de reinicio.

La indentación es fundamental en YAML, ya que determina la relación entre las claves y sus valores. Una estructura incorrecta puede impedir que la configuración se interprete como se espera.

## Parte 2 — Recolección de métricas: cAdvisor y Prometheus

### 2.1. Servicio cAdvisor

El servicio `cadvisor` utiliza la imagen `gcr.io/cadvisor/cadvisor:v0.47.2`. Su función es recopilar información sobre el uso de recursos de los contenedores, como el consumo de CPU y memoria.

Para obtener información del sistema anfitrión, se configuran los siguientes montajes:

- `/:/rootfs:ro`
- `/var/run:/var/run:ro`
- `/sys:/sys:ro`
- `/var/lib/docker/:/var/lib/docker:ro`
- `/dev/disk/:/dev/disk:ro`

Estos montajes permiten que cAdvisor consulte distintas partes del sistema y la información relacionada con los contenedores. El sufijo `ro` indica que los recursos se montan en modo de solo lectura, limitando las modificaciones a través de esos montajes.

El servicio está conectado a `mynetwork`. Por eso, Prometheus puede consultar sus métricas utilizando el nombre `cadvisor` y el puerto `8080`.

### 2.2. Servicio Prometheus

El servicio `prometheus` utiliza la imagen `prom/prometheus:latest`. Es el componente responsable de consultar los destinos configurados, recolectar las métricas y almacenarlas para su posterior consulta.

En el archivo `docker-compose.yml` se publica el puerto mediante:

`"9090:9090"`

El primer número corresponde al puerto del host y el segundo al puerto del contenedor. Esta configuración permite acceder a la interfaz web de Prometheus desde el host mediante el puerto `9090`.

El archivo de configuración se monta de la siguiente manera:

`./prometheus.yml:/etc/prometheus/prometheus.yml`

La primera ruta identifica el archivo del proyecto en el host y la segunda indica su ubicación dentro del contenedor. En ese archivo se definen el intervalo de recolección y los destinos que Prometheus consulta.

### 2.3. Configuración de prometheus.yml

El archivo `prometheus.yml` establece un intervalo global de recolección de cinco segundos mediante `scrape_interval: 5s`.

También define cuatro trabajos de recolección:

- **prometheus:** `localhost:9090`. Permite consultar las métricas del propio Prometheus desde su contenedor.
- **cadvisor:** `cadvisor:8080`. Permite recolectar métricas de los contenedores.
- **mysql:** `dbexporter:9104`. Permite obtener las métricas de MySQL expuestas por el exportador.
- **crud-app:** `host.docker.internal:8888`. Permite consultar las métricas de la aplicación Flask que se ejecuta en el host.

Cada trabajo se identifica mediante `job_name` y contiene una sección `static_configs`, en la que se especifican los destinos a través de `targets`.

Esta organización permite identificar las distintas fuentes de métricas y configurar qué servicios consulta Prometheus.

### 2.4. Función de extra_hosts

En la configuración del servicio Prometheus aparece:

`host.docker.internal:host-gateway`

Esta entrada se declara mediante `extra_hosts`, que permite agregar nombres de host al mecanismo de resolución de nombres del contenedor.

En este caso, `host.docker.internal` se asocia con la dirección del host accesible desde Docker. Esto permite que Prometheus consulte la aplicación CRUD que se ejecuta directamente en el host y expone sus métricas en el puerto `8888`.

La directiva `extra_hosts` no inicia la aplicación ni publica su puerto. Para que la recolección funcione, la aplicación debe estar ejecutándose y ser accesible desde el contenedor.

### 2.5. Sintaxis YAML aplicada

En esta parte se utilizan los siguientes elementos:

- **`ports`** publica puertos del contenedor en el host mediante la relación `host:contenedor`.
- **`volumes`** permite proporcionar archivos de configuración al contenedor.
- **`extra_hosts`** agrega nombres de host que el contenedor puede resolver.
- **`global`** define opciones generales de Prometheus, como el intervalo de recolección.
- **`scrape_configs`** agrupa los trabajos de recolección de métricas.
- **`job_name`** identifica cada trabajo.
- **`static_configs`** permite declarar destinos de forma estática.
- **`targets`** contiene las direcciones de los servicios o aplicaciones que se consultan.

Los guiones representan elementos de listas y la indentación establece la estructura jerárquica del archivo. Estos elementos permiten expresar la configuración de forma organizada y legible.

## Parte 3 — Visualización y topología: Grafana y mynetwork

### 3.1. Servicio Grafana

El servicio `grafana` utiliza la imagen `grafana/grafana:13.2.2`. La versión está fijada en la configuración para utilizar una versión concreta durante la práctica.

El puerto se publica mediante:

`"3000:3000"`

Esto permite acceder a la interfaz web de Grafana desde el host utilizando el puerto `3000`.

La configuración incluye la variable de entorno `GF_SECURITY_ADMIN_PASSWORD`, con el valor `admin`, que establece la contraseña del administrador para este entorno de práctica.

La directiva `depends_on` declara una dependencia respecto del servicio `prometheus`. Docker Compose inicia Prometheus antes de Grafana, pero esto no garantiza por sí solo que Prometheus esté completamente preparado para recibir consultas.

### 3.2. Función de Grafana

Grafana permite construir paneles y gráficos para visualizar las métricas consultadas desde una fuente de datos, como Prometheus.

Prometheus se encarga de recolectar y almacenar las series temporales, mientras que Grafana proporciona una interfaz visual para analizarlas e interpretarlas.

Para visualizar las métricas, Grafana debe tener configurado Prometheus como fuente de datos. La conexión de ambos servicios a una red común facilita la comunicación, pero la declaración de los contenedores en Docker Compose no configura automáticamente todos los paneles ni la fuente de datos.

### 3.3. Red compartida mynetwork

Al final del archivo `docker-compose.yml` se declara la red `mynetwork`. Los cinco servicios —`mysql`, `dbexporter`, `cadvisor`, `prometheus` y `grafana`— están conectados a ella.

Esta red permite que los contenedores se comuniquen mediante los nombres de los servicios, sin tener que establecer manualmente direcciones IP fijas para cada uno.

Por ejemplo, `dbexporter` puede utilizar `mysql` como destino para conectarse a la base de datos, mientras que Prometheus puede consultar a `cadvisor` y `dbexporter` mediante sus nombres y puertos correspondientes.

La red común facilita la comunicación interna y permite diferenciarla de los puertos publicados en el host. En esta configuración, Prometheus publica el puerto `9090` y Grafana publica el puerto `3000`.

### 3.4. Sintaxis YAML aplicada

Los elementos principales de esta parte son:

- **`networks`** declara las redes utilizadas por los servicios.
- **`mynetwork`** identifica la red compartida por los contenedores.
- **`depends_on`** establece el orden de inicio entre servicios.
- **`ports`** publica puertos para acceder a las interfaces desde el host.
- **`environment`** define parámetros del servicio, como la contraseña de administrador de Grafana.
- **`image`** permite seleccionar una imagen y una versión específica mediante su etiqueta.

La combinación de servicios, redes, dependencias, variables de entorno y puertos permite describir la infraestructura en un único archivo de configuración, facilitando su lectura y administración.

## Conclusión

El análisis del archivo `docker-compose.yml` y de los archivos auxiliares permite comprender cómo se integran los componentes de una infraestructura de monitoreo. MySQL administra los datos de la aplicación; `dbexporter` expone métricas de la base de datos; cAdvisor recopila información de los contenedores; Prometheus recolecta y almacena las métricas; y Grafana permite representarlas visualmente.

También se identificó la importancia de YAML para declarar servicios, variables de entorno, montajes, puertos, redes y dependencias. La red `mynetwork` facilita la comunicación entre los contenedores, mientras que `extra_hosts` permite que Prometheus acceda a la aplicación CRUD que se ejecuta en el host.

Por último, el análisis permite distinguir entre el orden de inicio y la disponibilidad real de un servicio, así como entre utilizar una base de datos y configurar explícitamente la persistencia de sus datos mediante volúmenes.

La elaboración del informe y el uso de commits descriptivos en Git permiten documentar los cambios realizados, explicar las decisiones técnicas y mantener un historial claro del proceso de desarrollo.
