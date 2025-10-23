# Sistema de Sincronización de Archivos - file_sync

## Descripción

Sistema modular de sincronización de archivos para Windows que permite comparar y sincronizar directorios entre servidores usando robocopy. Diseñado específicamente para casos de uso de producción y contingencia con validaciones robustas y reportes detallados.

## Arquitectura

```
file_sync_system/
├── file_sync.yml                         # Playbook principal
├── roles/file_sync/                      # Rol de sincronización
│   ├── defaults/main.yml                 # Configuración por defecto
│   └── tasks/
│       ├── main.yml                      # Orquestador principal
│       ├── validate_prerequisites.yml    # Validaciones de conectividad
│       ├── collect_source_files.yml      # Recolección archivos origen
│       ├── collect_target_files.yml      # Recolección archivos destino
│       ├── compare_files.yml             # Comparación y análisis
│       ├── sync_files.yml                # Sincronización con robocopy
│       └── generate_stats.yml            # Estadísticas para AWX
└── inventory/                            # Inventarios por ambiente
    └── windows/hosts                     # Servidores Windows
```

## Características

### Funcionalidades Principales

- **Análisis de archivos**: Comparación detallada entre origen y destino
- **Sincronización inteligente**: Solo copia archivos faltantes o desactualizados
- **Filtrado por patrones**: Soporte para múltiples extensiones de archivo
- **Validaciones robustas**: Conectividad, rutas y permisos
- **Reportes visuales**: Salidas estructuradas con bordes Unicode
- **Integración AWX**: Métricas y estadísticas automáticas

### Tipos de Comparación

- **Archivos faltantes**: Existen en origen pero no en destino
- **Archivos desactualizados**: Versión en origen es más reciente
- **Archivos idénticos**: Mismo checksum en ambos servidores

### Modos de Operación

- **Solo análisis**: Compara archivos sin sincronizar
- **Análisis + sincronización**: Compara y sincroniza archivos

## Instalación y Configuración

### Pre-requisitos

```bash
# Ansible y colecciones necesarias
pip install ansible
ansible-galaxy collection install ansible.windows
ansible-galaxy collection install community.windows
```

### Configuración de Inventario

```ini
# inventory/windows/hosts
[windows_produccion_app1]
rdpinfralab ansible_host=90.4.0.82 server_role=produccion app_instance=app1

[windows_contingencia]
lbdgvrdpdatos ansible_host=90.4.23.108 server_role=contingencia app_instance=backup

[windows_servers:children]
windows_produccion_app1
windows_contingencia

[windows_servers:vars]
ansible_user=DOMAIN\username
ansible_password="password"
ansible_connection=winrm
ansible_winrm_scheme=https
ansible_winrm_transport=credssp
ansible_winrm_port=5986
ansible_winrm_server_cert_validation=ignore
```

## Uso

### Línea de Comandos

#### Ejemplo Básico - Solo Análisis

```bash
ansible-playbook file_sync.yml -i inventory/windows/hosts \
  -e '{"source_server": "rdpinfralab", 
       "target_server": "lbdgvrdpdatos", 
       "sync_path": "C:\\Temp\\Documentos", 
       "sync_patterns": ["*.txt", "*.ps1"], 
       "enable_sync": false}'
```

#### Ejemplo Avanzado - Análisis y Sincronización

```bash
ansible-playbook file_sync.yml -i inventory/windows/hosts \
  -e '{"source_server": "rdpinfralab", 
       "target_server": "lbdgvrdpdatos", 
       "custom_sync_task_id": "sync_reportes", 
       "custom_sync_description": "Sincronización reportes diarios", 
       "sync_path": "D:\\Reportes\\Diarios", 
       "sync_patterns": ["*.bat", "*.ps1", "*.xml"], 
       "enable_sync": true}'
```

### Configuración AWX Job Template

#### Variables de Survey

| Variable | Tipo | Descripción | Ejemplo |
|----------|------|-------------|---------|
| `source_server` | Choice | Servidor origen | `rdpinfralab` |
| `target_server` | Choice | Servidor destino | `lbdgvrdpdatos` |
| `sync_path` | Text | Ruta a sincronizar | `C:\Temp\Prueba Copia` |
| `sync_patterns` | Textarea | Patrones (JSON) | `["*.txt", "*.ps1"]` |
| `enable_sync` | Boolean | Ejecutar sincronización | `true`/`false` |
| `custom_sync_task_id` | Text | ID de tarea | `sync_0.3` |
| `custom_sync_description` | Text | Descripción | `Reportes diarios` |

#### Configuración del Job Template

```yaml
Name: File Synchronization
Inventory: Windows Servers
Project: Automation Catalog
Playbook: file_sync.yml
Survey Enabled: true
Prompt on Launch: Variables
```

## Variables y Parámetros

