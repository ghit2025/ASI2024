
Proyecto de administración de sistemas informáticos (Mansible): Uso de Docker
Objetivo
El objetivo de este documento es explicar cómo se puede usar Docker como infrasestructura para el desarrollo de la práctica, lo que conlleva varias ventajas:

    Permite usar la misma distribución que se ha utilizado para preparar la práctica (Ubuntu 24.04) sin necesidad de instalarla en una máquina virtual o física.
    Proporciona un entorno virtual ágil que, ante cualquier incidencia que deje corrupto el estado de la máquina, permite crear rápidamente una nueva instancia.
    Posibilita crear de forma eficiente una red de máquinas lo que permite probar las dos últimas fases de la práctica sin necesidad de tener varias máquinas físicas o virtuales.. 

Como punto negativo, Docker no gestiona adecuadamente los volúmenes lógicos dentro de un contenedor lo que causa problemas a la hora de probar el módulo relacionado con esa funcionalidad (lvol), teniéndose que realizar las pruebas fuera de los contenedores.

En cualquier caso, consideramos que puede merecer la pena ya que, como objetivo secundario, permite familiarizarse con el uso de Docker.

Creación de la imagen
En este documento, se propone crear en una máquina Linux, física o virtual, varias instancias de una imagen, cuyo Dockerfile se proporciona, que tiene las siguientes características (corresponden a las requeridas por la práctica de la asignatura):

    Está basada en Ubuntu 24.04.
    Tiene instalado el paquete que corresponde a la funcionalidad servidor de SSH.
    Incluye el paquete que corresponde al editor nano, para poder disponer de un editor dentro del contenedor.
    Dispone del paquete que habilita la funcionalidad de SUDO.
    Crea un usuario con el nombre especificado a la hora de construir la imagen.
    Configura el nuevo usuario para que pueda usar SUDO sin contraseña.
    Establece una disposición de manera que al arrancar un contenedor de esa imagen se active el servicio SSH.
    Especifica que el contenedor ejecutará en el contexto del nuevo usuario tal que su directorio base sea el directorio de trabajo inicial.
    Establece que al activarse la imagen en un contenedor se arrancará un proceso que ejecuta bash. 

El primer paso es, evidentemente, instalar Docker.

Una vez instalado podemos crear la imagen, a la que denominaremos ASI/image, para lo que ejecutaremos este mandato en el directorio donde reside el Dockerfile especificando en el argumento USUARIO cuál será el nombre del usuario en cuyo contexto se ejecutará un contenedor creado con esta imagen:

docker build --build-arg USUARIO=$USER -t ASI/image .

De cara a la práctica, queremos que el usuario sea el mismo que el que está construyendo la imagen.

Finalizada la generación de la imagen, podemos comprobar que se ha creado:

$ docker image ls ASI/image
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
ASI/image    latest    44310349c23d   22 hours ago   250MB

Creación de la infraestructura
Vamos a crear dos instancias de esa imagen que corresponderán a las máquinas que se pretenden gestionar. El equipo virtual o físico en el que hemos realizado la instalación de Docker ejercerá el rol de máquina maestra de control. Nótese que ese equipo solo se usará en las dos últimas fases y puede tener instalada, en principio, cualquier distribución de Linux que permita usar Docker.

A continuación, se indican los pasos que hay que realizar tomando como punto de partida el directorio que contiene el material de apoyo descomprimido.

# en una primera ventana, que corresponde a la máquina de control
$ cd ASI/proyecto_2024/
$ chmod -R 777 . # total accesibilidad desde el contenedor
$ truncate -s 1GiB disco1 disco2 disco3 disco4 # crea 4 ficheros vacíos de 1GiB
# crea dispositivos de bloques de tipo loopback asociados
$ for (( i=1; i<=4; i++ )); do sudo losetup -f --show disco$i; done
/dev/loop14
/dev/loop15
/dev/loop16
/dev/loop17

Asumimos que en ese equipo "anfitrión" se han creado ya las claves que se usarán en el acceso SSH requerido por las dos últimas fases. Si no fuera así, hay que ejecutar:

