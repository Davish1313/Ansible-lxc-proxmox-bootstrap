# LXC Proxmox Bootstrap

Ansible playbook para aprovisionar y mantener contenedores LXC en Proxmox VE. Actualiza el sistema, instala paquetes, despliega archivos, instala herramientas desde fuente, despliega Grafana Alloy para observabilidad y bootstrapea Python/pip en servidores que no lo tengan.

## Requisitos

- Ansible >= 2.14
- Acceso SSH a los contenedores LXC
- Python 3 en los hosts remotos (o usar `deploy_pip.yml` si no está presente)

## Inicio rápido

1. Clonar el repositorio:

```bash
git clone https://github.com/TU_USER/lxc-proxmox-bootstrap.git
cd lxc-proxmox-bootstrap
```

2. Copiar y editar el inventario:

```bash
cp inventory.ini.example inventory.ini
# Edita inventory.ini con tus hosts
```

3. Generar clave SSH dedicada para ansible (una sola vez):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_lxc -C "ansible@lxc-bootstrap" -N ""
```

4. Bootstrap inicial (crea usuario `ansible` con sudo + clave SSH en todos los hosts):

```bash
ansible-playbook playbook.yml --tags bootstrap
```

5. Ejecutar playbook completo (provision + mantenimiento):

```bash
ansible-playbook playbook.yml
```

## Estructura

```
.
├── ansible.cfg                   # Configuracion de Ansible (incluye become)
├── inventory.ini.example         # Template del inventario
├── playbook.yml                  # Playbook principal (dos plays: bootstrap + provision)
├── deploy_alloy.yml              # Playbook standalone: despliega Grafana Alloy
├── deploy_pip.yml                # Playbook standalone: bootstrap Python3 + pip
├── inventory.yml                 # Playbook de inventario (reportes por host)
├── inventory-host/               # Directorio de salida de reportes (generado)
└── roles/
    ├── bootstrap_user/           # Creacion usuario ansible + sudo + SSH keys
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── system_update/            # apt update + upgrade
    │   ├── meta/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── packages/                 # Instalacion de paquetes base
    │   ├── meta/main.yml
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── copy_files/               # Despliegue de archivos a /opt/scripts
    │   ├── meta/main.yml
    │   ├── defaults/main.yml     # copy_files_list: [] (poblar con archivos a desplegar)
    │   ├── tasks/main.yml
    │   └── files/                # Archivos fuente: test.sh, test.txt, zsh.sh
    ├── git_source_install/       # Instalacion de herramientas desde source
    │   ├── meta/main.yml
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml     # Cleanup post-instalacion
    ├── alloy/                    # Grafana Alloy (Loki logs + node metrics)
    │   ├── defaults/main.yml     # Config: Loki URL, tenant, paths
    │   ├── tasks/
    │   │   ├── main.yml          # import_tasks: install + configure
    │   │   ├── install.yml       # Instalacion desde repositorio Grafana
    │   │   └── configure.yml     # Templates + validacion de config
    │   ├── handlers/main.yml     # Restart alloy
    │   └── templates/
    │       ├── config.alloy.j2   # Configuracion Alloy (loki + prometheus)
    │       └── alloy-defaults.j2 # Environment defaults (--disable-reporting)
    ├── pip/                      # Instalacion de Python3 + pip (Debian/RedHat/Alpine)
    │   ├── meta/main.yml
    │   └── tasks/main.yml
    └── inventory/                # Reportes de inventario por host
        ├── meta/main.yml
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── templates/
            └── host_report.yml.j2
```

## Inventario

Editar `inventory.ini` con los hosts LXC. Usar `inventory.ini.example` como referencia:

```ini
[lxcs]

hostname1 ansible_host=192.168.0.200
hostname2 ansible_host=192.168.0.202

[lxcs:vars]

ansible_user=root          # Usuario para bootstrap inicial (play 1)
ansible_port=22
ansible_python_interpreter=/usr/bin/python3
```

**Modelo de doble usuario:**

| Fase | Play | ansible_user | Clave SSH | gather_facts | Propósito |
|------|------|--------------|-----------|--------------|-----------|
| Bootstrap | 1 | `root` | `~/.ssh/id_ed25519` | `false` | Crea usuario `ansible` + sudo + deploy clave |
| Provision | 2 | `ansible` | `~/.ssh/ansible_lxc` (dedicada) | `true` (default) | Ejecuta tareas con sudo |

Tras bootstrappear **todos** los hosts, puedes actualizar el inventario para comandos directos:

```ini
[lxcs:vars]
ansible_user=ansible
ansible_port=77
ansible_ssh_private_key_file=~/.ssh/ansible_lxc
ansible_python_interpreter=/usr/bin/python3
```

Para entornos con multiples propositos, se recomienda organizar por subgrupos:

```ini
[lxcs:children]
infra
utils

[infra]
dns ansible_host=192.168.0.202

