# PROYECTO ASO: DESPLIEGUE AUTOMATIZADO DE SERVICIOS CON ANSIBLE

## 1. DESCRIPCIÓN DEL PROYECTO

Este proyecto automatiza la instalación y configuración de servicios en una máquina con openSUSE 16.0 mediante Ansible.

El despliegue se realiza desde una máquina maestra hacia una máquina clonada que funciona como destino. La comunicación entre ambas se establece mediante SSH.

Servicios incluidos:

- SSH
- Apache
- MariaDB
- Samba
- phpMyAdmin
- vsftpd
- PostgreSQL

Las contraseñas y variables sensibles se gestionan mediante Ansible Vault.

**Importante:** este procedimiento está diseñado para un entorno de máquinas virtuales de laboratorio. La configuración de acceso remoto debe revisarse antes de utilizarse en producción.

## 2. REQUISITOS PREVIOS

**Máquina maestra:**

- openSUSE con Ansible instalado.
- Git instalado.
- Conectividad de red con la máquina clonada.
- Acceso al repositorio del proyecto.
- Contraseña de Ansible Vault.
- Permisos administrativos para configurar y ejecutar Ansible.

**Máquina clonada:**

- openSUSE 16.0 instalado.
- Conexión de red e Internet.
- Repositorios de paquetes disponibles.
- OpenSSH instalado y habilitado.
- Una cuenta con permisos suficientes para realizar las instalaciones y configuraciones.

Ambas máquinas deben poder comunicarse mediante la red configurada en el entorno de virtualización.

## 3. IDENTIFICAR LA IP DE LA MÁQUINA CLONADA

La dirección IP puede cambiar según la red utilizada y la configuración de las máquinas virtuales. Por eso, hay que comprobarla antes de ejecutar Ansible.

**Paso 1.** En la máquina clonada, ejecutar:

```bash
ip -br addr
```

**Paso 2.** Identificar la dirección IPv4 de la interfaz de red activa.

**Paso 3.** Anotar la IP actual.

Ejemplo:

```text
192.168.1.2
```

La dirección anterior es solo un ejemplo. Se debe utilizar la IP real de la máquina clonada.

## 4. VERIFICAR LA CONECTIVIDAD ENTRE LAS MÁQUINAS

Desde la máquina maestra, ejecutar:

```bash
ping -c 4 IP_DE_LA_CLONADA
```

Ejemplo:

```bash
ping -c 4 192.168.1.2
```

Comprobar que se reciben respuestas.

Si no hay respuesta, revisar la IP, la configuración de red de las máquinas virtuales y la conectividad entre ambas.

## 5. PREPARAR SSH EN LA MÁQUINA CLONADA

La máquina clonada necesita un servidor SSH activo para permitir la administración remota.

**Paso 1.** En la consola de la máquina clonada, comprobar si OpenSSH está instalado:

```bash
rpm -q openssh
```

**Paso 2.** Si falta el paquete, actualizar los repositorios e instalarlo:

```bash
zypper refresh
zypper install -y openssh
```

Si PackageKit bloquea la instalación, detenerlo de forma controlada:

```bash
systemctl stop packagekit
```

Después, volver a intentar la instalación. Si el bloqueo continúa, comprobar qué procesos están utilizando el gestor de paquetes:

```bash
ps -ef | grep -E 'packagekit|zypper|rpm' | grep -v grep
```

No finalizar procesos de instalación a la fuerza ni eliminar archivos de bloqueo sin comprobar qué está ocurriendo.

**Paso 3.** Habilitar e iniciar SSH:

```bash
systemctl enable --now sshd
```

**Paso 4.** Verificar que esté activo:

```bash
systemctl is-active sshd
```

El resultado esperado es:

```text
active
```

**Paso 5.** Comprobar que el servidor escucha en el puerto 22:

```bash
ss -lntp | grep ':22 '
```

Instalar OpenSSH no habilita automáticamente el acceso de root ni la autenticación por contraseña. Estos aspectos dependen de la configuración del servidor.

## 6. CONFIGURAR Y PROBAR LA AUTENTICACIÓN SSH

Desde la máquina maestra, probar la conexión:

```bash
ssh root@IP_DE_LA_CLONADA
```

Si aparece una advertencia sobre la huella del servidor, comprobar que la IP corresponde a la máquina clonada correcta antes de aceptarla.

Si se utiliza acceso de root mediante contraseña, verificar que la cuenta tenga una contraseña válida y que la configuración de SSH permita ese método.

En la máquina clonada, se puede comprobar el estado de la contraseña:

```bash
passwd -S root
```

