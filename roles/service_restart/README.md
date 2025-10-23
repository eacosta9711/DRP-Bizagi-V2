# Ansible Windows Service Restart Role

Sistema automatizado para gestión y reinicio de servicios Windows con validación exhaustiva, manejo de dependencias y generación de métricas detalladas.

## Descripción

Este rol de Ansible proporciona capacidades robustas para reiniciar, detener o iniciar servicios Windows de manera controlada. Incluye validación de existencia de servicios, manejo automático de dependencias, reportes detallados del estado antes/después y métricas para integración con AWX/Tower.

## Características Principales

- **Validación Multi-nivel**: Verifica existencia de servicios antes de procesar
- **Manejo de Dependencias**: Control automático de servicios dependientes
- **Estados Múltiples**: Soporte para restart, stop y start
- **Reportes Detallados**: Estado inicial, proceso de reinicio y estado final
- **Manejo de Errores**: Continúa procesando aunque algunos servicios fallen
- **Métricas AWX**: Registro completo de estadísticas para dashboards
- **Procesamiento en Lote**: Manejo eficiente de múltiples servicios
- **Detección Automática**: Identifica servicios faltantes vs existentes

## Estructura del Proyecto

```
.
├── service_restart.yml                  # Playbook principal
└── roles/
    └── service_restart/
        ├── defaults/
        │   └── main.yml                 # Variables por defecto
        └── tasks/
            ├── main.yml                 # Orquestador principal
            ├── validate_prerequisites.yml   # Validaciones previas
            ├── get_initial_status.yml      # Estado inicial
            ├── restart_services.yml        # Reinicio de servicios
            └── generate_stats.yml          # Generación de métricas
```

## Requisitos

### Sistema
- **Ansible**: >= 2.9
- **Python**: >= 3.6
- **Sistema Operativo**: Windows Server 2012 R2 o superior

### Módulos Ansible Requeridos
- `ansible.windows.win_ping`
- `ansible.windows.win_service`
- `ansible.builtin.assert`
- `ansible.builtin.set_fact`
- `ansible.builtin.debug`
- `ansible.builtin.include_tasks`
- `ansible.builtin.set_stats`
- `ansible.builtin.pause`

### Conectividad
- WinRM configurado y funcional
- Puerto 5985 (HTTP) o 5986 (HTTPS) abierto
- Credenciales con permisos administrativos locales
- Permisos para gestionar servicios Windows

## Instalación

1. Instalar la colección Windows de Ansible:

```bash
ansible-galaxy collection install ansible.windows
```

2. Copiar el rol en tu directorio de roles:

```bash
cp -r roles/service_restart /path/to/your/ansible/roles/
```

3. Configurar el inventario con los servidores Windows:

```ini
[windows_servers]
WEBSERVER01 ansible_host=10.0.1.10 server_role=web app_instance=primary
APPSERVER01 ansible_host=10.0.1.20 server_role=app app_instance=primary
DBSERVER01 ansible_host=10.0.1.30 server_role=database app_instance=primary

[windows_servers:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5985
```

## Uso

### Reinicio de un Servicio Único

```bash
ansible-playbook service_restart.yml \
  -e "target_server=WEBSERVER01" \
  -e "service_names='W3SVC'"
```

### Reinicio de Múltiples Servicios

```bash
ansible-playbook service_restart.yml \
  -e "target_server=APPSERVER01" \
  -e "service_names=['Spooler','W3SVC','MSSQLSERVER']"
```

### Detener Servicios

```bash
ansible-playbook service_restart.yml \
  -e "target_server=DBSERVER01" \
  -e "service_names=['MSSQLSERVER','SQLSERVERAGENT']" \
  -e "service_restart_action=stopped"
```

### Iniciar Servicios

```bash
ansible-playbook service_restart.yml \
  -e "target_server=WEBSERVER01" \
  -e "service_names=['W3SVC','WAS']" \
  -e "service_restart_action=started"
```

## Variables

