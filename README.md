# Ansible Automation Suite - Infraestructura Bizagi

Suite completa de automatización Ansible para gestión de infraestructura Bizagi, incluyendo sincronización de archivos, gestión de servicios, tareas programadas, pruebas de conectividad y notificaciones.

## Descripción General

Este proyecto proporciona una colección integrada de playbooks y roles de Ansible diseñados específicamente para la gestión automatizada de infraestructura Bizagi en ambientes Windows y Linux. Incluye capacidades de gestión activo/pasivo, sincronización de archivos entre ambientes de producción y contingencia, gestión de servicios Windows, validación de conectividad y sistema de notificaciones.

## Componentes Principales

### Playbooks

| Playbook | Descripción | Uso Principal |
|----------|-------------|---------------|
| `webhook.yml` | Sistema de notificaciones | Envío de alertas a Google Chat/Teams |
| `file_sync.yml` | Sincronización de archivos | Mantener consistencia entre servidores |
| `service_restart.yml` | Gestión de servicios Windows | Reiniciar/detener/iniciar servicios |
| `task_scheduler.yml` | Tareas programadas activo/pasivo | Gestión de failover/failback |
| `test_connection.yml` | Pruebas de conectividad | Validación de red (ping/telnet) |

### Roles

| Rol | Funcionalidad | Plataforma |
|-----|---------------|------------|
| `cards` | Notificaciones con tarjetas visuales | Multiplataforma |
| `file_sync` | Sincronización diferencial de archivos | Windows |
| `service_restart` | Gestión completa de servicios | Windows |
| `task_scheduler` | Control activo/pasivo de tareas | Windows |
| `test_connection` | Validación de conectividad de red | Windows/Linux |

## Estructura del Proyecto

```
davi-aspl-biz-ctg-ansible-script/
├── ansible.cfg                  # Configuración de Ansible
├── inventory/
│   ├── hosts                   # Inventario principal
│   └── windows/                # Inventarios específicos Windows
├── roles/
│   ├── cards/                  # Rol de notificaciones
│   │   ├── defaults/
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── vars/
│   ├── file_sync/              # Rol de sincronización
│   │   ├── defaults/
│   │   └── tasks/
│   ├── service_restart/        # Rol de servicios
│   │   ├── defaults/
│   │   └── tasks/
│   ├── task_scheduler/         # Rol de tareas programadas
│   │   ├── defaults/
│   │   └── tasks/
│   └── test_connection/        # Rol de conectividad
│       ├── defaults/
│       ├── tasks/
│       └── templates/
├── webhook.yml                 # Playbook de notificaciones
├── file_sync.yml              # Playbook de sincronización
├── service_restart.yml        # Playbook de servicios
├── task_scheduler.yml         # Playbook de tareas
├── test_connection.yml        # Playbook de conectividad
└── README.md                  # Esta documentación
```

## Requisitos Generales

### Software Base
- **Ansible**: >= 2.9
- **Python**: >= 3.6
- **ansible-navigator**: Para ejecución en contenedores (opcional)

### Colecciones Ansible Requeridas

```bash
ansible-galaxy collection install ansible.windows
ansible-galaxy collection install community.windows
```

### Configuración de Sistemas Objetivo

#### Windows
- Windows Server 2012 R2 o superior
- PowerShell 5.0 o superior
- WinRM configurado y habilitado
- Puertos: 5985 (HTTP) o 5986 (HTTPS)

#### Linux
- RedHat/CentOS 7+ o Debian/Ubuntu 18.04+
- SSH habilitado
- Python instalado

## Instalación

1. **Clonar el repositorio:**

```bash
git clone <repository_url>
cd davi-aspl-biz-ctg-ansible-script
```

2. **Instalar dependencias:**

```bash
ansible-galaxy collection install -r requirements.yml
```

3. **Configurar el inventario:**

```bash
cp inventory/hosts.example inventory/hosts
# Editar inventory/hosts con los servidores reales
```

4. **Configurar credenciales:**

```bash
ansible-vault create group_vars/windows_servers/vault.yml
# Agregar credenciales Windows
```

## Configuración del Inventario

### Ejemplo de inventario (`inventory/hosts`):