Si se necesita establecer una contraseña para el laboratorio:

```bash
passwd root
```

En el entorno de esta práctica se puede utilizar un archivo adicional de configuración SSH. Antes de cambiarlo, revisar la configuración existente para evitar sobrescribir ajustes necesarios.

Por ejemplo, si el acceso de root mediante contraseña está autorizado para el laboratorio, el archivo `/etc/ssh/sshd_config.d/00-ansible.conf` puede contener:

```text
PermitRootLogin yes
PasswordAuthentication yes
KbdInteractiveAuthentication yes
```

Validar la configuración:

```bash
/usr/sbin/sshd -t -f /usr/etc/ssh/sshd_config
```

Si no aparecen errores, reiniciar el servicio:

```bash
systemctl restart sshd
systemctl is-active sshd
```

Comprobar los valores efectivos:

```bash
sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication'
```

**Importante:** permitir el acceso de root mediante contraseña es una configuración de laboratorio, no una recomendación para producción.

## 7. INTERCAMBIO DE CLAVES SSH ENTRE LA MAESTRA Y LA CLONADA

La autenticación mediante clave pública permite que la máquina maestra se conecte a la clonada sin tener que introducir la contraseña de la cuenta remota en cada conexión.

Este paso es recomendable cuando se desea utilizar autenticación por clave pública, pero no es necesario repetirlo si la conexión SSH ya funciona con el método configurado y Ansible puede administrar el destino correctamente.

**Paso 1.** En la máquina maestra, comprobar si ya existe una clave SSH:

```bash
ls -la ~/.ssh
```

Buscar archivos como:

```text
id_ed25519
id_ed25519.pub
```

También pueden existir claves RSA u otros nombres de clave.

El archivo terminado en `.pub` es la clave pública. El archivo sin `.pub` es la clave privada y nunca debe compartirse ni copiarse al servidor destino.

**Paso 2.** Si no existe una clave apropiada, generar una:

```bash
ssh-keygen -t ed25519
```

Seguir las instrucciones del programa. Si se establece una frase de contraseña para la clave, será necesario gestionarla cuando se utilice.

**Paso 3.** Copiar la clave pública a la máquina clonada:

```bash
ssh-copy-id root@IP_DE_LA_CLONADA
```

Este comando puede solicitar la contraseña de root de la máquina clonada durante la configuración inicial.

Si `ssh-copy-id` no está disponible, se puede instalar el paquete que lo proporciona o configurar manualmente la clave pública en el archivo `~/.ssh/authorized_keys` de la cuenta remota, respetando los permisos requeridos.

**Paso 4.** Probar la conexión:

```bash
ssh root@IP_DE_LA_CLONADA
```

Si la clave está configurada correctamente, la conexión debería utilizar la clave pública sin solicitar la contraseña de la cuenta remota. Puede solicitar la frase de contraseña de la clave si se configuró una.

**¿Por qué el proyecto funcionó sin hacer este paso manualmente?**

Ansible utiliza el mecanismo de autenticación SSH que tenga disponible. Si ya existía una clave autorizada, se utilizaba un agente SSH o se había configurado otro método de autenticación, el intercambio manual podía no ser necesario.

Por tanto, no se debe asumir que el proyecto está mal configurado solo porque no se ejecutó `ssh-copy-id`. Lo importante es que la autenticación funcione y que Ansible pueda acceder al destino.

Si se utiliza autenticación mediante contraseña y Ansible la solicita, puede ser necesario ejecutar el playbook con `--ask-pass`. Si se necesita elevar privilegios mediante contraseña, puede utilizarse `--ask-become-pass`. Estos parámetros dependen de la configuración real del proyecto.

## 8. CONFIGURAR EL INVENTARIO DE ANSIBLE

En este proyecto, el archivo `hosts` está dentro de la carpeta `inventory`, ubicada en el directorio de Ansible.

La ruta utilizada es:

```text
/etc/ansible/inventory/hosts
```

**Paso 1.** Abrir el inventario en la máquina maestra:

```bash
vim /etc/ansible/inventory/hosts
```

**Paso 2.** Comprobar que exista el grupo utilizado por el playbook:

```ini
[servidores]
192.168.1.2
```

La IP es un ejemplo. Sustituirla por la dirección actual de la máquina clonada.

**Paso 3.** Guardar los cambios y salir de Vim:

- Pulsar `Esc`.
- Escribir `:wq`.
- Pulsar `Enter`.

No almacenar contraseñas en texto plano dentro del inventario.

Si la IP cambia, actualizar el archivo antes de ejecutar el proyecto.

