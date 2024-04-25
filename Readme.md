# Lab 2. Dockefile y Docker Compose

En esta guía vamos a desarrollar una aplicación en Python que incluya un contador cuyo valor se guardará en un almacenamiento Redis.
Para ello, crearemos, en el directorio app, el fichero app.py con el código:

```python
from flask import Flask, render_template
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"
    name=os.getenv("NAME", "world")
    hostname=socket.gethostname()
    return render_template('index.html', name=name, hostname=hostname, visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080)
```

Como vemos, se importan los paquetes Flask y Redis, se crea una conexión con un almacenamiento Redis que estaría ubicado en un host llamado redis y una vez iniciado el programa, atenderá peticiones desde cualquier host, por el puerto 80 y ejecutará la función hello que obtendrá el objeto counter del almacenamiento Redis, lo incrementará y lo guardará en la variable visits. Si se produce algún error durante la conexión con Redis, se guardará en la variable visist un mensaje indicando el error.

Este código tiene como dependencias los paquetes Flask y Redis, que necesitamos instalar mediante el instalador de paquetes de Python (pip). Para ello, vamos a crear un fichero requirements.txt incluyendo estas dependencias.

```
# requirements.txt
Flask
Redis
```
En un entorno de Python se instalarían con:
```
$ pip install -r requirements.txt
```
Una vez definida la aplicación, vamos a crear una imagen a partir de un Dockerfile para nuestra aplicación y la ejecutaremos en un contenedor.

## 1. Fichero Dockerfile
(https://docs.docker.com/engine/reference/builder/)

```
# Use an official Python runtime as a parent image
FROM python:3.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY ./app /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 8080 available to the world outside this container
EXPOSE 8080

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

## 2. Construir la imagen 
```
$ docker image build -t python-webapp .
```

## 3. Ejecuta en un contenedor
```
$ docker container run -d -p 8080:8080 --name webapp python-webapp
```
Abre el navegador y comprueba el resultado.
 
Obviamente, aún no tenemos el contenedor con el almacenamiento Redis; por lo tanto, aparece el mensaje de error.

## 4. Subir la imagen a docker Hub

Autoriza a Docker a trabajar con tu Docker Hub, introduciendo las credenciales con:
```
$ docker login
```

Etiqueta la imagen con tu usuario, el nombre que desees darle a la imagen y un tag (será lastest si no incluyes ninguno).
```
$ docker image tag imagen username/imagen:tag
```
NOTA: Cambia username, imagen y tag con los valores adecuados.

Sube la imagen al DockerHub
```
$ docker image push username/imagen:tag
```
NOTA: Cambia username, imagen y tag con los valores adecuados.

## 5. Elimina el contenedor y la imagen del host local
```
$ docker container stop webapp
$ docker container rm webapp
$ docker image rm python-webapp
```

## 6. Iniciar un contenedor con el almacenamiento Redis 
Si lo necesitas, Busca en el Docker Hub para más información sobre la imagen Redis.
```
$ docker container run --name redis -d redis redis-server --save 20 1 --loglevel warning
```

## 7. Iniciar contenedor de la aplicación 

Usando la imagen del Docker Hub conectándola con el almacenamiento Redis

```
$ docker run --name front-end -d -p 8080:8080 --link redis username/imagen:tag
```
NOTA: Recuerda que username, imagen y tag deben ser los valores de la imagen en el DockerHub.

Abre el navegador y comprueba el resultado.
 
Ya disponemos del contenedor con el almacenamiento Redis; por lo tanto, 
aparecerá el número de visitas, que irá aumentando con cada visita. 
Sin embargo, si el contendor redis se destruye no persistirán los datos y 
si un nuevo contenedor redis es creado comenzará la cuenta de visitas de nuevo desde el principio.

```
$ docker container stop redis
$ docker container rm redis
$ docker container run --name redis -d redis redis-server --save 20 1 --loglevel warning
```
Ver de nuevo en el navegador.

Para resolver este problema, vamos a añadir un volumen persistente al contenedor redis.

### Creamos el volumen

```
$ docker volume create redis-vol
```
Podemos inspeccionarlo

```
$ docker volume inspect redis-vol
```
También, podemos eliminarlo con:

```
$ docker volume rm redis-vol
```
Si volvemos a ejecutar el contenedor Redis pero ahora con el volumen persistente

```
$ docker run --name redis -d -v redis-vol:/data redis redis-server --save 20 1 --loglevel warning
``` 
Si ahora paramos y eliminamos el contenedor, al iniciar uno nuevo con el mismo volumen 
el contador habrá persistido, continuando por el siguiente valor.


# Docker compose
## 1. Docker Compose

Docker Compose, ahora, es un módulo que forma parte de Docker y puede ser instalado como parte de Docker CLI.
Comprueba si lo tienes instalado con:

```
$ docker compose version
Docker Compose version v2.26.1
```

## 2. Crear el fichero docker-compose.yml con los servicios, volúmenes, etc.
```
version: '3.9'
services:
  redis:
    image: redis
    container_name: redis
    volumes:
      - redis-vol:/data
  web-app:
    image: python-webapp
    container_name: python-app
    ports:
      - "8080:8080"
    depends_on:
      - redis
volumes:
  redis-vol:
    name: redis-vol
```

## 3. Iniciar los servicios

```
$ docker-compose up -d
```

## 4. Detener los servicios

```
$ docker-compose down
```

El volumen persistirá, si deseamos eliminarlo 
podemos incluir la opción -v en el comando anterior.