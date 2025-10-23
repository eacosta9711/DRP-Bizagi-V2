# Ansible Role: test_connection

## Descripción

Rol Ansible reutilizable para validar conectividad de red desde hosts pivote hacia múltiples destinos usando protocolos telnet y/o ping, con soporte multi-plataforma (Linux/Windows).

## Funcionalidades

### **Tipos de Conectividad Soportados**

- **telnet**: Validación TCP a puerto específico
- **ping**: Validación ICMP básica
- **ambos**: Ejecución combinada de telnet + ping

### **Plataformas Soportadas**

- **Linux**: RHEL, CentOS, Debian, Ubuntu
- **Windows**: Windows Server 2016+, PowerShell 5.1+

### **Características Avanzadas**

- Reintentos automáticos configurable
- Manejo robusto de errores
- Salidas estructuradas para AWX
- Timeouts personalizables
- Logging detallado

## Instalación

### Desde Línea de Comandos

```bash
# Clonar el proyecto completo
git clone <repository_url>
cd catalogo_automatizaciones
```

## Variables del Rol

### Variables Requeridas

| Variable                       | Tipo   | Descripción           | Ejemplo                         |
| ------------------------------ | ------ | --------------------- | ------------------------------- |
| `test_connection_target_hosts` | Lista  | IPs destino a validar | `["90.5.0.16", "10.8.32.10"]`   |
| `test_connection_check_type`   | String | Tipo de test          | `"telnet"`, `"ping"`, `"ambos"` |

### Variables Condicionales

| Variable                | Tipo  | Requerida Cuando            | Descripción |
| ----------------------- | ----- | --------------------------- | ----------- | --------------- |
| `test_connection_ports` | Lista | `check_type` = telnet/ambos | Puertos TCP | `["80", "443"]` |

### Variables Opcionales

| Variable                           | Tipo    | Default                | Descripción                  |
| ---------------------------------- | ------- | ---------------------- | ---------------------------- |
| `test_connection_retries`          | Integer | `3`                    | Número de reintentos         |
| `test_connection_retry_delay`      | Integer | `2`                    | Delay entre reintentos (seg) |
| `test_connection_task_id`          | String  | `"unknown"`            | ID único de la tarea         |
| `test_connection_task_description` | String  | `"Connectivity check"` | Descripción de la tarea      |
| `test_connection_show_details`     | Boolean | `true`                 | Mostrar detalles en pantalla |

## Uso del Rol

### Ejemplo Básico - Telnet

```yaml
- name: Validar conectividad DataPower
  include_role:
    name: test_connection
  vars:
    test_connection_check_type: "telnet"
    test_connection_target_hosts:
      - "90.5.0.16"
    test_connection_ports:
      - "9130"
    test_connection_task_id: "datapower_check"
    test_connection_task_description: "DataPower connectivity validation"
```

### Ejemplo Básico - Ping

```yaml
- name: Validar conectividad OKTA
  include_role:
    name: test_connection
  vars:
    test_connection_check_type: "ping"
    test_connection_target_hosts:
      - "90.5.3.1.134"
      - "90.5.3.1.135"
      - "10.8.4.23"
    test_connection_task_id: "okta_connectivity"
    test_connection_task_description: "OKTA ICMP validation"
```

### Ejemplo Avanzado - Combinado

```yaml
- name: Validación completa bus de servicios
  include_role:
    name: test_connection
  vars:
    test_connection_check_type: "ambos"
    test_connection_target_hosts:
      - "90.5.4.230"
      - "10.8.32.10"
    test_connection_ports:
      - "80"
    test_connection_task_id: "bus_services_full"
    test_connection_task_description: "Bus services full validation"
    test_connection_retries: 5
    test_connection_retry_delay: 3
```

## Salidas del Rol

### Variables de Resultado

El rol establece las siguientes variables de hechos:

| Variable                          | Tipo   | Descripción                        |
| --------------------------------- | ------ | ---------------------------------- |
| `connectivity_result`             | Dict   | Resultado completo de la ejecución |
| `connectivity_result.status`      | String | `"success"` o `"failed"`           |
| `connectivity_result.start_time`  | String | Timestamp de inicio                |
| `connectivity_result.task_id`     | String | ID de la tarea ejecutada           |
| `connectivity_result.source_host` | String | Host que ejecutó las pruebas       |
| `connectivity_result.results`     | Lista  | Resultados detallados por target   |

### Ejemplo de Resultado

```yaml
connectivity_result:
  start_time: "2025-07-18 - 10:30:00"
  task_id: "datapower_check"
  source_host: "linux_prod"
  status: "success"
  results:
    - "TELNET 90.5.0.16:9130 = SUCCESS"
    - "PING 90.5.0.16 = SUCCESS"
    - "RESUMEN: 2/2 exitosos (100.00%)"
```

### Stats para AWX

El rol registra automáticamente estadísticas usando `set_stats`:

```yaml
task_datapower_check:
  task_info:
    task_id: "datapower_check"
    description: "DataPower connectivity validation"
    target: "90.5.0.16:9130"
    execution_time: "2025-07-18T10:30:00Z"
  hosts:
    linux_prod:
      status: "success"
      timestamp: "2025-07-18T10:30:00Z"
      details: ["TELNET 90.5.0.16:9130 = SUCCESS"]
```

## Salidas en Pantalla

### Formato de Salida

