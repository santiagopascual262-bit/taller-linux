# Taller Linux — Automatización con Ansible

Proyecto de automatización con Ansible para el despliegue de una aplicación web PHP con base de datos MariaDB, sobre servidores Ubuntu (base de datos) y CentOS (servidor web).

> El detalle de la consigna del obligatorio se encuentra en [`OBLIGATORIO.md`](./OBLIGATORIO.md). Este README documenta el estado actual del código tal como quedó implementado en el repositorio.

## Arquitectura

El inventario (`inventory/hosts.ini`) define los siguientes hosts y grupos:

- **Grupo `ubuntu`** — host `ubuntu01` (IP `10.0.2.100`): servidor Ubuntu (también actúa como base de datos).
- **Grupo `ubuntu`** — host `ubuntu02` (IP `10.0.2.101`): servidor Ubuntu.
- **Grupo `database`** — host `ubuntu-db` (IP `10.0.2.100`): alias de `ubuntu01`, target de los playbooks de base de datos.
- **Grupo `centos`** — host `centos01` (IP `10.0.2.15`): servidor CentOS Stream, web.
- **Grupo `centos`** — host `centos02` (IP `10.0.2.3`): servidor CentOS Stream, web.
- **Grupo `linux`** — grupo padre que agrupa `centos` + `ubuntu`.

En resumen: **MariaDB** corre sobre Ubuntu (`ubuntu-db`, que es el mismo host que `ubuntu01`), y **Apache + PHP** corren sobre los nodos CentOS, que consultan la base de datos remota.

`inventory/group_vars/linux.yaml` fija el usuario de conexión para todo el grupo `linux`:

```yaml
ansible_user: sysadmin
```

La autenticación SSH y el `become` (sudo) se resuelven fuera del repo (clave SSH / agente / `--ask-pass` según cómo se ejecute), ya que el inventario no contiene contraseñas en texto plano.

## Estructura del repositorio

```
taller-linux/
├── collections/
│   └── requirements.yaml       # Colecciones de Ansible requeridas
├── files/
│   ├── cumples.sql             # Datos iniciales de la base "cumples"
│   └── jail.local              # Configuración de fail2ban
├── inventory/
│   ├── hosts.ini                # Inventario (hosts y grupos)
│   └── group_vars/
│       └── linux.yaml           # Variables comunes al grupo "linux"
├── playbooks/
│   ├── hardening.yaml           # Hardening inicial de Ubuntu (UFW + fail2ban)
│   ├── basededatos3.yaml        # Versión final (idempotente, vault, tasks/)
│   ├── backup_mariadb.yaml      # Backup automático diario de MariaDB
│   └── servidorweb3.yaml        # Instalación y configuración del servidor web
│   └── site.yaml                # Playbook integrador
├── tasks/
│   └── initialize_root.yaml     # Tarea reutilizable: fija password root de MariaDB
├── templates/
│   └── cumple.j2                # Plantilla PHP de la app (index.php)
│   ├── backup_mysql.sh.j2       # Script de backup de MariaDB
│   └── my_backup.cnf.j2         # Credenciales para mysqldump (permisos 0600)
├── vars/
│   └── database.yaml            # Variables de la BD, cifradas con Ansible Vault
├── OBLIGATORIO.md               # Consigna del obligatorio
├── LICENSE                      # CC0 1.0 Universal
└── README.md
```

## Requisitos previos

- Ansible instalado en el nodo de control (`ansible-core >= 2.14` recomendado).
- Acceso SSH a los servidores del inventario, con un usuario (`sysadmin`) con privilegios de `sudo`.
- Python instalado en los hosts remotos (requisito estándar de Ansible).
- La contraseña del **Ansible Vault** usada para cifrar `vars/database.yaml` (pedirla a quien gestione el proyecto si no se cuenta con ella).

## Instalación de colecciones de Ansible

`collections/requirements.yaml` declara las tres colecciones que usan los playbooks (`community.general` para UFW, `community.mysql` para `mysql_user`/`mysql_db`, y `ansible.posix` para `firewalld`/`seboolean`):

```yaml
collections:
  - name: community.general
  - name: community.mysql
  - name: ansible.posix
```

Instalarlas antes de correr cualquier playbook:

```bash
ansible-galaxy collection install -r collections/requirements.yaml
```

Verificar que quedaron instaladas:

```bash
ansible-galaxy collection list | grep -E "community.general|community.mysql|ansible.posix"
```

## Variables de la base de datos

