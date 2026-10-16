# Configuración de Traefik v3.6 (Desarrollo local y producción)

Este repositorio contiene la configuración base para desplegar **Traefik v3.6** como proxy inverso mediante Docker. Está diseñado para funcionar tanto en un entorno de desarrollo local como en un servidor VPS de producción, permitiendo gestionar el acceso a diferentes aplicaciones y habilitar conexiones HTTPS mediante certificados de Let's Encrypt.

## 1. Requisitos previos

Antes de iniciar, asegúrate de tener instalados los siguientes componentes:

- **Docker:** para ejecutar los contenedores.
- **Docker Compose:** para definir y administrar los servicios.
- **Traefik v3.6:** configurado mediante el archivo `docker-compose.yml`.

Además, es necesario crear una red externa de Docker que permita la comunicación entre Traefik y las aplicaciones que se publicarán a través del proxy.

Ejecuta el siguiente comando:

```
docker network create web
```

Esta red se utilizará para conectar Traefik con los contenedores de las aplicaciones.

## 2. Preparación del archivo de certificados SSL/TLS

Traefik utiliza Let's Encrypt para obtener y renovar automáticamente los certificados SSL/TLS. Para almacenar la información de estos certificados, se utiliza el archivo `acme.json`.

### Crear el archivo

Desde la raíz del proyecto, ejecuta:

```
touch acme.json
```

### Configurar los permisos

Por motivos de seguridad, Traefik requiere que este archivo tenga permisos restrictivos. Configúralos con el siguiente comando:

```
chmod 600 acme.json
```

Estos permisos permiten que únicamente el propietario del archivo pueda leerlo y modificarlo.

**Importante:** el archivo `acme.json` contiene información sensible relacionada con los certificados y las claves privadas. Nunca lo subas a un repositorio público ni lo compartas con terceros.

Para evitar que Git lo incluya en los commits, agrega lo siguiente a tu archivo `.gitignore`:

```
acme.json
```

Si el archivo ya estaba siendo rastreado por Git, tendrás que dejar de rastrearlo antes de que esta regla tenga efecto:

```
git rm --cached acme.json
```

Este comando elimina el archivo del seguimiento de Git, pero conserva el archivo local.

## 3. Configuración de los entornos

El proyecto está preparado para utilizarse en dos escenarios: desarrollo local y producción.

### 3.1. Entorno local

En el entorno local, Traefik permite probar la configuración y verificar el funcionamiento del proxy sin necesidad de publicar las aplicaciones en Internet.

Los accesos configurados son:

- **Dashboard de Traefik:** `http://localhost:8081`
- **Puerto web:** `http://localhost:8002`

Estas direcciones corresponden a los puertos publicados en el archivo `docker-compose.yml`. Si modificas la configuración, las direcciones también pueden cambiar.

El dashboard permite visualizar información sobre los routers, servicios y otros componentes que Traefik administra.

> **Recomendación:** utiliza el dashboard únicamente en entornos de confianza. No lo expongas públicamente sin configurar los mecanismos de autenticación y protección adecuados.

### 3.2. Entorno de producción (VPS)

Para desplegar Traefik en un servidor VPS y habilitar HTTPS, es necesario adaptar el archivo `docker-compose.yml` a las condiciones del entorno de producción.

#### Paso 1. Configurar la redirección HTTP a HTTPS

En la configuración del punto de entrada `web`, habilita la redirección hacia `websecure`.

Esto permite que las solicitudes recibidas mediante HTTP se redirijan a HTTPS, siempre que los puntos de entrada y los puertos estén configurados correctamente.

#### Paso 2. Configurar Let's Encrypt

Habilita la sección `certificatesresolvers` y configura el proveedor ACME para obtener certificados automáticamente.

Sustituye el correo de ejemplo por una dirección válida:

```
certificatesResolvers:
  myresolver:
    acme:
      email: tu-email@correo.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

Esta es una configuración de ejemplo para el desafío HTTP-01. Debe adaptarse a la estructura y la sintaxis del archivo `docker-compose.yml` del proyecto.

Para que el desafío funcione, el dominio debe apuntar al servidor y el puerto 80 debe ser accesible desde Internet.

#### Paso 3. Verificar los puertos

Comprueba que los puertos 80 y 443 estén disponibles en el VPS y que el firewall permita el tráfico correspondiente.

La configuración habitual es:

- **Puerto 80:** recepción de solicitudes HTTP y validación HTTP-01 de Let's Encrypt.
- **Puerto 443:** recepción de conexiones HTTPS.

Si otro servicio ya utiliza estos puertos, será necesario resolver el conflicto antes de iniciar Traefik.

#### Paso 4. Configurar los DNS

Antes de iniciar el entorno de producción, configura los registros DNS de tu dominio.

Por lo general, debes crear un registro de tipo `A` que apunte el dominio a la dirección IPv4 pública del VPS. Si utilizas IPv6, también puedes necesitar un registro `AAAA` correctamente configurado.

Verifica que la resolución DNS sea correcta antes de solicitar los certificados.

## 4. Publicación de nuevas aplicaciones

Una de las principales ventajas de Traefik es que puede detectar automáticamente los contenedores Docker y dirigir el tráfico hacia ellos mediante routers y servicios.

Para publicar una nueva aplicación, esta debe cumplir las siguientes condiciones:

1. Estar conectada a la red Docker `web`.
2. Tener habilitada la integración con Traefik mediante etiquetas.
3. Definir un router con la regla de dominio correspondiente.
4. Indicar el punto de entrada y, cuando corresponda, el mecanismo de TLS.
5. Especificar el puerto interno de la aplicación si Traefik no puede detectarlo correctamente.

### Ejemplo de configuración

El siguiente ejemplo muestra cómo conectar una aplicación a Traefik desde su propio archivo `docker-compose.yml`:

```
services:
  mi-app:
    image: nginx:alpine
    container_name: mi-app
    restart: unless-stopped

    networks:
      - web

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mi-app.rule=Host(`tudominio.com`)"
      - "traefik.http.routers.mi-app.entrypoints=websecure"
      - "traefik.http.routers.mi-app.tls=true"
      - "traefik.http.routers.mi-app.tls.certresolver=myresolver"
      - "traefik.http.services.mi-app.loadbalancer.server.port=80"