### Variables Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `source_server` | Servidor origen en inventario | `rdpinfralab` |
| `target_server` | Servidor destino en inventario | `lbdgvrdpdatos` |
| `sync_path` | Ruta Windows válida | `C:\Temp\Documentos` |
| `sync_patterns` | Lista de patrones de archivo | `["*.txt", "*.ps1"]` |

### Variables Opcionales

| Variable | Default | Descripción |
|----------|---------|-------------|
| `enable_sync` | `false` | Ejecutar sincronización real |
| `custom_sync_task_id` | `sync_0.3` | ID único de la tarea |
| `custom_sync_description` | `Bizagi - Sincronización` | Descripción personalizada |

### Variables del Rol

| Variable | Default | Descripción |
|----------|---------|-------------|
| `file_sync_robocopy_flags` | `/E /XO /R:3 /W:10 /MT:4` | Flags de robocopy |
| `file_sync_timeout` | `300` | Timeout en segundos |
| `file_sync_retries` | `3` | Número de reintentos |

## Salidas y Reportes

### Reporte de Comparación

```
╔═══════════════════════════════════════════════════════════════
║            REPORTE DE COMPARACIÓN DE ARCHIVOS                 
╠═══════════════════════════════════════════════════════════════
║ Task: Bizagi - Validar reportes diarios
║ ID: 0.3
║ Fecha: 2025-08-14 17:24:50
╠═══════════════════════════════════════════════════════════════
║ RESUMEN DE RESULTADOS:
║ Total archivos en producción:    4
║ Total archivos en contingencia:  3
║ Archivos faltantes:              1
║ Archivos desactualizados:        1
║ Archivos idénticos:              2
╠═══════════════════════════════════════════════════════════════
║ ESTADO: WARNING
╚═══════════════════════════════════════════════════════════════
```

### Reporte de Sincronización

```
╔═══════════════════════════════════════════════════════════════
║           REPORTE DE SINCRONIZACIÓN DE ARCHIVOS               
╠═══════════════════════════════════════════════════════════════
║ Task: Bizagi - Validar reportes diarios
║ ID: 0.3
║ Fecha: 2025-08-14 17:24:50
╠═══════════════════════════════════════════════════════════════
║ SINCRONIZACIÓN:
║ Origen:  rdpinfralab (90.4.0.82)
║ Destino: lbdgvrdpdatos (90.4.23.108)
║ Ruta:    C:\Temp\Prueba Copia
╠═══════════════════════════════════════════════════════════════
║ RESUMEN DE RESULTADOS:
║ Total archivos en origen:         4
║ Total archivos en destino:        3
║ Archivos faltantes copiados:      1
║ Archivos desactualizados:         1
║ Total archivos sincronizados:     2
╠═══════════════════════════════════════════════════════════════
║ RESULTADO: EXITOSO
║ Mensaje: Files copied successfully!
╚═══════════════════════════════════════════════════════════════
```

### Estadísticas AWX

```json
{
  "task_sync_0_3": {
    "task_info": {
      "task_id": "sync_0.3",
      "description": "Bizagi - Validar reportes diarios",
      "source_server": "rdpinfralab",
      "target_server": "lbdgvrdpdatos",
      "sync_path": "C:\\Temp\\Prueba Copia",
      "metrics": {
        "total_source": "4",
        "total_target": "3",
        "missing_count": "1",
        "outdated_count": "1",
        "files_to_sync": "2",
        "sync_required": true
      }
    },
    "host": "rdpinfralab",
    "status": "success"
  }
}
```

## Casos de Uso

### Caso 1: Validación de Reportes Diarios

```bash
ansible-playbook file_sync.yml -i inventory/windows/hosts \
  -e '{"source_server": "SADGVBPMAPP1", 
       "target_server": "CCMVBPMAPP", 
       "custom_sync_task_id": "0.3", 
       "custom_sync_description": "Bizagi - Validar reportes diarios", 
       "sync_path": "D:\\\\Mis Documentos", 
       "sync_patterns": ["*.bat", "*.ps1"], 
       "enable_sync": false}'
```

### Caso 2: Backup de Documentos

```bash
ansible-playbook file_sync.yml -i inventory/windows/hosts \
  -e '{"source_server": "CCMVBPMAPP", 
       "target_server": "SADGVBPMAPP2", 
       "custom_sync_task_id": "0.5", 
       "custom_sync_description": "Backup docsbanccapersona", 
       "sync_path": "D:\\\\docsbanccapersona", 
       "sync_patterns": ["*.*"], 
       "enable_sync": true}'
```

### Caso 3: Sincronización Scripts de Configuración

```bash
ansible-playbook file_sync.yml -i inventory/windows/hosts \
  -e '{"source_server": "rdpinfralab", 
       "target_server": "lbdgvrdpdatos", 
       "custom_sync_task_id": "config_scripts", 
       "custom_sync_description": "Scripts de configuración", 
       "sync_path": "C:\\\\Scripts\\\\Config", 
       "sync_patterns": ["*.ps1", "*.bat", "*.xml"], 
       "enable_sync": true}'
```