## 9. DESCARGAR Y REVISAR EL PROYECTO

En la máquina maestra, ubicarse en el directorio donde se guardará el proyecto.

Ejemplo:

```bash
cd /root
```

Descargar el repositorio:

```bash
git clone URL_DEL_REPOSITORIO
```

Entrar en la carpeta descargada:

```bash
cd NOMBRE_DEL_REPOSITORIO
```

Revisar los archivos:

```bash
ls -la
```

Confirmar que estén disponibles el playbook principal, los roles, los archivos de configuración y las variables cifradas que requiera el proyecto.

Si el proyecto ya está descargado, verificar que se estén ejecutando los archivos de la ubicación correcta y no copias antiguas.

## 10. REVISAR EL PLAYBOOK Y LOS ROLES

El archivo principal del proyecto es `site.yml`.

Los roles incluidos actualmente son:

```yaml
---
- hosts: servidores
  become: yes
  roles:
    - ssh
    - apache
    - mariadb
    - samba
    - phpmyadmin
    - vsftpd
    - postgresql
```

Comprobar que los roles y sus archivos de configuración estén disponibles.

Revisar especialmente:

**SSH**
- Configuración administrada por el rol.
- Validación antes de reiniciar el servicio.
- Método de autenticación permitido.

**Apache**
- Instalación y estado del servicio.
- Disponibilidad de la página web.

**MariaDB**
- Instalación y estado del servicio.
- Configuración de acceso y credenciales.

**Samba**
- Usuarios del sistema y de Samba.
- Directorios compartidos.
- Permisos de acceso y escritura.
- Configuración de `create mask`.

**vsftpd**
- Archivo `vsftpd.conf`.
- Configuración `write_enable=YES`, si la práctica requiere escritura.
- Permisos del sistema de archivos para los usuarios autorizados.

**PostgreSQL**
- Instalación e inicialización de la base de datos.
- Configuración de conexiones locales y remotas.
- Creación del usuario y la base de datos requeridos.

## 11. PREPARAR ANSIBLE VAULT

Ansible Vault protege las variables sensibles utilizadas durante el despliegue.

Comprobar que el archivo cifrado esperado por el proyecto exista y esté correctamente referenciado.

La contraseña de Vault debe obtenerse mediante el procedimiento acordado para el equipo. No debe guardarse en el README ni publicarse en el repositorio.

La variable `pass_general` utilizada por algunas tareas debe estar definida y disponible cuando se ejecuten esas tareas.

**Importante:** la contraseña de Vault y el valor de `pass_general` son distintos. La primera permite descifrar los datos protegidos; la segunda se utiliza como valor de configuración de los servicios.

## 12. VALIDAR LA CONFIGURACIÓN ANTES DEL DESPLIEGUE

Desde la máquina maestra, ubicarse en el directorio que contiene `site.yml`.

Comprobar el inventario:

```bash
ansible-inventory -i /etc/ansible/inventory/hosts --graph
```

Validar la sintaxis del playbook:

```bash
ansible-playbook -i /etc/ansible/inventory/hosts site.yml --syntax-check
```

Comprobar la conectividad con el grupo de destino:

```bash
ansible servidores -i /etc/ansible/inventory/hosts -m ping
```

El resultado esperado es una respuesta `SUCCESS` con `"ping": "pong"` para la máquina clonada.

Si Ansible no puede conectarse, comprobar la IP, el inventario, SSH y el método de autenticación antes de continuar.

## 13. EJECUTAR EL PLAYBOOK CON VAULT

Una vez comprobadas la conectividad, la autenticación SSH, el inventario y las variables protegidas, ejecutar:

```bash
ansible-playbook -i /etc/ansible/inventory/hosts site.yml --ask-vault-pass
```

**Paso 1.** Introducir la contraseña de Vault cuando se solicite.

**Paso 2.** Esperar a que se ejecuten las tareas de todos los roles.

**Paso 3.** Revisar la salida, especialmente las tareas que hayan fallado o cuyos errores se hayan ignorado.

**Paso 4.** Revisar el resumen final de ejecución.

El objetivo es que la ejecución termine sin hosts inaccesibles ni tareas fallidas. Los errores ignorados también deben investigarse y resolverse cuando correspondan.

Si una tarea falla, identificar la causa, corregir el rol correspondiente y volver a ejecutar el playbook.

## 14. COMPROBAR LOS SERVICIOS DESPLEGADOS

Las siguientes comprobaciones se realizan en la máquina clonada.

**Apache**

```bash
systemctl is-active apache2
curl -I http://127.0.0.1/
```

