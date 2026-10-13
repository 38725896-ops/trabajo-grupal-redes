# trabajo-grupal-redes

# Informe Grupal - Etapa 6: Monitoreo e Infraestructura

## Introducción General
<!-- A cargo de: [Cecilia Fabio] -->

En entornos donde una aplicación corre junto a una base de datos y otros servicios, no alcanza con que todo esté funcionando: hace falta poder observar cómo se comporta. ¿Cuántas peticiones recibe la aplicación? ¿Cuánta CPU consume cada contenedor? ¿Está respondiendo la base de datos? El monitoreo responde estas preguntas con datos concretos y permite detectar problemas antes de que los usuarios los noticen.

Este trabajo documenta la Etapa 6 del proyecto, en la que se agrega una capa de monitoreo a un CRUD desarrollado en Python. La aplicación corre en el host en el puerto 8888 y usa MySQL como base de datos. Sobre ella se levantó una plataforma de observabilidad basada en Prometheus, que recolecta métricas, y Grafana, que las visualiza en dashboards.

Las métricas son series numéricas que se registran a lo largo del tiempo, como requests por segundo o uso de memoria. Se diferencian de los logs, que registran eventos puntuales. Prometheus las obtiene mediante un modelo de scrape: cada pocos segundos consulta a sus targets, que son los puntos que exponen métricas. En esta práctica los targets son:

prometheus: sus propias métricas internas.
cadvisor: el consumo de recursos de cada contenedor.
dbexporter: las estadísticas de MySQL, traducidas al formato de Prometheus.
crud-app: la aplicación Python, ubicada en el host.

Toda la infraestructura se define en un único archivo docker-compose.yml con cinco servicios conectados por la red mynetwork: mysql, dbexporter, cadvisor, prometheus y grafana. El informe se organiza en tres partes que siguen la estructura de ese archivo:

Persistencia de datos: los servicios mysql y dbexporter, con sus variables de entorno, la configuración montada en solo lectura y las dependencias entre ambos.
Recolección de métricas: los servicios cadvisor y prometheus, las rutas del host que se montan, los puertos publicados y el acceso a la aplicación que corre fuera de Docker.
Visualización y topología: el servicio grafana, la red compartida y la resolución de nombres entre los cinco servicios.

## Parte 1 — Persistencia de Datos
<!-- A cargo de: [Evelyn Pizzichini] -->

mysql: es la base de datos del CRUD. Usa la imagen mysql:8.0 y publica el 3306.
Variables de entorno: MYSQL_ROOT_PASSWORD define la contraseña de root y MYSQL_DATABASE crea la base crud al iniciar. Van como variables de entorno y no dentro de la imagen, para poder cambiarlas sin reconstruirla.
dbexporter: traduce las estadísticas internas de MySQL al formato de Prometheus y las expone en el puerto 9104. No lo publiqué al host porque solo lo necesita Prometheus, dentro de la red.
my.cnf en ro: contiene las credenciales con las que el exporter se conecta a MySQL. Se monta como solo lectura (:ro) para que el contenedor no pueda modificarlo, siguiendo el principio de mínimo privilegio.
depends_on: dbexporter depende de mysql, así que Compose arranca primero la base. Aclaralo en el informe: depends_on controla el orden de arranque, no que MySQL ya esté listo para aceptar conexiones. Por eso el target puede aparecer en DOWN los primeros segundos.


## Parte 2 — Recolección de Métricas
<!-- A cargo de: [Lautaro Ferreyra] -->

cadvisor: mide el consumo de CPU, memoria y red de cada contenedor. Para eso necesita leer información del host, y por eso monta /, /var/run, /sys y /var/lib/docker, todos en solo lectura porque solo observa y no modifica nada.
prometheus: hace el scrape cada 5 s según prometheus.yml y guarda las series de tiempo.
Puertos publicados:
9090: acceder a la UI de Prometheus y a /targets para verificar el estado.
8080: inspeccionar cAdvisor directamente.
3000: acceder a Grafana desde el navegador.
3306: acceso directo a MySQL desde el host.
dbexporter: sin publicar, porque solo lo consulta Prometheus por la red interna.
extra_hosts: host.docker.internal:host-gateway hace que ese nombre resuelva a la IP del host. Es necesario porque la app Python corre en el host y no en un contenedor, y en Linux ese nombre no existe por defecto.

## Parte 3 — Visualización y Topología
<!-- A cargo de: [Priscila Gigena] -->

grafana: muestra los datos de Prometheus en dashboards y publica el puerto 3000.
depends_on hacia Prometheus: Grafana necesita a Prometheus como fuente de datos, así que arranca después. Vale la misma aclaración sobre el orden de arranque.
Red mynetwork: los 5 servicios comparten una red bridge definida por el usuario, que incluye un DNS interno. Por eso se alcanzan por nombre de servicio: Prometheus usa cadvisor:8080 y dbexporter:9104, y Grafana usa http://prometheus:9090. Sin red compartida habría que usar IPs, que cambian en cada arranque.


## Conclusión
<!-- A cargo de: [Candela Tome] -->

La práctica permitió armar, con un único archivo docker-compose.yml, una plataforma de monitoreo completa formada por cinco servicios: mysql y dbexporter (persistencia), cadvisor y prometheus (recolección) y grafana (visualización). Cada servicio tiene una responsabilidad concreta y puede reemplazarse o ampliarse sin tocar al resto.

Orden de arranque y dependencias. depends_on por sí solo solo garantiza que un contenedor arranque antes que otro, no que esté listo para atender. Al agregar un healthcheck a MySQL y a Prometheus y usar condition: service_healthy, dbexporter espera a que la base acepte conexiones y Grafana espera a que Prometheus responda. Así se evitan los targets en DOWN durante los primeros segundos.

Seguridad y acceso. Los archivos de configuración (my.cnf, prometheus.yml) y las rutas del host que lee cAdvisor se montan en modo solo lectura (ro), porque los contenedores solo necesitan consultarlos. Solo se publicaron los puertos que el usuario necesita alcanzar desde el navegador o el host (9090, 8080, 3000 y 3306). El dbexporter quedó sin publicar, ya que únicamente Prometheus lo consulta, por la red interna.

Red y resolución de nombres. La red mynetwork conecta los cinco servicios y provee un DNS interno, por lo que se comunican por nombre de servicio (cadvisor:8080, dbexporter:9104, http://prometheus:9090) sin depender de IPs que cambian. La aplicación Python corre en el host, fuera de esa red, y por eso Prometheus la alcanza mediante extra_hosts con host.docker.internal:host-gateway.

Modelo de monitoreo. Prometheus trabaja con scrape: consulta periódicamente a cada target y guarda series numéricas. Esto lo diferencia de los logs, que registran eventos puntuales. Las métricas muestran tendencias, como las requests por segundo subiendo bajo carga, y los logs aportan el detalle de lo que ocurrió en un evento. Ambos se complementan.

Limitaciones y mejoras posibles. Para simplificar la práctica se usaron credenciales de root en texto plano en el compose y en my.cnf. En un entorno real convendría un usuario dedicado con permisos mínimos y el uso de secretos. También faltaría persistir los datos de Prometheus y Grafana con volúmenes, y agregar healthchecks a los servicios restantes.

Si tu grupo tiene resultados propios, por ejemplo cuántas requests por segundo alcanzó el panel durante la carga, conviene sumar una oración con ese dato para que la conclusión no quede genérica. Si lo preferís, te la acorto a un solo párrafo.

