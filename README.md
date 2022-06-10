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





