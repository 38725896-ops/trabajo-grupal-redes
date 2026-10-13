# trabajo-grupal-redes

# Informe Grupal - Etapa 6: Monitoreo e Infraestructura

## Introducción General
<!-- A cargo de: [Cecilia Fabio] -->

## Parte 1 — Persistencia de Datos
<!-- A cargo de: [Evelyn Pizzichini] -->
- Servicios: `mysql` y `dbexporter`
- Variables de entorno para contraseña de MySQL
- Montaje de `my.cnf` en modo solo lectura (`ro`)
- Relación de dependencia (`depends_on`)

## Parte 2 — Recolección de Métricas
<!-- A cargo de: [Lautaro Ferreyra] -->
- Servicios: `cadvisor` y `prometheus`
- Montaje de rutas del host en `cadvisor` (solo lectura)
- Puertos publicados y justificación
- Configuración de `extra_hosts` con `host.docker.internal`

## Parte 3 — Visualización y Topología
<!-- A cargo de: [Priscila Gigena] -->
- Servicio: `grafana` y red `mynetwork`
- Dependencia de Grafana hacia Prometheus
- Resolución de nombres por red común entre los 5 servicios

## Conclusión
<!-- A cargo de: [Candela Tome] -->