### Variables Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `target_server` | Servidor donde ejecutar (debe existir en inventario) | `WEBSERVER01` |
| `service_names` | Servicio(s) a procesar (string o lista) | `['W3SVC','Spooler']` |

### Variables Opcionales del Playbook

| Variable | Descripción | Valor por Defecto |
|----------|-------------|-------------------|
| `custom_task_id` | ID personalizado para tracking | `service_restart` |
| `custom_description` | Descripción de la tarea | `Reinicio de Servicios Windows` |

### Variables del Rol

| Variable | Descripción | Valor por Defecto |
|----------|-------------|-------------------|
| `service_restart_action` | Acción a ejecutar | `restarted` |
| `service_restart_force_dependent_services` | Forzar servicios dependientes | `true` |
| `service_restart_start_mode` | Modo de inicio del servicio | `auto` |
| `service_restart_validate_final_state` | Validar estado final | `true` |
| `service_restart_show_detailed_output` | Mostrar salida detallada | `true` |

### Valores para service_restart_action

- `restarted`: Reinicia el servicio (stop + start)
- `stopped`: Detiene el servicio
- `started`: Inicia el servicio

### Valores para service_restart_start_mode

- `auto`: Inicio automático
- `manual`: Inicio manual
- `disabled`: Deshabilitado

## Flujo de Ejecución

1. **Validación de Parámetros** (localhost)
   - Verifica servidor definido
   - Valida existencia en inventario
   - Procesa lista de servicios

2. **Validación de Prerequisitos** (target server)
   - Confirma sistema Windows
   - Verifica conectividad WinRM
   - Valida existencia de cada servicio
   - Identifica servicios faltantes

3. **Captura de Estado Inicial**
   - Registra estado actual de cada servicio
   - Documenta modo de inicio
   - Genera estadísticas iniciales

4. **Ejecución del Reinicio**
   - Procesa cada servicio existente
   - Maneja dependencias si está habilitado
   - Registra errores sin detener el proceso
   - Pausa de estabilización post-reinicio

5. **Validación de Estado Final**
   - Verifica estado actual de servicios
   - Compara con estado inicial
   - Identifica cambios exitosos/fallidos

6. **Generación de Métricas**
   - Calcula tasa de éxito
   - Registra estadísticas en AWX
   - Genera reportes visuales

## Reportes y Métricas

### Reporte Visual de Reinicio

```
╔═══════════════════════════════════════════════════════════════
║              REPORTE DE REINICIO DE SERVICIOS                 
╠═══════════════════════════════════════════════════════════════
║ Servidor: WEBSERVER01
║ Task ID: iis_restart_prod
║ Fecha: 2024-01-15 14:30:00
╠═══════════════════════════════════════════════════════════════
║ RESULTADOS POR SERVICIO:
║  W3SVC: EXITOSO (running → running)
║  WAS: EXITOSO (running → running)
║  IISADMIN: EXITOSO (running → running)
╠═══════════════════════════════════════════════════════════════
║ RESUMEN:
║ Total servicios: 3
║ Exitosos: 3
║ Fallidos: 0
║ Corriendo: 3
╚═══════════════════════════════════════════════════════════════
```

### Métricas AWX

El rol registra las siguientes métricas:

- `task_{task_id}`: Información completa de la tarea
  - `task_info`: Detalles completos
  - `host`: Servidor procesado
  - `status`: success/partial_success/failed

- `service_restart_summary`: Resumen ejecutivo
  - `total_requested`: Servicios solicitados
  - `total_processed`: Servicios existentes procesados
  - `total_successful`: Reiniciados exitosamente
  - `total_failed`: Fallos en reinicio
  - `total_running`: Servicios corriendo post-reinicio
  - `success_rate`: Porcentaje de éxito
  - `services_missing`: Lista de servicios no encontrados

## Manejo de Errores

### Servicios No Existentes

El rol identifica y reporta servicios que no existen:

```
════════════════════════════════════════════
ADVERTENCIA - SERVICIOS FALTANTES
════════════════════════════════════════════
Servicios no encontrados: InvalidService
Solo se procesarán los servicios existentes
════════════════════════════════════════════
```

