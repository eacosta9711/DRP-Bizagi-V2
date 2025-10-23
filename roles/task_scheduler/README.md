# Ansible Task Scheduler Management Role

Sistema automatizado para gestión de tareas programadas Windows en configuración activo/pasivo, diseñado para mantener alta disponibilidad en ambientes de producción y contingencia.

## Descripción

Este rol de Ansible gestiona tareas programadas de Windows en una arquitectura activo/pasivo entre servidores de producción y contingencia. Garantiza que las tareas críticas solo estén habilitadas en el ambiente activo, evitando ejecuciones duplicadas y conflictos entre ambientes.

## Características Principales

- **Gestión Activo/Pasivo**: Control automático basado en ambiente activo
- **Validación Exhaustiva**: Verifica existencia de tareas antes de modificar
- **Cambios Inteligentes**: Solo aplica cambios cuando es necesario
- **Reportes Detallados**: Estado antes/después con información completa
- **Failover Support**: Facilita cambio entre producción y contingencia
- **Métricas AWX**: Registro completo para dashboards y auditoría
- **Idempotencia**: Ejecuciones múltiples sin efectos adversos
- **Procesamiento Serial**: Gestión controlada servidor por servidor

## Arquitectura del Sistema

```
Configuración Activo/Pasivo:
┌─────────────────────┐     ┌─────────────────────┐
│  SERVIDOR PROD      │     │  SERVIDOR CONT      │
│  (Producción)       │     │  (Contingencia)     │
├─────────────────────┤     ├─────────────────────┤
│ Tarea: HABILITADA   │     │ Tarea: DESHABILITADA│
│ (si activo=prod)    │     │ (si activo=prod)    │
├─────────────────────┤     ├─────────────────────┤
│ Tarea: DESHABILITADA│     │ Tarea: HABILITADA   │
│ (si activo=cont)    │     │ (si activo=cont)    │
└─────────────────────┘     └─────────────────────┘
```

## Estructura del Proyecto

```
.
├── task_scheduler.yml                   # Playbook principal
└── roles/
    └── task_scheduler/
        ├── defaults/
        │   └── main.yml                 # Variables por defecto
        └── tasks/
            ├── main.yml                 # Orquestador principal
            ├── validate_task.yml        # Validación de tarea
            ├── manage_task.yml          # Gestión de estado
            └── generate_report.yml      # Generación de reportes
```

## Requisitos

### Sistema
- **Ansible**: >= 2.9
- **Python**: >= 3.6
- **Sistema Operativo**: Windows Server 2012 R2 o superior
- **PowerShell**: >= 5.0

### Módulos Ansible Requeridos
- `community.windows.win_scheduled_task`
- `community.windows.win_scheduled_task_stat`
- `ansible.builtin.assert`
- `ansible.builtin.set_fact`
- `ansible.builtin.debug`
- `ansible.builtin.include_tasks`
- `ansible.builtin.set_stats`

### Configuración de Infraestructura
- Exactamente 2 servidores en grupo `bizagi_task_servers`
- Un servidor con `server_role: produccion`
- Un servidor con `server_role: contingencia`
- WinRM configurado en ambos servidores

## Instalación

1. Instalar la colección Windows de Ansible:

```bash
ansible-galaxy collection install community.windows
```

2. Copiar el rol en tu directorio de roles:

```bash
cp -r roles/task_scheduler /path/to/your/ansible/roles/
```

3. Configurar el inventario con la estructura requerida:

```ini
[bizagi_task_servers]
TASKPROD01 ansible_host=10.0.1.50 server_role=produccion
TASKCONT01 ansible_host=10.0.2.50 server_role=contingencia

[bizagi_task_servers:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5985
```

## Uso

### Activar Ambiente de Producción

Habilita la tarea en producción y la deshabilita en contingencia:

```bash
ansible-playbook task_scheduler.yml \
  -e "active_environment=produccion" \
  -e "task_name='Backup Nocturno'"
```

### Activar Ambiente de Contingencia

Habilita la tarea en contingencia y la deshabilita en producción:

```bash
ansible-playbook task_scheduler.yml \
  -e "active_environment=contingencia" \
  -e "task_name='Backup Nocturno'"
```

### Ejemplos Avanzados

#### Gestión de múltiples tareas en failover:

```bash
# Script para failover completo
for task in "Backup Nocturno" "Limpieza Logs" "Sincronización DB"; do
  ansible-playbook task_scheduler.yml \
    -e "active_environment=contingencia" \
    -e "task_name='$task'"
done
```

#### Verificación de estado sin cambios:

```bash
ansible-playbook task_scheduler.yml \
  -e "active_environment=produccion" \
  -e "task_name='Proceso ETL'" \
  --check
```

## Variables

### Variables Requeridas

| Variable | Descripción | Valores Válidos |
|----------|-------------|-----------------|
| `active_environment` | Define qué ambiente está activo | `produccion`, `contingencia` |
| `task_name` | Nombre exacto de la tarea programada | String (nombre de tarea Windows) |

### Variables del Rol

| Variable | Descripción | Valor por Defecto |
|----------|-------------|-------------------|
| `task_scheduler_should_enable` | Si la tarea debe estar habilitada | Calculado automáticamente |
| `task_scheduler_server_role` | Rol del servidor actual | Tomado del inventario |
| `task_scheduler_active_environment` | Ambiente actualmente activo | Pasado desde playbook |
| `task_scheduler_validate_script` | Validar script de la tarea | `true` |
| `task_scheduler_show_details` | Mostrar detalles extendidos | `true` |