## Troubleshooting

### Errores Comunes

#### Error: "Servidor no existe en inventario"

```
ERROR: El servidor origen no existe en el inventario:
- Servidor buscado: "SERVIDOR_INEXISTENTE"
```

**Solución**: Verificar que el servidor esté definido en el inventario.

#### Error: "Ruta debe cumplir con formato Windows válido"

```
ERROR: La ruta debe cumplir con formato Windows válido:
- Debe iniciar con letra de unidad seguida de :\ (ej: C:\, D:\)
- Usar BACKSLASH (\) no forward slash (/)
```

**Solución**: Usar formato Windows correcto: `C:\\Temp\\Directorio`

#### Error: "Conectividad WinRM fallida"

```
ERROR: No se puede conectar al servidor via WinRM
```

**Solución**: 
1. Verificar credenciales en inventario
2. Confirmar conectividad de red
3. Validar configuración WinRM en servidor destino

### Debug Mode

```bash
# Ejecutar con debug detallado
ansible-playbook file_sync.yml -i inventory/windows/hosts -vvv \
  -e '{"source_server": "rdpinfralab", 
       "target_server": "lbdgvrdpdatos", 
       "sync_path": "C:\\Temp", 
       "sync_patterns": ["*.txt"]}'
```

### Validación Manual

```bash
# Test conectividad básica
ansible -i inventory/windows/hosts windows_servers -m win_ping

# Validar ruta específica
ansible -i inventory/windows/hosts rdpinfralab -m win_stat \
  -a "path='C:\\Temp\\Prueba Copia'"

# Test robocopy manual
ansible -i inventory/windows/hosts rdpinfralab -m win_shell \
  -a "robocopy C:\\Source C:\\Dest *.txt /L"
```

## Optimización y Performance

### Configuración de Robocopy

```yaml
# Para archivos grandes
file_sync_robocopy_flags: "/E /XO /R:5 /W:15 /MT:8 /NFL /NDL"

# Para muchos archivos pequeños
file_sync_robocopy_flags: "/E /XO /R:2 /W:5 /MT:16"

# Para transferencias WAN
file_sync_robocopy_flags: "/E /XO /R:10 /W:30 /MT:2"
```

### Filtros de Exclusión

```bash
# Excluir archivos temporales
sync_patterns: ["*.txt", "*.ps1"]
# Robocopy automáticamente filtra solo estos tipos
```

## Extensiones y Personalización

### Nuevos Tipos de Archivo

Para soportar nuevos patrones de archivo:

```yaml
sync_patterns: ["*.xml", "*.json", "*.yaml", "*.conf"]
```

### Modificación de Flags Robocopy

```yaml
# Agregar logging detallado
file_sync_robocopy_flags: "/E /XO /R:3 /W:10 /MT:4 /LOG:C:\\sync.log"

# Modo mirror (cuidado: borra archivos extra)
file_sync_robocopy_flags: "/MIR /R:3 /W:10 /MT:4"
```

### Hooks Personalizados

Para ejecutar acciones pre/post sincronización, modificar `sync_files.yml`:

```yaml
- name: "Pre-sincronización hooks"
  ansible.windows.win_shell: |
    # Acciones previas
    
- name: "Sincronización principal"
  # ... código existente ...
  
- name: "Post-sincronización hooks"
  ansible.windows.win_shell: |
    # Acciones posteriores
```

## Seguridad

### Mejores Prácticas

1. **Credenciales**: Usar Ansible Vault para passwords
2. **Permisos**: Validar permisos UNC antes de sincronización
3. **Logging**: Habilitar logging detallado para auditoría
4. **Backup**: Siempre hacer backup antes de sincronización masiva

### Configuración Segura

```yaml
# En group_vars/windows_servers/vault.yml (encriptado)
ansible_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653162336...
```

## Contribución

### Estructura para Nuevas Características

1. **Nuevas validaciones**: Agregar en `validate_prerequisites.yml`
2. **Nuevos tipos de análisis**: Extender `compare_files.yml`
3. **Nuevos métodos de sync**: Crear tasks en `sync_files.yml`
4. **Nuevas métricas**: Actualizar `generate_stats.yml`

### Testing

```bash
# Test en modo dry-run
ansible-playbook file_sync.yml --check

# Test con archivos de prueba
mkdir /tmp/test_files
touch /tmp/test_files/{file1.txt,file2.ps1,file3.bat}
```

## Referencias

- [Documentación Robocopy](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)
- [Ansible Windows Modules](https://docs.ansible.com/ansible/latest/collections/ansible/windows/)
- [AWX set_stats Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_stats_module.html)

---

**Autor**: cgarzont (NTTDATA)  
**Versión**: 1.0.0  
**Compatibilidad**: Ansible 2.9+, Windows Server 2016+  
**Última actualización**: Agosto 2025