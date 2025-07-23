# Writeup: Pequeñas Mentirosas – DockerLabs

La máquina **Pequeñas Mentirosas** es un entorno Docker que expone un servicio HTTP en el puerto 80 y un servicio SSH en el puerto 22. Durante el proceso de enumeración inicial se identificaron estos dos puertos abiertos, pero el servicio web no reveló ninguna pista útil al principio.

El objetivo de esta máquina es ganar acceso al sistema mediante fuerza bruta por SSH, aprovechando pistas sutiles sobre credenciales, y luego escalar privilegios explotando la capacidad de ejecutar scripts en Python como root.

## Enumeración

Comenzamos accediendo al puerto 80 y realizando un análisis básico de directorios mediante `ffuf` y `gobuster`. Sin embargo, no se reveló nada útil con los siguientes comandos:

```bash
ffuf -u http://172.17.0.2/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fs 85
```

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/dirb/wordlists/common.txt -o directorios.txt
```

Posteriormente se lanzó un escaneo con **Nikto**, que tampoco identificó vulnerabilidades relevantes. Probamos entonces escanear archivos potenciales con extensiones comunes:

```bash
ffuf -u http://172.17.0.2/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -e .php,.txt,.bak,.zip,.tar,.log,.conf,.inc -mc 200
```

Tampoco hubo éxito. Ni siquiera con una lista más precisa:

```bash
ffuf -u http://172.17.0.2/./FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.bak,.zip,.tar,.log,.conf,.inc -mc 200
```

Al ver que el servicio web no ofrecía ninguna entrada clara, decidimos cambiar de enfoque.

## Ataque por fuerza bruta a SSH

La única pista disponible en la página web mencionaba: *"Encuentra la clave para A en los archivos."* Esto nos llevó a suponer que el usuario podría llamarse **a** o **A**.

Se intentó un ataque de fuerza bruta por SSH usando `hydra`, con el usuario **a** (en minúsculas) y el diccionario rockyou:

```bash
hydra -l a -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -vV -f -o resultados.txt
```

La contraseña encontrada fue `secret`, lo que nos permitió iniciar sesión por SSH como el usuario `a`.

## Acceso por SSH y post-explotación inicial

Una vez dentro de la máquina como el usuario `a`, no encontramos archivos visibles en su directorio personal. Para realizar una enumeración más profunda, decidimos subir **LinPEAS** desde nuestra máquina atacante usando `scp`:

```bash
scp linpeas.sh a@172.17.0.2:/tmp/
```

Le dimos permisos de ejecución y ejecutamos el script en la máquina víctima. Sin embargo, LinPEAS no reveló ninguna vulnerabilidad aprovechable ni binarios interesantes con permisos elevados.

## Enumeración de usuarios y segundo ataque por fuerza bruta

Tras explorar otros directorios, notamos que existía otro usuario en el sistema llamado **spencer**, al revisar `/home/` y también al inspeccionar `/etc/passwd`.

Decidimos lanzar un segundo ataque de fuerza bruta por SSH, esta vez contra el usuario `spencer`. El resultado fue exitoso: la contraseña era `password1`, lo que nos dio acceso a la máquina como este nuevo usuario.

## Escalada de privilegios a root

Ya como `spencer`, ejecutamos el comando `sudo -l` y descubrimos que teníamos permisos para ejecutar Python3 como root:

```bash
sudo python3 -c 'import os; os.system("/bin/bash")'
```

Este comando nos proporcionó una shell con privilegios de root directamente, completando así la escalada de privilegios.

## Flags

- **Acceso inicial (user `a`)**: Se logró mediante fuerza bruta por SSH con el diccionario rockyou.
- **Acceso a `spencer`**: Otro ataque por fuerza bruta reveló la contraseña `password1`.
- **Acceso root**: Se obtuvo al ejecutar Python3 como root gracias a permisos concedidos a `spencer`.