```
====================================
TAREA datapower_check: DataPower connectivity validation
====================================
Desde: linux_prod
Tipo: telnet
Destinos: 90.5.0.16
Puertos: 9130
====================================

TELNET 90.5.0.16:9130 = SUCCESS

RESUMEN: 1/1 exitosos (100.00%)

====================================
TAREA datapower_check completada:
====================================
Hora: 2025-07-18 - 10:30:00
Host: linux_prod
Status: success
Results: ['TELNET 90.5.0.16:9130 = SUCCESS', 'RESUMEN: 1/1 exitosos (100.00%)']
====================================
```

## Testing del Rol

### Test Manual

```bash
# Test básico telnet
ansible-playbook -i inventory/linux/hosts test_role.yml \
  -e test_connection_target_hosts='["8.8.8.8"]' \
  -e test_connection_ports='["53"]' \
  -e test_connection_check_type="telnet"

# Test básico ping
ansible-playbook -i inventory/linux/hosts test_role.yml \
  -e test_connection_target_hosts='["8.8.8.8"]' \
  -e test_connection_check_type="ping"
```

### Playbook de Test

```yaml
# test_role.yml
---
- name: Test del rol test_connection
  hosts: all
  gather_facts: true

  tasks:
    - name: Test conectividad con Google DNS
      include_role:
        name: test_connection
      vars:
        test_connection_check_type: "{{ test_connection_check_type }}"
        test_connection_target_hosts: "{{ test_connection_target_hosts }}"
        test_connection_ports: "{{ test_connection_ports | default([]) }}"
        test_connection_task_id: "test_google_dns"
```

## Troubleshooting

### Errores Comunes

#### 1. "Parámetro requerido faltante"

```yaml
# Error
test_connection_target_hosts: []

# Solución
test_connection_target_hosts:
  - "90.5.0.16"
```

#### 2. "Puertos requeridos para check_type telnet"

```yaml
# Error
test_connection_check_type: "telnet"
# test_connection_ports no definido

# Solución
test_connection_check_type: "telnet"
test_connection_ports:
  - "80"
```

#### 3. "SYSTEM_ERROR" en resultados

**Causas posibles:**

- Host pivote no alcanzable
- Credenciales incorrectas
- PowerShell no disponible (Windows)
- Timeout de comandos

**Diagnóstico:**

```bash
# Verificar conectividad básica
ansible -i inventory all -m ping

# Test manual de comando
ansible -i inventory linux_pivote -m shell -a "echo >/dev/tcp/8.8.8.8/53"
ansible -i inventory windows_pivote -m win_shell -a "Test-NetConnection 8.8.8.8 -Port 53"
```

### Debug Avanzado

#### Habilitar Debug del Rol

```yaml
- name: Debug test_connection
  include_role:
    name: test_connection
  vars:
    test_connection_show_details: true
    # Otras variables...
  debug: true
```

#### Variables de Debug Útiles

```bash
# Mostrar todas las variables del rol
ansible-playbook playbook.yml -e debug_vars=true -vvv

# Solo mostrar templates generados
ansible-playbook playbook.yml -e debug_templates=true
```

## Personalización

### Nuevos Tipos de Test

Para agregar un nuevo tipo de test (ej: `http`):

1. **Actualizar templates**:

   ```bash
   # En templates/test_connection_linux.sh.j2
   {% if test_connection_check_type in ['http', 'ambos'] %}
   # HTTP tests aquí
   {% endif %}
   ```

2. **Actualizar validaciones**:
   ```yaml
   # En tasks/main.yml
   - name: "Validar puertos para http"
     ansible.builtin.fail:
       msg: "Puertos requeridos para check_type http"
     when: test_connection_check_type in ['http']
       and (test_connection_ports is not defined)
   ```

### Templates Personalizados

```yaml
# Usar template personalizado
- name: Test con template custom
  include_role:
    name: test_connection
  vars:
    test_connection_custom_template: "custom_connectivity.sh.j2"
```

## Dependencias

### Paquetes del Sistema

#### Linux

```bash
# Requeridos
bash >= 4.0
bc  # Para cálculos de porcentajes

# Opcionales
nc  # netcat como alternativa
telnet  # telnet tradicional como alternativa
```

#### Windows

```bash
# Requeridos
PowerShell >= 5.1
Test-NetConnection cmdlet
Test-Connection cmdlet
```

### Módulos Ansible

```yaml
# En collections/requirements.yml
collections:
  - name: ansible.windows
    version: ">=1.0.0"
```

## Contribución

### Estructura del Rol

```
roles/test_connection/
├── defaults/main.yml          # Variables por defecto
├── tasks/main.yml            # Tareas principales
├── tasks/linux.yml           # Tareas específicas Linux
├── tasks/windows.yml         # Tareas específicas Windows
├── templates/                # Templates Jinja2
│   ├── test_connection_linux.sh.j2
│   └── test_connection_windows.ps1.j2
├── vars/main.yml            # Variables del rol
└── README.md                # Este archivo
```

### Guidelines de Desarrollo

1. **Compatibilidad**: Mantener soporte para Linux y Windows
2. **Idempotencia**: Las tareas deben ser repetibles
3. **Error Handling**: Usar bloques rescue apropiados
4. **Testing**: Probar en ambas plataformas antes de commit
5. **Documentación**: Actualizar README con nuevas funcionalidades

## Referencias

- [Ansible Role Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [AWX set_stats Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_stats_module.html)
- [Jinja2 Template Documentation](https://jinja.palletsprojects.com/)

---

**Autor**: cgarzont (NTTDATA)  
**Versión del Rol**: 1.0.0  
**Compatibilidad**: Ansible 2.9+, AWX 15.0+  
**Mantenedor**: Equipo Automatizacion NTTDATA