```ini
[windows_servers]
SADGVBPMAPP1 ansible_host=10.0.1.10 server_role=production app_instance=primary
SADGVBPMAPP2 ansible_host=10.0.1.11 server_role=production app_instance=secondary
CCMVBPMAPP ansible_host=10.0.2.10 server_role=contingency app_instance=primary

[linux_servers]
LBDGVANSIBLE1 ansible_host=10.0.3.10 server_role=control

[bizagi_task_servers]
TASKPROD01 ansible_host=10.0.1.50 server_role=produccion
TASKCONT01 ansible_host=10.0.2.50 server_role=contingencia

[windows_produccion]
SADGVBPMAPP1
SADGVBPMAPP2

[windows_contingencia]
CCMVBPMAPP

[windows_servers:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5985

[linux_servers:vars]
ansible_connection=ssh
ansible_user=ansiblecoe
```

## Guía de Uso por Componente

### 1. Notificaciones (webhook.yml)

Envía notificaciones estructuradas a plataformas de mensajería:

```bash
# Notificación simple
ansible-playbook webhook.yml \
  -e "input_webhook_url=https://chat.googleapis.com/v1/spaces/XXX/messages?key=YYY" \
  -e "card_title='Despliegue Completado'" \
  -e "card_subtitle='Ambiente Producción'"

# Con botones y URLs
ansible-playbook webhook.yml \
  -e "input_webhook_url=https://..." \
  -e "show_buttons=true" \
  -e "logs_url=https://jenkins.example.com/job/123" \
  -e "dashboard_url=https://grafana.example.com"
```

### 2. Sincronización de Archivos (file_sync.yml)

Sincroniza archivos entre servidores con análisis diferencial:

```bash
# Modo análisis (sin cambios)
ansible-playbook file_sync.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "target_server=CCMVBPMAPP" \
  -e "sync_path='D:\\BizagiProjects\\Config'" \
  -e "sync_patterns=['*.xml','*.config']" \
  -e "enable_sync=false"

# Modo sincronización completa
ansible-playbook file_sync.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "target_server=CCMVBPMAPP" \
  -e "sync_path='D:\\BizagiProjects\\Config'" \
  -e "sync_patterns=['*.xml','*.config']" \
  -e "enable_sync=true"
```

### 3. Gestión de Servicios (service_restart.yml)

Controla servicios Windows con manejo de dependencias:

```bash
# Reiniciar servicios
ansible-playbook service_restart.yml \
  -e "target_server=SADGVBPMAPP1" \
  -e "service_names=['W3SVC','WAS','IISADMIN']"

# Detener servicios específicos
ansible-playbook service_restart.yml \
  -e "target_server=SADGVBPMAPP1" \
  -e "service_names=['MSSQLSERVER','SQLSERVERAGENT']" \
  -e "service_restart_action=stopped"
```

### 4. Gestión de Tareas Programadas (task_scheduler.yml)

Gestiona configuración activo/pasivo para alta disponibilidad:

```bash
# Activar producción
ansible-playbook task_scheduler.yml \
  -e "active_environment=produccion" \
  -e "task_name='Backup Nocturno'"

# Failover a contingencia
ansible-playbook task_scheduler.yml \
  -e "active_environment=contingencia" \
  -e "task_name='Backup Nocturno'"
```

### 5. Pruebas de Conectividad (test_connection.yml)

Valida conectividad de red desde servidores específicos:

```bash
# Test ping
ansible-playbook test_connection.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "test_connection_check_type=ping" \
  -e "task_targets_input='192.168.1.100
192.168.1.101
192.168.1.102'"

# Test telnet a puertos
ansible-playbook test_connection.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "test_connection_check_type=telnet" \
  -e "task_targets_input='192.168.1.100'" \
  -e "task_ports_input=5985"

# Test combinado
ansible-playbook test_connection.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "test_connection_check_type=ambos" \
  -e "task_targets_input='192.168.1.100'" \
  -e "task_ports_input=5985"
```

## Casos de Uso Integrados

### Escenario: Mantenimiento Programado