### Fallos en Reinicio

Si un servicio falla al reiniciar, el rol:
1. Continúa con los demás servicios
2. Registra el error específico
3. Reporta en el resumen final
4. Ajusta el status a `partial_success`

### Validación de Estado Final

El rol siempre verifica el estado final, independientemente del resultado del comando de reinicio.

## Servicios Windows Comunes

### Servicios de Sistema

| Nombre del Servicio | Descripción |
|-------------------|-------------|
| `Spooler` | Servicio de cola de impresión |
| `Themes` | Servicio de temas de Windows |
| `EventLog` | Registro de eventos de Windows |
| `WinRM` | Administración remota de Windows |

### Servicios IIS

| Nombre del Servicio | Descripción |
|-------------------|-------------|
| `W3SVC` | Servicio de publicación World Wide Web |
| `WAS` | Servicio de activación de procesos Windows |
| `IISADMIN` | Servicio de administración de IIS |

### Servicios SQL Server

| Nombre del Servicio | Descripción |
|-------------------|-------------|
| `MSSQLSERVER` | Motor de base de datos SQL Server |
| `SQLSERVERAGENT` | Agente SQL Server |
| `MSSQLFDLauncher` | Demonio de filtro de texto completo |

## Troubleshooting

### Error: "Parámetro requerido faltante"

**Causa**: No se definieron `target_server` o `service_names`

**Solución**:
```bash
ansible-playbook service_restart.yml \
  -e "target_server=SERVIDOR" \
  -e "service_names='NombreServicio'"
```

### Error: "El servidor no existe en el inventario"

**Causa**: El servidor especificado no está en el inventario

**Solución**: Agregar el servidor al inventario o verificar el nombre

### Error: WinRM no responde

**Causa**: Problemas de conectividad o configuración WinRM

**Diagnóstico**:
```bash
# Test de conectividad
ansible SERVIDOR -m win_ping

# Verificar WinRM en el servidor
winrm quickconfig
winrm enumerate winrm/config/listener
```

### Servicios no se reinician correctamente

**Causa**: Dependencias no manejadas o permisos insuficientes

**Solución**:
1. Verificar dependencias: `sc qc NombreServicio`
2. Habilitar manejo de dependencias: `-e "service_restart_force_dependent_services=true"`
3. Verificar permisos del usuario

## Mejores Prácticas

1. **Siempre validar servicios primero**
   ```bash
   # Verificar en el servidor
   Get-Service | Where-Object {$_.Name -like "*pattern*"}
   ```

2. **Usar nombres exactos de servicios**
   - Usar el nombre del servicio, no el display name
   - Ejemplo: Usar `W3SVC` no "World Wide Web Publishing Service"

3. **Procesar servicios relacionados juntos**
   ```bash
   -e "service_names=['W3SVC','WAS','IISADMIN']"
   ```

4. **Documentar con task IDs descriptivos**
   ```bash
   -e "custom_task_id=monthly_maint_20240115"
   ```

5. **Considerar el orden de reinicio**
   - Para SQL: Detener Agent antes que el Motor
   - Para IIS: Considerar el orden de dependencias

## Seguridad

- **Credenciales**: Usar Ansible Vault para credenciales sensibles
- **Permisos**: Cuenta de servicio con permisos mínimos necesarios
- **Auditoría**: Todos los cambios son registrados con timestamp
- **Validación**: No ejecuta acciones sin validar existencia primero

## Limitaciones

- Solo funciona en sistemas Windows
- Requiere WinRM configurado y funcional
- No maneja servicios en contenedores Docker/Windows
- Tiempo de estabilización fijo de 3 segundos
- No soporta reinicio condicional basado en métricas

---

**Autor**: cgarzont (NTTDATA)  
**Versión**: 1.0.0  
**Compatibilidad**: Ansible 2.9+, Windows Server 2016+  
**Última actualización**: Agosto 2025