[utils]
test-server ansible_host=192.168.0.200

[lxcs:vars]
ansible_user=root          # o ansible tras bootstrap completo
ansible_port=22
ansible_python_interpreter=/usr/bin/python3
```

Esto permite ejecutar el playbook solo en un grupo:

```bash
ansible-playbook playbook.yml --tags "update" --limit infra
```

### Generar Reporte de Inventario

El playbook `inventory.yml` genera reportes detallados por host en `inventory-host/` (todas las tasks usan `tags: always`):

```bash
# Todos los hosts
ansible-playbook -i inventory.ini inventory.yml

# Host especifico
ansible-playbook -i inventory.ini inventory.yml --limit hostname1

# Multiples hosts
ansible-playbook -i inventory.ini inventory.yml --limit "host1,host2"
```

El reporte incluye: sistema operativo, CPU, RAM, disco, red, servicios, puertos, usuarios y paquetes instalados.

## Uso

Ejecutar playbook completo (bootstrap idempotente + provision):

```bash
ansible-playbook playbook.yml
```

Solo bootstrap (hosts nuevos o primera vez):

```bash
ansible-playbook playbook.yml --tags bootstrap
```

Solo provision (salta bootstrap si ya existe):

```bash
ansible-playbook playbook.yml --skip-tags bootstrap
```

Ejecutar por tags:

```bash
# Solo actualizar sistema
ansible-playbook playbook.yml --tags "update"

# Solo upgrade de paquetes
ansible-playbook playbook.yml --tags "upgrade"

# Solo instalar paquetes
ansible-playbook playbook.yml --tags "packages"

# Solo desplegar archivos
ansible-playbook playbook.yml --tags "copy_files"

# Instalar xtop (incluye verificacion de version)
ansible-playbook playbook.yml --tags "install_xtop"

# Bootstrap Python3 + pip (servidores sin Python)
ansible-playbook deploy_pip.yml

# Bootstrap Python3 en host especifico
ansible-playbook deploy_pip.yml --limit hostname1

# Desplegar Grafana Alloy en todos los LXC
ansible-playbook deploy_alloy.yml

# Desplegar Grafana Alloy en un host especifico
ansible-playbook deploy_alloy.yml --limit hostname1

# Generar inventario
ansible-playbook -i inventory.ini inventory.yml

# Combinar tags
ansible-playbook playbook.yml --tags "update,packages"
```

Limitar a un host:

```bash
ansible-playbook playbook.yml --limit hostname1
```

Verificar conexion:

```bash
ansible lxcs -m ping
```

El playbook ejecuta un pre-flight check automatico de SSH connectivity antes de aplicar cambios. Si algun host no es reachable, falla temprano con un error claro.

## Roles

| Role | Tags | Descripcion |
|------|------|-------------|
| `bootstrap_user` | `bootstrap` | Crea usuario `ansible` con sudo NOPASSWD y despliega clave SSH ed25519 |
| `system_update` | `update`, `upgrade` | `apt update` + `apt upgrade --safe` con autoremove y autoclean |
| `packages` | `packages` | Instala paquetes base (configurable via `packages_essential_packages`) |
| `copy_files` | `copy_files` | Crea `/opt/scripts` y despliega archivos desde `roles/copy_files/files/` según `copy_files_list`. Por defecto vacío — poblar en defaults, grupo o vars de host |
| `git_source_install` | `install_xtop` | Instala xtop desde `.deb` con verificacion SHA256, version check y cleanup via handler |
| `alloy` | — | Despliega Grafana Alloy (Loki logs collector + node metrics) via `deploy_alloy.yml`. Standalone, no incluido en `playbook.yml` |
| `pip` | `pip` | Instala Python3 + pip3 + venv via `deploy_pip.yml`. Standalone, con raw bootstrap para servidores sin Python. Multi-distro (Debian/RedHat/Alpine) |
| `inventory` | `inventory` | Genera reportes de inventario por host en `inventory-host/` |

## Seguridad

### Modelo de acceso dual

El playbook implementa **separación de privilegios** usando dos usuarios SSH:

```
┌─────────────────────────────────────────────────────────────┐
│  Controlador (tu máquina)                                   │
│  ~/.ssh/id_ed25519       →  root@host  (solo bootstrap)     │
│  ~/.ssh/ansible_lxc      →  ansible@host (provision diario) │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Host LXC                                                   │
│  root:  authorized_keys = id_ed25519.pub                    │
│  ansible: authorized_keys = ansible_lxc.pub                 │
│  ansible ALL=(ALL) NOPASSWD:ALL  (via sudoers.d)            │
└─────────────────────────────────────────────────────────────┘
```

**Ventajas:**

| Aspecto | Beneficio |
|---------|-----------|
| Claves separadas | Compromiso de `ansible_lxc` ≠ acceso root |
| ed25519 | Curva elíptica moderna, más rápido y seguro que RSA |
| Sudo auditado | Cada comando privilegiado queda en logs del sistema |
| Principio menor privilegio | `ansible` usuario normal, escala solo cuando requiere |
| Rotación independiente | Cambias clave de ansible sin tocar root |

### Flujo de bootstrap

1. **Play 1** conecta como `root` (clave `~/.ssh/id_ed25519` en vars de play, `gather_facts: false`) → ejecuta `bootstrap_user`
2. `bootstrap_user` crea usuario `ansible`, grupo `sudo`, `/etc/sudoers.d/ansible`
3. Despliega `~/.ssh/ansible_lxc.pub` a `/home/ansible/.ssh/authorized_keys`
4. **Play 2** conecta como `ansible` (clave dedicada) → `become: true` → sudo a root
5. Resto de roles ejecutan con privilegios elevados

## Personalización

### Agregar paquetes

Editar `roles/packages/defaults/main.yml` y agregar a la lista:

```yaml
packages_essential_packages:
  - curl
  - git
  - htop
  - jq