```bash
#!/bin/bash
# maintenance.sh - Script de mantenimiento completo

# 1. Notificar inicio
ansible-playbook webhook.yml \
  -e "input_webhook_url=$WEBHOOK_URL" \
  -e "card_title='Mantenimiento Iniciado'" \
  -e "card_subtitle='$(date)'"

# 2. Verificar conectividad pre-mantenimiento
ansible-playbook test_connection.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "test_connection_check_type=ambos" \
  -e "task_targets_input='10.0.2.10'" \
  -e "task_ports_input=5985"

# 3. Sincronizar configuraciones
ansible-playbook file_sync.yml \
  -e "source_server=SADGVBPMAPP1" \
  -e "target_server=CCMVBPMAPP" \
  -e "sync_path='D:\\BizagiConfig'" \
  -e "sync_patterns=['*.xml']" \
  -e "enable_sync=true"

# 4. Activar contingencia
ansible-playbook task_scheduler.yml \
  -e "active_environment=contingencia" \
  -e "task_name='ProcessingJob'"

# 5. Reiniciar servicios en producción
ansible-playbook service_restart.yml \
  -e "target_server=SADGVBPMAPP1" \
  -e "service_names=['BizagiService']"

# 6. Notificar finalización
ansible-playbook webhook.yml \
  -e "input_webhook_url=$WEBHOOK_URL" \
  -e "card_title='Mantenimiento Completado'" \
  -e "deployment_status=success"
```

### Escenario: Failover de Emergencia

```bash
#!/bin/bash
# emergency_failover.sh

# 1. Cambiar todas las tareas a contingencia
TASKS=("Backup" "Sync" "Report" "Cleanup")
for task in "${TASKS[@]}"; do
  ansible-playbook task_scheduler.yml \
    -e "active_environment=contingencia" \
    -e "task_name='$task'"
done

# 2. Notificar del cambio
ansible-playbook webhook.yml \
  -e "input_webhook_url=$WEBHOOK_URL" \
  -e "card_title='FAILOVER EJECUTADO'" \
  -e "card_subtitle='Contingencia Activa'"
```

## Variables Globales Importantes

### Variables de Inventario

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `server_role` | Rol del servidor en la arquitectura | `production`, `contingency` |
| `app_instance` | Instancia de aplicación | `primary`, `secondary` |
| `ansible_host` | IP del servidor | `10.0.1.10` |

### Variables de Ejecución Comunes

| Variable | Playbook | Descripción |
|----------|----------|-------------|
| `source_server` | file_sync, test_connection | Servidor origen |
| `target_server` | file_sync, service_restart | Servidor destino |
| `active_environment` | task_scheduler | Ambiente activo |
| `input_webhook_url` | webhook | URL del webhook |

## Integración con AWX/Tower

### Job Templates Recomendados

1. **File Sync Analysis**
   - Playbook: `file_sync.yml`
   - Extra vars: Solicitar en survey
   - Schedule: Diario

2. **Service Health Check**
   - Playbook: `service_restart.yml`
   - Extra vars: Solo verificación
   - Schedule: Cada hora

3. **Connectivity Validation**
   - Playbook: `test_connection.yml`
   - Extra vars: Targets predefinidos
   - Schedule: Cada 30 minutos

4. **Scheduled Task Management**
   - Playbook: `task_scheduler.yml`
   - Extra vars: Por ambiente
   - Schedule: Manual

### Workflows AWX

```yaml
name: Complete Maintenance Workflow
nodes:
  - name: Pre-check connectivity
    job_template: Connectivity Validation
  - name: Sync files
    job_template: File Sync Analysis
    success_nodes:
      - name: Execute sync
        job_template: File Sync Execute
  - name: Switch to contingency
    job_template: Task Scheduler Contingency
  - name: Restart services
    job_template: Service Restart
  - name: Send notification
    job_template: Webhook Notification
```

## Monitoreo y Métricas

### Métricas Registradas

Todos los playbooks registran métricas en AWX mediante `set_stats`:

- **file_sync**: Archivos sincronizados, tasa de éxito
- **service_restart**: Servicios procesados, estado final
- **task_scheduler**: Cambios aplicados, ambiente activo
- **test_connection**: Pruebas exitosas/fallidas
- **webhook**: Notificaciones enviadas

### Dashboard Recomendado

Crear dashboard en Grafana con:
- Panel de estado de servicios
- Gráfico de sincronización de archivos
- Estado activo/pasivo de tareas
- Matriz de conectividad
- Log de notificaciones

