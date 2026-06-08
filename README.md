# LXC Proxmox Bootstrap

Ansible playbook para aprovisionar y mantener contenedores LXC en Proxmox VE. Actualiza el sistema, instala paquetes, despliega archivos y instala herramientas desde fuente.

## Requisitos

- Ansible >= 2.14
- Acceso SSH a los contenedores LXC
- Python 3 en los hosts remotos

## Inicio rapido

1. Clonar el repositorio:

```bash
git clone https://github.com/TU_USER/lxc-proxmox-bootstrap.git
cd lxc-proxmox-bootstrap
```

2. Copiar y editar el inventario:

```bash
cp inventory.ini.example inventory.ini
```

3. Ejecutar el playbook completo:

```bash
ansible-playbook playbook.yml
```

## Estructura

```
.
├── ansible.cfg                  # Configuracion de Ansible (incluye become)
├── inventory.ini.example        # Template del inventario
├── playbook.yml                 # Playbook principal con pre-flight checks
├── inventory.yml                # Playbook de inventario (reportes por host)
├── inventory-host/              # Directorio de salida de reportes (generado)
├── files/                       # Archivos a desplegar en los hosts
│   ├── test.sh
│   └── test.txt
└── roles/
    ├── system_update/           # apt update + upgrade
    │   ├── meta/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── packages/                # Instalacion de paquetes base
    │   ├── meta/main.yml
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── copy_files/              # Despliegue de archivos a /opt/scripts
    │   ├── meta/main.yml
    │   └── tasks/main.yml
    ├── git_source_install/      # Instalacion de herramientas desde source
    │   ├── meta/main.yml
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── inventory/               # Reportes de inventario por host
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

ansible_user=root
ansible_port=22
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
ansible_user=root
ansible_port=22
ansible_python_interpreter=/usr/bin/python3
```

Esto permite ejecutar el playbook solo en un grupo:

```bash
ansible-playbook playbook.yml --tags "update" --limit infra
```

### Generar Reporte de Inventario

El playbook `inventory.yml` genera reportes detallados por host en `inventory-host/`:

```bash
# Todos los hosts
ansible-playbook -i inventory.ini inventory.yml --tags inventory

# Host especifico
ansible-playbook -i inventory.ini inventory.yml --tags inventory --limit hostname1

# Multiples hosts
ansible-playbook -i inventory.ini inventory.yml --tags inventory --limit "host1,host2"
```

El reporte incluye: sistema operativo, CPU, RAM, disco, red, servicios, puertos, usuarios y paquetes instalados.

## Uso

Ejecutar playbook completo:

```bash
ansible-playbook playbook.yml
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

# Generar inventario
ansible-playbook -i inventory.ini inventory.yml --tags "inventory"

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
| `system_update` | `update`, `upgrade` | `apt update` + `apt upgrade --safe` con autoremove y autoclean |
| `packages` | `packages` | Instala paquetes base (configurable via `packages_essential_packages`) |
| `copy_files` | `copy_files` | Crea `/opt/scripts` y despliega archivos con permisos diferenciados (0755 scripts, 0644 configs) |
| `git_source_install` | `install_xtop` | Instala xtop desde `.deb` con verificacion SHA256 y version check |
| `inventory` | `inventory` | Genera reportes de inventario por host en `inventory-host/` |

## Personalizacion

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

1. Colocar los archivos en `files/`
2. Agregar entradas al loop en `roles/copy_files/tasks/main.yml`:

```yaml
loop:
  - { src: "script.sh", mode: "0755" }
  - { src: "config.txt", mode: "0644" }
  - { src: "otro.sh", mode: "0755" }
```

### Agregar herramientas desde source

Agregar tasks en `roles/git_source_install/tasks/main.yml` siguiendo el patron existente (download con checksum + install local):

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
  tags: install_herramienta
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
