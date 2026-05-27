# LXC Proxmox Bootstrap

Ansible playbook para aprovisionar y mantener contenedores LXC en Proxmox VE. Actualiza el sistema, instala paquetes, despliega archivos y instala herramientas desde fuente.

## Requisitos

- Ansible >= 2.9
- Acceso SSH a los contenedores LXC
- Python 3 en los hosts remotos

## Inicio rapido

1. Clonar el repositorio:

```bash
git clone https://github.com/TU_USER/lxc-proxmox-boostrap.git
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
├── ansible.cfg                # Configuracion de Ansible
├── inventory.ini.example      # Template del inventario
├── playbook.yml               # Playbook principal
├── files/                     # Archivos a desplegar en los hosts
│   ├── test.sh
│   └── test.txt
└── roles/
    ├── system_update/         # apt update + upgrade
    │   └── tasks/main.yml
    ├── packages/              # Instalacion de paquetes base
    │   └── tasks/main.yml
    ├── copy_files/            # Despliegue de archivos a /opt/scripts
    │   └── tasks/main.yml
    └── git_source_install/    # Instalacion de herramientas desde source
        └── tasks/main.yml
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

# Solo instalar herramientas desde source
ansible-playbook playbook.yml --tags "install_xtop"

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

## Roles

| Role | Tags | Descripcion |
|------|------|-------------|
| `system_update` | `update`, `upgrade` | `apt update` + `apt upgrade --safe` con autoremove y autoclean |
| `packages` | `packages` | Instala paquetes base definidos en la lista del rol |
| `copy_files` | `copy_files` | Crea `/opt/scripts` y despliega archivos con permisos diferenciados (0755 scripts, 0644 configs) |
| `git_source_install` | `install_xtop`, `check_version`, `show_version` | Instala herramientas desde source y verifica version |

## Personalizacion

### Agregar paquetes

Editar `roles/packages/tasks/main.yml` y agregar a la lista:

```yaml
- name: Instalar paquetes basicos
  apt:
    name:
      - curl
      - git
      - htop
    state: present
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

Agregar tasks en `roles/git_source_install/tasks/main.yml` siguiendo el patron existente:

```yaml
- name: Instalar herramienta
  shell: curl -fsSL URL_DEL_INSTALLER | bash
  args:
    creates: /usr/local/bin/herramienta
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

`ansible.cfg` incluye defaults:

| Parametro | Valor | Descripcion |
|-----------|-------|-------------|
| `inventory` | `inventory.ini` | Inventario por defecto |
| `roles_path` | `roles` | Ruta de roles |
| `retry_files_enabled` | `False` | Sin archivos .retry |
| `host_key_checking` | `False` | Sin verificacion de host key |
| `timeout` | `30` | Timeout de conexion SSH |
| `become` | `True` | Escalacion de privilegios activa |