### Variables de Inventario Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `server_role` | Rol del servidor en la arquitectura | `produccion` o `contingencia` |
| `ansible_host` | IP del servidor | `10.0.1.50` |

## Flujo de Ejecución

### Play 1: Validaciones (localhost)

1. **Validación de Parámetros**
   - Verifica `active_environment` válido
   - Confirma `task_name` definido
   - Valida grupo `bizagi_task_servers`

2. **Identificación de Servidores**
   - Detecta servidor de producción
   - Detecta servidor de contingencia
   - Muestra configuración deseada

### Play 2: Gestión (servidores objetivo)

1. **Validación de Tarea**
   - Obtiene información actual de la tarea
   - Verifica que la tarea existe
   - Registra estado actual (habilitada/deshabilitada)

2. **Gestión de Estado**
   - Determina si requiere cambio
   - Aplica cambio si es necesario
   - Verifica el cambio aplicado

3. **Generación de Reporte**
   - Crea estadísticas finales
   - Registra en AWX
   - Muestra reporte visual

### Play 3: Reporte Consolidado (localhost)

- Muestra resumen final de ambos servidores
- Confirma configuración activo/pasivo

## Lógica de Decisión

El rol determina automáticamente el estado deseado:

```
SI (server_role == 'produccion' Y active_environment == 'produccion')
    ENTONCES habilitar_tarea = TRUE
SI (server_role == 'contingencia' Y active_environment == 'contingencia')
    ENTONCES habilitar_tarea = TRUE
SINO
    habilitar_tarea = FALSE
```

## Reportes y Métricas

### Reporte Visual por Servidor

```
╔═══════════════════════════════════════════════════════════════
║         REPORTE FINAL - TASKPROD01
╠═══════════════════════════════════════════════════════════════
║ Servidor: TASKPROD01 (PRODUCCION)
║ Tarea: Backup Nocturno
║ Ambiente activo: PRODUCCION
║ Fecha: 2024-01-15 14:30:00
╠═══════════════════════════════════════════════════════════════
║ RESULTADO:
║ Estado final: HABILITADA
║ Cambio realizado: SÍ
║ Acción: HABILITADA
║ Estado: ÉXITO
╚═══════════════════════════════════════════════════════════════
```

### Métricas AWX

El rol registra las siguientes métricas:

Por servidor (`task_scheduler_{hostname}`):
- `server_info`: Información completa del servidor
  - `server`: Nombre del servidor
  - `server_role`: Rol (produccion/contingencia)
  - `task_name`: Nombre de la tarea
  - `active_environment`: Ambiente activo
  - `was_changed`: Si hubo cambio
  - `final_enabled`: Estado final
  - `action_taken`: Acción realizada
  - `success`: Resultado exitoso

Resumen global (`task_scheduler_summary`):
- `active_environment`: Ambiente configurado
- `total_servers`: Total de servidores procesados
- `timestamp`: Marca de tiempo ISO 8601

## Estados de Tareas Programadas

### Estados Posibles en Windows

| Estado | Descripción |
|--------|-------------|
| `Ready` | Lista para ejecutarse |
| `Running` | Ejecutándose actualmente |
| `Disabled` | Deshabilitada |
| `Queued` | En cola de ejecución |

### Error de conectividad WinRM

**Causa**: Configuración WinRM incorrecta

**Solución**:
```powershell
# En el servidor Windows
winrm quickconfig
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'
```

## Mejores Prácticas

1. **Verificar antes de cambiar**
   ```bash
   # Usar --check para simular
   ansible-playbook task_scheduler.yml ... --check
   ```

2. **Documentar cambios de ambiente**
   - Registrar fecha/hora del cambio
   - Documentar razón del failover
   - Notificar equipos afectados

3. **Validar post-cambio**
   ```powershell
   # En ambos servidores
   Get-ScheduledTask -TaskName "NombreTarea" | Select TaskName, State
   ```

4. **Mantener sincronización de tareas**
   - Asegurar que las tareas existan en ambos servidores
   - Mantener configuraciones idénticas
   - Solo el estado habilitado/deshabilitado debe diferir

5. **Automatizar failovers completos**
   - Crear scripts para todas las tareas críticas
   - Incluir validaciones pre y post cambio
   - Implementar notificaciones


## Seguridad

- **Credenciales**: Usar Ansible Vault para credenciales sensibles
- **Auditoría**: Todos los cambios son registrados con timestamp
- **Permisos**: Cuenta con permisos mínimos necesarios para gestionar tareas
- **Validación**: No ejecuta cambios sin validar existencia primero
- **Idempotencia**: Múltiples ejecuciones no causan problemas

## Limitaciones

- Solo funciona con exactamente 2 servidores
- Requiere que las tareas existan previamente
- No crea ni elimina tareas, solo gestiona estado
- No modifica la configuración de la tarea
- Procesamiento serial (un servidor a la vez)

---

**Autor**: cgarzont (NTTDATA)  
**Versión**: 1.0.0  
**Compatibilidad**: Ansible 2.9+, Windows Server 2016+  
**Última actualización**: Agosto 2025