`vars/database.yaml` está cifrado con Ansible Vault (contiene `DB_DBASE`, `DB_USER`, `DB_PASS`, `DB_ROOT_PW` y `DB_SERVER`, consumidas tanto por los playbooks de base de datos como por la plantilla `cumple.j2`). Para editarlo o ver su contenido:

```bash
ansible-vault view vars/database.yaml
ansible-vault edit vars/database.yaml
```

Cualquier playbook que dependa de este archivo debe ejecutarse pasando la contraseña del vault, por ejemplo con `--ask-vault-pass` o `--vault-password-file`.

## Playbooks

### 1. `playbooks/hardening.yaml` — Hardening inicial (Ubuntu)

Se aplica sobre el grupo `ubuntu` (`ubuntu01`, `ubuntu02`):

1. Actualiza todos los paquetes instalados (`apt upgrade`) y reinicia el servidor si hubo cambios.
2. Instala y resetea **UFW**: política por defecto `deny` en tráfico entrante, `allow` en saliente.
3. Habilita explícitamente el acceso SSH (`OpenSSH`) y activa UFW.
4. Instala **fail2ban** y despliega `files/jail.local` (`bantime=10m`, `findtime=10m`, `maxretry=5`, jail `sshd` activo).
5. Arranca y habilita el servicio `fail2ban`.

```bash
ansible-playbook -i inventory/hosts.ini playbooks/hardening.yaml --ask-become-pass --ask-vault-pass
```

> Al aplicar `policy: deny` en el tráfico entrante y solo permitir SSH, cualquier otro puerto (por ejemplo 3306 para MariaDB) queda bloqueado hasta que el playbook de base de datos abra el puerto correspondiente en UFW. Por eso conviene correr `hardening.yaml` **antes** que `basededatos3.yaml`.

### 2. `playbooks/basededatos3.yaml` — Base de datos MariaDB (versión final)

Se aplica sobre el grupo `database` (host `ubuntu-db`). Usa las variables cifradas de `vars/database.yaml`:

1. Instala `mariadb-server`, `python3-pymysql` y `ufw`.
2. Inicia y habilita el servicio `mariadb`.
3. Configura `bind-address = 0.0.0.0` en `50-server.cnf` para aceptar conexiones remotas (reinicia MariaDB solo si hubo cambios, vía handler).
4. Abre el puerto `3306/tcp` en UFW. Solo para servidores de aplicación web
5. Verifica si la contraseña de `root` de MariaDB ya está inicializada (login por socket unix sin password). Si **no** lo está, delega en `tasks/initialize_root.yaml`, que fija `DB_ROOT_PW` para `root@localhost` y `root@127.0.0.1`.
6. Copia `files/cumples.sql` al servidor e importa la base `cumples` (tabla `cumpleanios`, con datos de ejemplo).
7. Crea el usuario `DB_USER` (`intranet`) con privilegios `ALL` sobre `DB_DBASE`, accesible desde cualquier host (`%`).

```bash
ansible-playbook -i inventory/hosts.ini playbooks/basededatos3.yaml --ask-vault-pass --ask-become-pass
```

**Comportamiento esperado:**
- 1ra ejecución: la tarea de verificación de password de root falla (`ignore_errors`) porque todavía no hay password seteada → se dispara `initialize_root.yaml` (`changed`) → se importa `cumples.sql` → se crea el usuario `intranet`.
- Ejecuciones siguientes: la verificación de password ya tiene éxito (login con la password vigente), por lo que la tarea de inicialización se saltea (`skipping`) y el resto de las tareas quedan `ok` (idempotentes), sin cambios adicionales.

> `basededatos.yaml` y `basededatos2.yaml` son versiones previas conservadas en el repo a modo de historial de la implementación (la primera con variables hardcodeadas en el propio playbook, la segunda ya usando `vars_files` pero sin el chequeo idempotente de la password de root). El playbook vigente y recomendado para ejecutar es **`basededatos3.yaml`**.

### 3. `playbooks/servidorweb3.yaml` — Servidor web (CentOS)

Se aplica sobre el grupo `centos` (`centos01`, `centos02`), también usando `vars/database.yaml`:

