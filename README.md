# CoConut

Aplicación para llevar un seguimiento de las copias de seguridad realizadas por los alumnos de 2º ASIR del I.E.S Gonzalo Nazareno, escrita en python y que hace uso de una base de datos postGreSQL.

## Vista previa

![](http://www.charliejsanchez.com/wp-content/uploads/2017/12/example.jpg)

Created by @sfbenitez & @carlosjsanch3z

## Paquetes necesarios

- apache2
- postgresql-9.6
- libapache2-mod-wsgi
- python-virtualenv
- virtualenv

## Creación del entorno virtual con python2

```virtualenv coconut```

### Instalación de los requisitos

Con el entorno virtual activado, actualizar pip e instalar requsitos.

```pip install -U pip```

```pip install -r requirements.txt```

## Despliegue en un servidor Apache2 con módulo WSGI

### Creación del servidor virtual

``` [apache]
<VirtualHost *>
    ServerName coconut.ferrete.gonzalonazareno.org
    DocumentRoot /var/www/coconut
    WSGIDaemonProcess coconut user=www-data group=www-data python-home=/home/debian/venv python-path=/var/www/coconut
    WSGIProcessGroup coconut
    WSGIScriptAlias / /var/www/coconut/coconut.wsgi
    <Directory /var/www/coconut/>
	Require all granted
	<Files coconut.wsgi>
		Require all granted
	</Files>
   </Directory>
</VirtualHost>
```

### Creación del fichero wsgi
El fichero wsgi deberá estar en el mismo directorio que la aplicación.

``` [python]
import sys, os, bottle
import beaker.middleware

import coconut # Import coconut.py

sys.path = ['/var/www/coconut/'] + sys.path
os.chdir(os.path.dirname(__file__))

# Inicialice app with SessionMiddleware environ
application = beaker.middleware.SessionMiddleware(bottle.default_app(), coconut.session_opts)
```
### Si se usa apache para mostrar la página comentar las siguientes líneas dentro de coconut.py
``` [python]
#app = SessionMiddleware(app(), session_opts)
#debug(True) # Desactivar Debug en entorno de producción
#run(app=app, host = '0.0.0.0', port = 8080)
```

### En caso de errores comprobar el registro de apache

```tail -f /var/log/apache2/error.log```


## Configuración de la base de datos Postgres

### Permitir el inicio de manera local para los nuevos usuarios, sustituyendo "peer" por "md5"

postgres@server:~$ nano /etc/postgresql/9.6/main/pg_hba.conf

~~~
local   all             all                                              md5
host    all             all              0.0.0.0/0                       md5
~~~

### Habilitar conexiones remotas

postgres@server:~$ nano /etc/postgresql/9.6/main/postgresql.conf

~~~
listen_addresses = '*'
~~~

root@server:/home/vagrant# systemctl restart postgresql

### Iniciar sesion con el superuser

~~~
root@server:/home/vagrant# su - postgres
postgres@server:~$ psql
~~~

### Crear un usuario especifico para la nueva base de datos

~~~
CREATE ROLE admin PASSWORD 'admin' CREATEDB CREATEROLE INHERIT LOGIN;
~~~

### Crear una nueva base de datos

~~~
CREATE DATABASE  db_backup;
ALTER DATABASE db_backup OWNER TO admin;
~~~

### Iniciar sesion con el usuario admin y crear las tablas

~~~
postgres@server:~$ psql -U admin db_backup
~~~

### Crear un nuevo rol y darle los privilegios sobre las tablas

~~~
CREATE ROLE pupilgroup;
GRANT SELECT,UPDATE,INSERT ON ALL TABLES IN SCHEMA public TO pupilgroup;
~~~


### Para revocar los permisos

~~~
REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM pupilgroup;
~~~

### Creación manual de un usuario con rol profesor

~~~
CREATE USER "profesor" PASSWORD 'profesor' IN ROLE pupilgroup;
INSERT INTO USERS values ('profesor','Nombre Profesor','profesor@gmail.com',to_date('01/01/1971','DD/MM/YYYY'),1);
~~~
