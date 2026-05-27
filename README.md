# LXC Proxmox Bootstrap

Ansible playbook para aprovisionar y mantener contenedores LXC en Proxmox VE. Actualiza el sistema, instala paquetes, despliega scripts de monitoreo y instala herramientas desde fuente.

## Requisitos

- Ansible >= 2.9
- Acceso SSH a los contenedores LXC
- Python 3 en los hosts remotos

## Estructura

```
.
├── ansible.cfg
├── inventory.ini
├── playbook.yml
├── files/
│   ├── revisar.sh          # Script de reporte de estado del sistema
│   └── notas.txt
└── roles/
    ├── system_update/       # apt update + upgrade
    ├── packages/            # Instalación de paquetes base
    ├── send_files/          # Despliegue de archivos a /opt/scripts
    └── git_source_install/  # Instalación de xtop desde source
```

## Inventario

Editar `inventory.ini` con los hosts LXC:

```ini
[lxcs]
test    ansible_host=192.168.0.200
adguard ansible_host=192.168.0.202

[lxcs:vars]
ansible_user=root
ansible_port=77
ansible_python_interpreter=/usr/bin/python3
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

# Solo desplegar scripts
ansible-playbook playbook.yml --tags "copy_files"

# Solo instalar xtop
ansible-playbook playbook.yml --tags "install_xtop"

# Combinar tags
ansible-playbook playbook.yml --tags "update,packages"
```

Limitar a un host:

```bash
ansible-playbook playbook.yml --limit test
```

Verificar conexión:

```bash
ansible lxcs -m ping
```

## Roles

| Role | Tags | Descripción |
|------|------|-------------|
| `system_update` | `update`, `upgrade` | `apt update` + `apt upgrade --safe` con autoremove |
| `packages` | `packages` | Instala paquetes base (`curl`, etc.) |
| `send_files` | `copy_files` | Crea `/opt/scripts` y despliega `revisar.sh` (0755) y `notas.txt` (0644) |
| `git_source_install` | `install_xtop`, `check_version`, `show_version` | Instala [xtop](https://github.com/xscriptor/xtop) desde source y verifica versión |

## revisar.sh

Script de reporte de estado del sistema que se despliega en `/opt/scripts/`. Muestra:

- SO, IP, uptime, load average
- RAM disponible, uso de swap, uso de disco
- Cuentas con shell y sesiones activas
- Servicios fallidos
- Puertos a la escucha con proceso y consumo CPU/RAM
- Contenedores Docker activos (si aplica)
- Entorno Proxmox: VMs y LXC (si aplica)
- Top 10 servicios activos

Ejecución remota:

```bash
ansible lxcs -a "/opt/scripts/revisar.sh"
```

## Configuración

`ansible.cfg` incluye defaults sensibles: inventario, roles path, sin retry files, sin host key checking y become habilitado.
