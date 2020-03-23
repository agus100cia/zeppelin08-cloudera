# Livy y Zeppelin para Cloudera Manager and CDH.


Recuerda que Zeppelin 0.7 no reconoce a Spark 2.3, para la ultima versión de Spark necesitas Zeppelin 0.8

Este repositorio de git es usando para construir el CSD y parcel de zeppelin para Cloudera Manager

Pueden correr multiples instancias de Livy (una para Spark y otra para Spark2), y esto podrá correr en una instancia de Zeppelin que utilice a Livy como interprete.

Tanto Livy como Zeppelin funcionarán como servicios en Cloudera manager, y sus configuraciones estarán disponibles a través de Cloudera Manager UI

Esto puede trabajar bajo un ambiente kerberizado.

This has been tested on CDH 5.12.1.

## Construcción e instalación

En este repositorio estan pre-construidos los CSD y parcel para Livy y Zeppelin.
Para contruir el CSD y Parcel debes hacer un clone de este proyecto

````sh
git clone URL
`````
Y luego ejecutar:

```
#Build the Parcel files, this make take some time
sh build.sh parcel

#Build the CSDs
sh build.sh csd
```

## Instalación en Cloudera.

Lo primero que debes hacer es copiar el CSD en la carpeta /opt/cloudera/csd

`````sh
sudo cp $GIT_PROJECT/ZEPPELIN-0.8.0.jar /opt/cloudera/csd
sudo cp $GIT_PROJECT/LIVY-0.5.0.jar /opt/cloudera/csd
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/csd/ZEPPELIN-0.8.0.jar
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/csd/LIVY-0.5.0.jar
sudo chmod 644 /opt/cloudera/csd/ZEPPELIN-0.8.0.jar
sudo chmod 644 /opt/cloudera/csd/LIVY-0.5.0.jar
sudo service cloudera-scm-server restart

```````   

Debemos tener instalado un servidor apache en un servidor, en mi caso lo instalare en un servidor llamado nodo1 sobre centos 7.

````sh
sudo yum install httpd
sudo systemctl start httpd

```````

Ahora debemos crear una carpeta en la ruta pública del servidor apache

````sh
sudo mkdir /var/www/html/zeppelin08
sudo mkdir /var/www/html/livy05

``````

Y copiar el parcel y manifest a esa carpeta

`````ssh
sudo cp $GIT_PROJECT/ZEPPELIN-0.8.0_build/*  /var/www/html/zeppelin08
sudo cp $GIT_PROJECT/LIVY-0.5.0_build/*  /var/www/html/livy05

```````   

Deberíamos poder acceder a los archivos mediante la ruta:

http://nodo1/zeppelin08/

Ahora en Cloudera Manager vamos a parcels ==> Configuration ==> Remote Parcel Repository URLs 

Y agregamos la nueva URL http://nodo1/zeppelin08/ y http://nodo1/livy05/

Al dar click en "Check for new parcels", debería listarse nuestro Zeppelin y Livy.

Finalmente queda presionar "Download"=>Distribute=>Unpacked=>Activate


## Agregar el servicio de Livy.

Livy es un servicio de Apache que publica el SparkSession en forma de un servicio web REST, esto es imprescindible para que trabaje con Zeppelin.

Para agregar el servicio:

Add service ==> Livy ==> Seleccionar el Host ==> Gateway en el mismo host ==> Next ==> Finish

#### Error1: Aquí seguramente se presentará un error:

`````sh
Error found before invoking supervisord: 'getpwnam(): Name not found livy'

````````    

#### Solución Error 1:

`````sh
groupadd livy
useradd livy -g livy

``````

## Agregar el servicio de Zeppelin.

Para poder agregar el servicio de Zeppelin, es necesario que el parcel se haya instalado.

Add service ==> ZEPPELIN ==> Seleccionar el Host ==> Next ==> Finish


### Error2: 

`````sh

java.lang.IllegalArgumentException: The variable [${zeppelin_java_options}] does not have a correspondin

``````  

### Solución Error 2:

Ir a Cloudera Manager ==> Zeppelin ==> Configuration ==> Buscar zeppelin_java_options

Colocar el valor : -Xms1024m


### Error 3:

`````sh
mkdir: `file:///var/local/zeppelin/conf': Input/output error

`````` 

### Solución error 3:

``````sh
sudo mkdir -p /var/local/zeppelin
chown zeppelin:zeppelin /var/local/zeppelin

`````` 


Information about installing custom services can be found at [https://www.cloudera.com/documentation/enterprise/latest/topics/cm_mc_addon_services.html](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_mc_addon_services.html).

## Configuration

Para iniciar


### Kerberos

Livy and Zeppelin can be configured to work in a secure environment using Kerberos:

![Kerberos Architecture](kerberos.png)

Livy will talk to a secure cluster and proxy to the user provided by Zeppelin. Zeppelin will talk to Livy using SPNEGO and provide the logged-in user as a proxy user to Livy. As a result, the logged-in user will be the owner of the Spark application in YARN.

#### Livy Configuration
To set this up, enable both User Authentication (spnego) and Access Control in the Livy setting.

You will also need to set up `hadoop.proxyuser.livy.groups` and `hadoop.proxyuser.livy.hosts` in the HDFS core-site to allow Livy to proxy to users. You should limit this only to the groups that will be using Zeppelin and the host Livy is running on.

If you are using a KMS you will also need to add `hadoop.kms.proxyuser.livy.groups` and `hadoop.kms.proxyuser.livy.hosts` into the KMS proxy kms-site configuration.

#### Zeppelin Configuration

To configure Zeppelin to talk to a secure Livy instance, disable the Annonymous Allowed option and enable the Shiro Enabled option. Other configuration options such as disabling public notes and isolating the interpreter scope per user should also be considered along with SSL encryption to prevent login credentials being transfered in plaintext.

Then provide a Shiro configuration into the shiro.ini safety valve. See section below for information.

### SSL

SSL can be enabled on the Zeppelin web UI using the standard CM configuration details.

There is also an option to enable SSL on Livy, however this hasn't been tested. Zeppelin authenticates with Livy using SPNEGO so credentials are not transfered between the two services.

### Shiro

When using authentication in Zeppelin you must provide your own Shiro configuration into the shiro.ini safety value.

The easiest option is to use PAM authentication, restricting access to Zeppelin to those users that can SSH to the machine:

```
# Sample configuration that uses PAM for login and allows users to restart interpreters
[users]

[main]
pamRealm=org.apache.zeppelin.realm.PamRealm
pamRealm.service=sshd
sessionManager = org.apache.shiro.web.session.mgt.DefaultWebSessionManager
securityManager.sessionManager = $sessionManager
securityManager.sessionManager.globalSessionTimeout = 86400000
shiro.loginUrl = /api/login

[roles]
role1 = *
role2 = *
role3 = *
admin = *

[urls]
/api/version = anon
/api/interpreter/setting/restart/** = authc
/api/interpreter/** = authc, roles[admin]
/api/configurations/** = authc, roles[admin]
/api/credential/** = authc, roles[admin]
/** = authc
```

## Tested

The following features are tested:
* Kerberos for Livy
* Kerberos for Zeppelin
* Shiro PAM authentication
* SSL on Zeppelin
* Hive context
* Spark and Spark2 (Hive on Spark 1.6 in cluster-mode doesn't work due to [this](https://issues.apache.org/jira/browse/SPARK-18160))
* Loading custom packages into Livy using `livy.spark.jars.packages`

The following features have not been tested:
* SSL on Livy for SSL communication between Zeppelin and Livy