1. Instala y habilita **firewalld**.
2. Abre los servicios `http` y `https` (permanente + inmediato).
3. Instala y habilita **Apache** (`httpd`).
4. Instala `php`, `php-mysqlnd`, `php-fpm` y `python3-libsemanage`; habilita `php-fpm`.
5. Despliega la plantilla `templates/cumple.j2` como `/var/www/html/index.php`, reemplazando `DB_SERVER`, `DB_USER`, `DB_PASS` y `DB_DBASE` por los valores reales.
6. Habilita los booleanos de **SELinux** `httpd_can_network_connect` y `httpd_can_network_connect_db` para que Apache/PHP puedan conectarse a la base remota (reinicia Apache vía handler si hubo cambios).

```bash
ansible-playbook -i inventory/hosts.ini playbooks/servidorweb3.yaml --ask-vault-pass --ask-become-pass
```
### 4. `playbooks/backup_mariadb.yaml` — Backup automático de MariaDB

Se aplica sobre el grupo `database` (host `ubuntu-db`). Usa las variables cifradas de `vars/database.yaml` (concretamente `DB_ROOT_PW` y `DB_DBASE`):

1. Crea el directorio `/var/backups/mariadb` (permisos `0700`, solo accesible por `root`).
2. Despliega `/root/.my_backup.cnf` (permisos `0600`), un archivo de credenciales de MySQL con el usuario y password de `root`, para que `mysqldump` no necesite recibir la contraseña como argumento (evita exponerla en `ps aux`).
3. Despliega el script `/usr/local/bin/backup_mysql.sh`, que ejecuta `mysqldump` usando `--defaults-extra-file` contra el archivo de credenciales, y borra automáticamente los backups con más de `backup_retention_days` (7 por defecto) días de antigüedad.
4. Programa el script vía `cron` para correr todos los días a las 2 AM, con salida redirigida a `/var/log/backup_mysql.log`.

```bash
ansible-playbook -i inventory/hosts.ini playbooks/backup_mariadb.yaml --ask-vault-pass --ask-become-pass
```

**Verificación manual (sin esperar al cron):**

```bash
sudo /usr/local/bin/backup_mysql.sh
sudo ls -la /var/backups/mariadb/
```

> ⚠️ Los backups se guardan en el mismo disco que la base de datos original (`/var/backups/mariadb`, en el mismo filesystem que `/var/lib/mysql`). Esto protege contra errores humanos o corrupción de datos, pero **no** contra una falla del disco físico — para eso haría falta copiar los backups a otro host o almacenamiento externo.

### 5. `playbooks/site.yaml` - Playbook integrador

Ejecuta en orden los playbooks (Hardening, basededatos3 y servidorweb)

## Comando de ejecución

```bash

ansible-playbook -i inventory/hosts.ini playbooks/site.yaml --ask-vault-pass --ask-become-pass
```

El servidor web depende de que la base de datos y el usuario `intranet` ya existan, por lo que `basededatos3.yaml` debe correr antes que `servidorweb3.yaml`.

## La aplicación

- `files/cumples.sql` crea la base `cumples` con la tabla `cumpleanios` (`id`, `nombre`, `fecha`) y carga tres registros de ejemplo (Frodo Baggins, Aragorn, Arwen Undomiel).
- `templates/cumple.j2` genera `index.php`: se conecta a MariaDB con `mysqli`, hace `SELECT nombre, fecha FROM cumpleanios ORDER BY MONTH(fecha), DAY(fecha)` y renderiza una tabla HTML con el listado de cumpleaños.

## Verificación

```bash
# Apache responde en el servidor web
curl -I http://<IP_CENTOS>/index.php

# Conectividad de red hacia la base de datos
nc -zv <IP_UBUNTU_DB> 3306

# Privilegios del usuario de la app
mysql -u root -p -h 127.0.0.1 -e "SHOW GRANTS FOR 'intranet'@'%';"

# fail2ban activo (en el nodo Ubuntu)
sudo fail2ban-client status sshd
```

Si todo está correcto, `http://<IP_CENTOS>` debe mostrar la tabla de cumpleaños sin errores 500, y el `PLAY RECAP` de cada playbook debe terminar con `failed=0`.

## Seguridad

- Las credenciales de la base de datos viven cifradas en `vars/database.yaml` (Ansible Vault); nunca se versionan en texto plano.
- `hardening.yaml` deja UFW con política `deny` entrante por defecto, abriendo solo SSH (y luego el playbook de base de datos abre 3306 puntualmente).
- `fail2ban` protege el servicio SSH (`files/jail.local`) baneando IPs tras 5 intentos fallidos en 10 minutos, por 10 minutos.
- En el servidor web, SELinux se mantiene activo; solo se habilitan los booleanos estrictamente necesarios para que Apache/PHP hablen con la base de datos remota.