```

O sobreescribir via inventario:

```ini
[lxcs:vars]
packages_essential_packages=["curl", "git", "htop", "jq"]
```

### Agregar archivos a desplegar

`copy_files_list` por defecto está vacío — **debes poblarlo** para que los archivos se desplieguen:

1. Colocar los archivos en `roles/copy_files/files/`
2. Agregar entradas a `roles/copy_files/defaults/main.yml`:

```yaml
copy_files_list:
  # Scripts ejecutables
  - { src: "deploy.sh", mode: "0755" }
  - { src: "backup.sh", mode: "0755" }
  # Configuraciones
  - { src: "nginx.conf", mode: "0644" }
  - { src: ".env.template", mode: "0644" }
  # Servicios systemd
  - { src: "myapp.service", mode: "0644" }
  # Scripts con shebang (zsh, python, etc.)
  - { src: "zsh.sh", mode: "0755" }
  - { src: "sync.py", mode: "0755" }
  # Archivos planos / datos
  - { src: "allowed_ips.txt", mode: "0644" }
  - { src: "crontab", mode: "0600" }
```

> ⚠️ El listado se hace explícitamente por seguridad: ningún archivo se despliega sin estar declarado en `copy_files_list`. Si la lista queda vacía, la tarea se salta (`skipping`).

### Agregar herramientas desde source

Agregar tasks y handler de cleanup en `roles/git_source_install/` siguiendo el patron existente:

`tasks/main.yml`:
```yaml
- name: Check if herramienta is already installed
  ansible.builtin.command: herramienta --version
  register: git_source_install_herramienta_installed
  changed_when: false
  failed_when: false
  tags: install_herramienta

- name: Download herramienta deb package
  ansible.builtin.get_url:
    url: "URL_DEL_DEB"
    dest: /tmp/herramienta.deb
    checksum: "sha256:HASH"
    mode: "0644"
  when: git_source_install_herramienta_installed.rc != 0
  tags: install_herramienta

- name: Install herramienta from local deb
  ansible.builtin.apt:
    deb: /tmp/herramienta.deb
    state: present
  when: git_source_install_herramienta_installed.rc != 0
  notify: Remove downloaded deb package
  tags: install_herramienta
```

`handlers/main.yml`:
```yaml
- name: Remove downloaded deb package
  ansible.builtin.file:
    path: /tmp/herramienta.deb
    state: absent
```

### Variables por host

Si un host requiere configuracion diferente, sobreescribir variables en el inventario:

```ini
[lxcs]
hostname1 ansible_host=192.168.0.200
hostname2 ansible_host=192.168.0.202 ansible_port=2222 ansible_user=ubuntu
```

## Configuracion

`ansible.cfg` incluye defaults y privilege escalation:

| Parametro | Valor | Descripcion |
|-----------|-------|-------------|
| `inventory` | `inventory.ini` | Inventario por defecto |
| `roles_path` | `roles` | Ruta de roles |
| `retry_files_enabled` | `False` | Sin archivos .retry |
| `timeout` | `30` | Timeout de conexion SSH |
| `become` | `True` | Escalacion de privilegios activa |
| `become_method` | `sudo` | Metodo de escalacion |
| `become_user` | `root` | Usuario destino |

> **Nota:** El playbook define `ansible_user` y `ansible_ssh_private_key_file` por play (vars de play), lo que pisa cualquier valor del inventario. El inventory solo afecta comandos ad-hoc (`ansible lxcs -m ping`).

## Validacion

El proyecto pasa `ansible-lint` con profile `production`:

```bash
ansible-lint
```

Para verificar syntax sin ejecutar:

```bash
ansible-playbook playbook.yml --syntax-check
```

## Archivos Ignorados

`.gitignore` excluye:

- `inventory.ini` — datos sensibles de hosts
- `inventory-host/` — reportes generados
- `*.retry` — reintentos de Ansible
- `*.log` — logs
- `notas.txt` — notas personales