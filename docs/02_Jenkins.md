# Jenkins

## Instalar Jenkins

Seguiremos las instrucción detalladas en la página ofical de [Jenkins](https://jenkins.io/doc/book/installing/)

Creamos una red para que los dos contenederos que usamos se puedan comunicar
```
docker network create jenkins
```

Crear volúmenes para compartir datos y guardarlos en el "host"
```
docker volume create jenkins-docker-certs
docker volume create jenkins-data
```

Para poder ejecutar comandos dentro de Jenkins instaladmos docker:dind
```
docker container run \
  --name jenkins-docker \
  --rm \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind
```

Ahora lanzamos jenkins/blueocean
```
docker container run \
  --name jenkins-blueocean \
  --rm \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  jenkinsci/blueocean
```

> BONUS crear un archivo de docker compose para automatizar este proceso

Todos estos pasos se pueden automatizar mediante docker-compose. Ver [jenkins.yml](../compose/jenkins.yml)

Navegar al directorio "compose" y ejecutar
```
docker-compose -f jenkins.yml up -d
```

Si no hemos destruido o parado los contenedores anteriores, el sistema nos alertará de un conflicto con los puertos
```
WARNING: Host is already in use by another container

ERROR: for compose_jenkins-docker_1  Cannot start service jenkins-docker: driver failed programming external connectivity on endpoint compose_jenkins-docker_1 (f1f6a58539792074c55ec31787226ab836406cd8c2dca0144dd8799303d644e3): Bind for 0.0.0.0:2376 failed: port is already allocated
Creating compose_jenkins-blueocean_1 ... error

ERROR: for compose_jenkins-blueocean_1  Cannot start service jenkins-blueocean: driver failed programming external connectivity on endpoint compose_jenkins-blueocean_1 (12886eed248bb4cb9305dd46911aa3ca3178f7798c4732423cfa90c8e652bd61): Bind for 0.0.0.0:50000 failed: port is already allocated
```

Deberemos parar los contenedores existente volver a ejecutar
```
docker-compose -f jenkins.yml up -d
``` 

Podremos ver los volúmenes creados usando
```
docker volume ls
```

Veremos que tenemos dos versiones de los volúmenes.
Ahora borraremos las que no vayamos a usar con el comando
```
docker volume rm jenkins-data
docker volume rm jenkins-docker-certs
```

## Configuración inicial

Para listar los contenedores activos ejecutamos
```
docker ps
```

Deberíamos ver algo así
```
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                              NAMES
60f9136cfb3e        jenkinsci/blueocean   "/sbin/tini -- /usr/…"   9 minutes ago       Up 8 seconds        0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   compose_jenkins-blueocean_1
a134bd24d67c        docker:dind           "dockerd-entrypoint.…"   9 minutes ago       Up 8 seconds        2375/tcp, 0.0.0.0:2376->2376/tcp                   compose_jenkins-docker_1
```

Para instalar jenkins necesitamos una clave que se encuentra dentro del contenedor así que nos conectamos al contenedor de blueocean
```
docker exec -it 60f9136cfb3e bash
```

Una vez dentro del contenedor obtenemos la clave que necesitamos para completar la instalación
```
cat /var/jenkins_home/secrets/initialAdminPassword
```

Instalamos los plugins más comunes.

Para el usuario admin usaremos:

```
user = admin
password = admin123
```

Aceptar todo hasta llegar a la pantalla de Jenkins.

Ahora pararemos los contenedores y los volveremos a iniciar para asegurarnos de que nuestros datos persisten:
```
docker-compose -f jenkins.yml down
```

y después
```
docker-compose -f jenkins.yml up -d
```

Deberíamos poder hacer login con admin:admin123

> ¡ENHORABUENA! Hemos instalado Jenkins con éxito.

## Plugins

Nos vamos a Jenkins > Administrar jenkins e instalamos los plugins de

1. GitHub
2. Kubernetes
3. Docker
4. Kubernetes CLI plugin

y reiniciamos el servidor

Ahora generamos un Token de GitHub para darle accesso a Jenkins. Visitaremos [github/settings/tokens](https://github.com/settings/tokens)

y marcamos:

- repo (todo)
- admin:repo_hook (todo)

También podemos hacerlo directamente en Jenkins a través de 

Administrar Jenkins > Configurar > Scroll down hasta 'GitHub Servers' > Add GitHub Server > Avanzado > 

Manage Additional actions > Convert login and password to token

# Add Git credentials

Nos vamos a Credentials > Sytem > Global credentials > Add credentials

1. Seleccionamos username with password
2. Introducimos datos y como ID usamos Git (o GitHub o lo que nos parezca :) )
3. Damos a Ok

# Add Docker credentials

Nos vamos a Credentials > Sytem > Global credentials > Add credentials

1. Seleccionamos username with password
2. Introducimos datos y como ID usamos Docker
3. Damos a Ok