$ ssh-keygen
............
# comprobar el nombre del fichero que contiene la clave pública:
$ ls -l ~/.ssh/*.pub
-rw-r--r-- 1 pp pp 92 sep 27 08:38 /home/fperez/.ssh/id_ed25519.pub

Usaremos el nombre de ese fichero en el mandato que crea el contenedor. Debe adaptarlo al nombre de fichero que aparezca en su equipo que dependerá de qué tipo de clave se use (en este caso, ed25519).

Ahora creamos el primer "equipo" que hay que administrar:

# en una segunda ventana, que corresponde a la 1ª máquina gestionada
$ cd ASI/proyecto_2024/
$ docker run --rm -it --name equipo1 -v /home/$USER/.ssh/id_ed25519.pub:/home/$USER/.ssh/authorized_keys -v $PWD:/ASI --device /dev/loop14 --device /dev/loop15 ASI/image
To run a command as administrator (user "root"), use "sudo ".
See "man sudo_root" for details.

fperez@2d0192804832:~$

Revisemos los argumentos del mandato:

    --rm: el contenedor desaparece (se elimina el "equipo") cuando termina.
    -it: modo de operación interactivo con un terminal asociado.
    --name: nombre del contenedor.
    -v /home/$USER/.ssh/id_ed25519.pub:/home/$USER/.ssh/authorized_keys: enlaza el fichero con la clave pública en el anfitrión con el fichero de claves autorizadas en el contenedor lo que permite que se pueda hacer un SSH a ese usuario usando claves sin necesidad de contraseña. Recuerde que debe usar el nombre del fichero que contenga la clave pública en su equipo.
    -v $PWD:/ASI: deja accesible el directorio actual, que contiene los módulos,, a través del directorio /ASI del contenedor.
    --device: da visibilidad de esos discos dentro del contenedor.
    ASI/image: imagen que se instancia. 

De manera similar, se crea en otra ventana otra instancia con otro nombre de equipo y los otros dos discos:

# en una tercera ventana, que corresponde a la 2ª máquina gestionada
$ cd ASI/proyecto_2024/
$ docker run --rm -it --name equipo2 -v /home/$USER/.ssh/id_ed25519.pub:/home/$USER/.ssh/authorized_keys -v $PWD:/ASI --device /dev/loop16 --device /dev/loop17 ASI/image
To run a command as administrator (user "root"), use "sudo ".
See "man sudo_root" for details.

fperez@0e04df876b67:~$

En este punto podemos probar la conectividad. Averiguamos la IP de cada "máquina".

Primera máquina:

fperez@260309586f70:~$ hostname -I
172.17.0.2

Segunda máquina:

fperez@0e04df876b67:~$ hostname -I
172.17.0.3

Y comprobamos que se puede crear un fichero en el directorio raíz de ambos equipos mediante un SSH desde el anfitrión:

$ ssh 172.17.0.2 sudo touch /flag
$ ssh 172.17.0.3 sudo touch /flag

Primera máquina:

fperez@260309586f70:~$ ls /flag
/flag

Segunda máquina:

fperez@0e04df876b67:~$ ls /flag
/flag

Pruebas
Para probar los módulos usamos directamente el primer equipo (solo se muestran pruebas básicas para comprobar la viabilidad de la plataforma; consulte el enunciado para ver pruebas adicionales).

Empezamos por el primer módulo (package):

fperez@260309586f70:~$ cd /ASI
fperez@260309586f70:/ASI$ sudo ./package name=hello
fperez@260309586f70:/ASI$ hello
Hello, world!

Continuamos con el segundo (lvg):

fperez@260309586f70:/ASI$ sudo ./package name=lvm2
fperez@260309586f70:/ASI$ sudo ./lvg state=present vg=VG1 pvs=/dev/loop14,/dev/loop15
fperez@260309586f70:/ASI$ sudo vgs
  VG  #PV #LV #SN Attr   VSize VFree
  VG1   2   0   0 wz--n- 1.99g 1.99g

Se puede probar, por tanto, este módulo en este entorno basado en Docker. Sin embargo, el módulo lvol da problemas si se prueba dentro de un contenedor debido a que la solución de contenedores usada por Docker no gestiona adecuamente la gestión de volúmenes lógicos dentro de un contenedor. En consecuencia, este módulo hay que probarlo directamente fuera de un contenedor.

Pasamos a probar la penúltima fase ejecutando desde el anfitrión:

$ ./mansible "172.17.0.2 172.17.0.3" user name=test_user
172.17.0.2 | CHANGED
172.17.0.3 | CHANGED

Primera máquina:

fperez@260309586f70:/ASI$ getent passwd test_user
test_user:x:1002:1002::/home/test_user:/bin/bash

Segunda máquina:

fperez@0e04df876b67:/ASI$ getent passwd test_user
test_user:x:1002:1002::/home/test_user:/bin/bash

Finalmente, usamos en el anfitrión este playbook (prueba.play) para probar la última fase:

172.17.0.2 172.17.0.3
crea_usuario user name=test_user state=present

El resultado de la ejecución es:

$ ./mansible-playbook prueba.play
PLAY [prueba.play] *********************************************

TASK [crea_usuario] *********************************************
ok: [172.17.0.2]
ok: [172.17.0.3]

PLAY RECAP *********************************************
172.17.0.2: ok=1 changed=0 unreachable=0 failed=0
172.17.0.3: ok=1 changed=0 unreachable=0 failed=0