También se puede probar desde un navegador con conectividad hacia el destino:

```text
http://IP_DE_LA_CLONADA/
```

**MariaDB**

```bash
systemctl is-active mariadb
mysqladmin ping
```

**Samba**

```bash
systemctl is-active smb
testparm -s
```

Después, comprobar el acceso a los recursos compartidos y verificar los permisos que exige la práctica.

**vsftpd**

```bash
systemctl is-active vsftpd
ss -lntp | grep ':21 '
```

Realizar una conexión FTP y comprobar las operaciones permitidas al usuario de prueba. Si la práctica requiere escritura, intentar subir un archivo a una ubicación autorizada.

`write_enable=YES` no sustituye los permisos del sistema de archivos ni las demás restricciones de autenticación.

**PostgreSQL**

```bash
systemctl is-active postgresql
pg_isready
sudo -u postgres psql -c '\du'
sudo -u postgres psql -l
```

Verificar que existan el usuario `smbuser` y la base de datos `app_db` si esos son los nombres configurados en el rol.

**phpMyAdmin**

Abrir en el navegador la ruta configurada por el rol o por el paquete instalado. Confirmar que la página cargue y que el acceso a MariaDB funcione con las credenciales autorizadas.

La ruta exacta debe comprobarse en la configuración efectiva del servidor; no se debe asumir sin probarla.

## 15. CONSIDERACIONES PARA POSTGRESQL EN openSUSE 16.0

El comando `postgresql-setup initdb` puede no estar disponible en esta versión de openSUSE.

Antes de inicializar PostgreSQL en una máquina limpia, verificar el mecanismo de inicialización proporcionado por el paquete instalado y su documentación local.

En una máquina donde PostgreSQL ya se haya inicializado, comprobar:

```bash
test -f /var/lib/pgsql/data/PG_VERSION && echo "PostgreSQL inicializado"
systemctl status postgresql --no-pager
pg_isready
```

No volver a inicializar una base de datos que ya contiene datos. La tarea de inicialización del rol debe ser idempotente y compatible con el procedimiento de openSUSE 16.0.

## 16. SOLUCIÓN DE PROBLEMAS

**No hay respuesta al ping**
- Comprobar la IP actual de la máquina clonada.
- Revisar la red de las máquinas virtuales.
- Actualizar el inventario si la IP ha cambiado.

**SSH no conecta**
- Comprobar que OpenSSH esté instalado.
- Verificar que `sshd` esté activo.
- Revisar el puerto 22 y la red.
- Probar manualmente `ssh root@IP_DE_LA_CLONADA`.

**SSH rechaza la autenticación**
- Revisar el estado de la cuenta remota.
- Comprobar el método de autenticación configurado.
- Si se utilizan claves, verificar que la clave pública esté autorizada en el destino.
- Si se utilizan contraseñas, confirmar que el servidor permite ese método.

**Ansible informa `UNREACHABLE`**
- Verificar la IP y el inventario.
- Probar SSH manualmente.
- Revisar la autenticación y los permisos administrativos.

**zypper está bloqueado**
- Comprobar si PackageKit u otro gestor de paquetes está activo.
- Detener PackageKit de forma controlada si corresponde.
- No eliminar archivos de bloqueo ni finalizar procesos indiscriminadamente.

**PostgreSQL no se inicializa**
- Comprobar si existe `/var/lib/pgsql/data/PG_VERSION`.
- Revisar el estado del servicio.
- Ejecutar `pg_isready`.
- Verificar el procedimiento de inicialización de la versión instalada.
- No inicializar de nuevo una base de datos existente.

**El playbook presenta errores ignorados**
- Localizar la tarea que produjo el error.
- Analizar el mensaje completo.
- Corregir la causa.
- Comprobar el estado real del servicio.

## 17. CRITERIOS DE FINALIZACIÓN

El despliegue se considera validado cuando:

- La máquina maestra y la clonada tienen conectividad.
- SSH funciona mediante el método de autenticación configurado.
- Ansible identifica correctamente el destino.
- El playbook supera la validación de sintaxis.
- Vault permite acceder a las variables protegidas.
- La ejecución termina sin tareas fallidas ni errores ignorados sin resolver.
- Los servicios requeridos están activos.
- Las pruebas funcionales se completan correctamente.
- Los archivos de configuración y las instrucciones están documentados.
- El repositorio no contiene contraseñas ni claves privadas.

No se debe afirmar que un servicio funciona únicamente porque su paquete esté instalado o su proceso esté activo. Hay que realizar las pruebas funcionales correspondientes.



**Fin del README.**
