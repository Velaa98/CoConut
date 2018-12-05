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

## Creación y activación del entorno virtual con python2

Debemos tener en cuenta donde lo vamos a crear ya que en la configuración del virtual host de apache debemos indicar la ruta del entorno virtual.

Mediante el parámetro python podemos indicarle la versión de python a usar.

```
virtualenv --python=python2.7 coconut
source coconut/bin/activate
```

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

Dentro del registro de apache, cuando reiniciamos el servicio debemos comprobar que apache está usando la versión de python correcta
```
[Wed Dec 05 09:08:28.919019 2018] [mpm_event:notice] [pid 13758:tid 140057472476352] AH00491: caught SIGTERM, shutting down
[Wed Dec 05 09:08:28.976625 2018] [mpm_event:notice] [pid 21479:tid 140314524233792] AH00489: Apache/2.4.25 (Debian) mod_wsgi/4.5.11 Python/2.7 configured -- resuming normal operations
[Wed Dec 05 09:08:28.976829 2018] [core:notice] [pid 21479:tid 140314524233792] AH00094: Command line: '/usr/sbin/apache2'
```

```tail -f /var/log/apache2/error.log```

## Configuración de la base de datos Postgres

### Creación de las tablas necesarias

Mediante este [script](https://github.com/Velaa98/CoConut/blob/master/DATAdesign/build.sql) crearemos las tablas:

- Roles. Contiene los posibles roles existentes, en nuestro el rol 1 corresponde al profesor que tendrá acceso a todas las copias de los alumnos y el rol 2 que será el alumno y podrá almacenar las copias tanto automáticamente como manualmente.
- Users. Donde estarán almacenados los datos de los usuarios que usen la aplicación.
- Hosts. Almacena la dirección ip de cada máquina y su propietario.
- Backups. Guarda los datos relacionados con las copias, el usuario que la ha hecho, a que máquina pertenece, la fecha, una descripción y si es manual (desde la aplicación) o autómatica(si se ha realizado mediante un script que ha ejecutado el usuario).

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

## Modificaciones varias para adaptar la aplicación

En principio para el correcto funcionamiento solo debemos cambiar los parámetros de conexión a la base de datos(IP, usuario administrador y la contraseña de este). Además de esto debemos modificar los nombres de las plantillas para que se adapten al curso actual.

### Modificación de los parámetros de conexión a la base de datos.

Si usamos el editor nano mediante ```Alt + R``` podemos buscar y reemplazar.

En el fichero [functions.py](https://github.com/Velaa98/CoConut/blob/master/app/functions.py) debemos sustituir:

- La ip 172.22.200.110 por la de nuestro servidor de base de datos.
- El usuario en la variable ```v_useradmin = 'admin'``` por el usuario administrador de nuestra base de datos
- La contraseña en la variable  ```v_passwordadmin = 'admin1234'``` por la del usuario administrador de nuestra base de datos.

### Modificación de los nombres de los hosts.