networks:
  web:
    external: true
```

**Consideraciones importantes:**

- Sustituye `tudominio.com` por el dominio real de tu aplicación.
- El nombre `myresolver` debe coincidir con el resolver definido en la configuración de Traefik.
- El puerto `80` debe corresponder al puerto interno en el que escucha la aplicación. No necesariamente coincide con el puerto publicado en el host.
- La red `web` debe existir previamente y ser compartida por Traefik y la aplicación.
- Para utilizar `websecure`, Traefik debe tener configurado ese punto de entrada y debe estar publicado el puerto correspondiente.

Con esta configuración, las solicitudes HTTPS dirigidas a `tudominio.com` pueden ser encaminadas al contenedor `mi-app`, siempre que el DNS, los puertos y la configuración de Traefik sean correctos.

## 5. Arquitectura del sistema

El flujo de comunicación en producción funciona de la siguiente manera:

```
                 INTERNET
                     |
                     v
              Dominio y DNS
                     |
                     v
                Servidor VPS
                     |
                     v
               Puertos 80/443
                     |
                     v
              +--------------+
              |    Traefik   |
              | Proxy inverso|
              +--------------+
                     |
              Red Docker "web"
                /    |    \
               v     v     v
            App 1  App 2  App 3
```

Traefik recibe las solicitudes, identifica el dominio solicitado y las dirige al contenedor correspondiente. Cuando está configurado correctamente, también gestiona los certificados TLS para habilitar conexiones HTTPS.

Esta arquitectura permite publicar múltiples aplicaciones utilizando un único servidor y una única instancia de Traefik, sin necesidad de asignar una dirección IP pública independiente a cada contenedor.

## 6. Supervisión y depuración

Si Traefik no inicia correctamente, no genera certificados o no detecta las aplicaciones, puedes revisar los registros del contenedor.

### Consultar los logs en tiempo real

```
docker logs -f traefik
```

Este comando muestra los mensajes que genera el contenedor llamado `traefik`. Si el contenedor tiene otro nombre, reemplázalo por el nombre correspondiente.

### Verificar el estado de los contenedores

```
docker compose ps
```

Permite comprobar si los servicios están activos o si alguno presenta errores.

### Revisar la configuración de Compose

```
docker compose config
```

Este comando valida y muestra la configuración resultante de Docker Compose, lo que ayuda a identificar errores de sintaxis y problemas de interpolación de variables.

### Comprobar la red de Docker

```
docker network inspect web
```

Permite verificar qué contenedores están conectados a la red externa `web`.

## 7. Consideraciones de seguridad

Para mantener una configuración segura en producción, ten en cuenta las siguientes recomendaciones:

- **Proteger `acme.json`:** no lo subas a GitHub ni lo incluyas en imágenes Docker.
- **Restringir el dashboard:** evita exponerlo públicamente sin autenticación y controles de acceso.
- **Utilizar HTTPS:** configura TLS y certificados válidos para las aplicaciones que se publiquen en Internet.
- **Proteger los puertos:** permite únicamente el tráfico necesario mediante el firewall del VPS.
- **Evitar privilegios innecesarios:** no ejecutes contenedores con permisos elevados sin una razón justificada.
- **Gestionar las credenciales:** utiliza variables de entorno o mecanismos de gestión de secretos cuando sea apropiado, evitando incluir contraseñas directamente en los archivos versionados.
- **Mantener el sistema actualizado:** aplica actualizaciones de seguridad a Docker, Traefik y las aplicaciones desplegadas.

## 8. Puesta en marcha

Desde el directorio que contiene el archivo `docker-compose.yml`, puedes iniciar los servicios con:

```
docker compose up -d
```

Para comprobar el estado:

```
docker compose ps
```

Y para consultar los registros:

```
docker logs -f traefik
```

Antes de utilizar esta configuración en producción, verifica que los puertos, los puntos de entrada, los certificados, los nombres de red y las reglas de enrutamiento coincidan con tu configuración real.

---

## Conclusión

Este repositorio proporciona una base para desplegar Traefik v3.6 como proxy inverso mediante Docker, tanto en desarrollo local como en producción.

Su configuración permite centralizar el acceso a diferentes aplicaciones, administrar el enrutamiento por dominio y habilitar HTTPS mediante Let's Encrypt. Con una correcta configuración de DNS, redes, puertos y medidas de seguridad, es posible mantener una infraestructura organizada, extensible y fácil de administrar.

Alinear la redacción con tu configuración real