## Mejores Prácticas

### 1. Orden de Ejecución

Para cambios mayores, seguir este orden:
1. Test de conectividad
2. Sincronización de archivos (análisis)
3. Cambio de tareas programadas
4. Sincronización de archivos (ejecución)
5. Reinicio de servicios
6. Notificación de completado

### 2. Validación Pre-Cambios

Siempre ejecutar en modo check primero:

```bash
ansible-playbook <playbook>.yml --check -e "variables..."
```

### 3. Documentación de Cambios

Usar IDs descriptivos:

```bash
-e "custom_task_id=CHANGE_$(date +%Y%m%d_%H%M%S)"
-e "custom_description='Ticket #1234 - Descripción del cambio'"
```

### 4. Gestión de Credenciales

```bash
# Crear vault para credenciales
ansible-vault create group_vars/all/vault.yml

# Contenido sugerido:
vault_ansible_user: admin_user
vault_ansible_password: secure_password
vault_webhook_url: https://chat.googleapis.com/...
```

### 5. Logs y Auditoría

Habilitar logging detallado:

```bash
export ANSIBLE_LOG_PATH=/var/log/ansible/$(date +%Y%m%d).log
ansible-playbook <playbook>.yml -v
```

## Troubleshooting

### Problemas Comunes

| Problema | Causa | Solución |
|----------|-------|----------|
| WinRM timeout | Firewall o configuración | Verificar puerto 5985/5986 |
| Servidor unreachable | Credenciales o red | Test con `ansible -m win_ping` |
| Sincronización falla | Permisos insuficientes | Verificar cuenta de servicio |
| Servicios no reinician | Dependencias | Habilitar force_dependent |
| Tareas no cambian | No existe la tarea | Verificar nombre exacto |

### Comandos de Diagnóstico

```bash
# Test de conectividad básica
ansible all -m ping

# Verificar Windows
ansible windows_servers -m win_ping

# Listar servicios
ansible SERVIDOR -m win_shell -a "Get-Service | Select Name,Status"

# Verificar tareas programadas
ansible SERVIDOR -m win_shell -a "Get-ScheduledTask | Select TaskName,State"
```

## Seguridad

### Recomendaciones

1. **Credenciales**: Siempre usar Ansible Vault
2. **Conexiones**: Preferir HTTPS/5986 sobre HTTP/5985
3. **Permisos**: Principio de menor privilegio
4. **Auditoría**: Habilitar logs en todos los playbooks
5. **Validación**: Siempre validar entrada de usuario

### Configuración Segura

```yaml
# ansible.cfg
[defaults]
host_key_checking = True
vault_password_file = ~/.vault_pass
no_log = False
log_path = /var/log/ansible/ansible.log

[privilege_escalation]
become = True
become_method = runas
become_user = Administrator
```

## Mantenimiento del Proyecto

### Actualización de Roles

```bash
# Actualizar un rol específico
cd roles/role_name
git pull origin main

# Test del rol actualizado
ansible-playbook test_playbook.yml --tags role_name
```

### Backup de Configuración

```bash
# Backup del proyecto
tar -czf backup_$(date +%Y%m%d).tar.gz \
  --exclude='*.log' \
  --exclude='*.retry' \
  davi-aspl-biz-ctg-ansible-script/
```

## Contribuciones

### Estructura para Nuevos Roles

```bash
ansible-galaxy init roles/nuevo_rol
# Seguir la estructura existente
# Documentar en README.md del rol
# Agregar tests en tests/
```

### Estándares de Código

- Usar YAML válido con indentación de 2 espacios
- Nombrar tareas descriptivamente
- Incluir tags relevantes
- Documentar variables en defaults/main.yml
- Crear README.md para cada rol


### Recursos

- [Documentación Ansible](https://docs.ansible.com)
- [Windows Modules](https://docs.ansible.com/ansible/latest/collections/ansible/windows/)
- [AWX Documentation](https://ansible.readthedocs.io/projects/awx/)

## Licencia

Proyecto interno - Uso exclusivo corporativo

---

**Autor**: cgarzont (NTTDATA)  
**Versión**: 1.0.0  
**Compatibilidad**: Ansible 2.9+, Windows Server 2016+  
**Última actualización**: Agosto 2025