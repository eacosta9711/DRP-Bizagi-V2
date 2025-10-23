# Ansible Role: cards

## Descripción

Rol Ansible para envío de notificaciones con tarjetas visuales a plataformas de comunicación empresarial. Actualmente soporta Google Chat con arquitectura extensible para Teams y Slack.

## Características

### Plataformas Soportadas

- **Google Chat**: Implementación completa con tarjetas rich card
- **Teams**: Preparado para implementación futura
- **Slack**: Preparado para implementación futura

### Funcionalidades

- Envío de notificaciones con formato de tarjeta visual
- Templates personalizables con Jinja2
- Validación automática de parámetros requeridos
- Manejo robusto de errores de conectividad
- Debug opcional para troubleshooting
- Configuración centralizada y parametrizable

## Estructura del Rol

```
roles/cards/
├── defaults/main.yml          # Configuración por defecto
├── vars/main.yml             # Variables del rol
├── tasks/main.yml            # Tareas principales
├── tasks/google_chat.yml     # Implementación Google Chat
├── templates/
│   └── google_chat_card.json.j2  # Template de tarjeta
└── README.md                 # Este archivo
```

## Variables

### Variables Requeridas

| Variable | Tipo | Descripción | Ejemplo |
|----------|------|-------------|---------|
| `input_webhook_url` | String | URL del webhook de destino | `https://chat.googleapis.com/v1/spaces/...` |

### Variables del Rol

| Variable | Tipo | Default | Descripción |
|----------|------|---------|-------------|
| `cards_notification_platform` | String | `google_chat` | Plataforma de destino |

### Variables de Template

| Variable | Tipo | Requerida | Descripción |
|----------|------|-----------|-------------|
| `deployment_status` | String | Sí | Estado del deployment (success/failed) |
| `playbook_name` | String | Sí | Nombre del playbook ejecutado |
| `execution_date` | String | Sí | Fecha de ejecución |
| `execution_time` | String | Sí | Hora de ejecución |
| `target_host` | String | Sí | Host objetivo |
| `execution_user` | String | No | Usuario que ejecutó |
| `duration` | String | No | Duración de la ejecución |
| `notification_title` | String | No | Título personalizado |
| `card_title` | String | No | Título de la tarjeta |
| `card_subtitle` | String | No | Subtítulo de la tarjeta |

## Uso

### Playbook Básico

```yaml
---
- name: Enviar notificación con tarjeta
  hosts: localhost
  vars_files:
    - roles/cards/vars/main.yml
  
  tasks:
    - name: Determinar estado del despliegue
      ansible.builtin.set_fact:
        deployment_status: success
        playbook_name: "{{ playbook_dir | basename }}"
        target_host: "{{ ansible_hostname | default('localhost') }}"
        execution_date: "{{ ansible_date_time.date }}"
        execution_time: "{{ ansible_date_time.time }}"
        execution_timestamp: "{{ ansible_date_time.iso8601 }}"
        execution_user: "Ansible Automation"

    - name: Ejecutar role cards
      ansible.builtin.include_role:
        name: cards
```

### Ejecución desde Línea de Comandos

```bash
# Notificación básica
ansible-playbook webhook.yml \
  -e input_webhook_url="https://chat.googleapis.com/v1/spaces/XXXXX/messages?key=XXXXX&token=XXXXX"

# Con variables personalizadas
ansible-playbook webhook.yml \
  -e input_webhook_url="https://chat.googleapis.com/v1/spaces/XXXXX/messages?key=XXXXX&token=XXXXX" \
  -e notification_title="Deployment Production" \
  -e card_title="Sistema Bizagi" \
  -e card_subtitle="Actualización v2.1.0"
```

### Integración en Playbooks Existentes

```yaml
- name: Mi playbook principal
  hosts: all
  tasks:
    # ... tareas del playbook ...
    
  post_tasks:
    - name: Notificar resultado
      ansible.builtin.include_role:
        name: cards
      vars:
        deployment_status: "{{ ansible_failed_result is not defined | ternary('success', 'failed') }}"
        playbook_name: "Deployment Bizagi"
        execution_user: "{{ ansible_user }}"
      delegate_to: localhost
      run_once: true
```

## Configuración de Google Chat

### Obtener Webhook URL

1. En Google Chat, crear un espacio o usar uno existente
2. Configurar webhook en el espacio
3. Copiar la URL generada
4. Usar la URL como valor de `input_webhook_url`

### Formato de la URL

```
https://chat.googleapis.com/v1/spaces/{SPACE_ID}/messages?key={API_KEY}&token={TOKEN}
```

## Template Personalizado

### Estructura del Template

El template `google_chat_card.json.j2` genera un JSON compatible con Google Chat API:

```json
{
  "text": "Título de la notificación",
  "cards": [
    {
      "header": {
        "title": "Título de la tarjeta",
        "subtitle": "Subtítulo"
      },
      "sections": [
        {
          "widgets": [
            {
              "keyValue": {
                "topLabel": "Estado",
                "content": "SUCCESS",
                "icon": "STAR"
              }
            }
          ]
        }
      ]
    }
  ]
}
```

### Personalización

Para personalizar el template, modifica `templates/google_chat_card.json.j2`:

```jinja2
{
  "text": "{{ custom_title | default('Notificación de Ansible') }}",
  "cards": [
    {
      "header": {
        "title": "{{ project_name | default('Ansible Deployment') }}",
        "subtitle": "{{ environment | default(playbook_name) }}"
      }
      // ... resto del template
    }
  ]
}
```

## Variables de Configuración

### Iconos Disponibles

```yaml
cards_google_chat:
  icons:
    success: "STAR"
    failed: "REPORT_PROBLEM" 
    running: "CLOCK"
    warning: "EMAIL"
```

### Iconos Google Chat Soportados

- `STAR` - Estrella (éxito)
- `REPORT_PROBLEM` - Advertencia (error)
- `CLOCK` - Reloj (en proceso)
- `EMAIL` - Email (información)
- `PERSON` - Persona (usuario)
- `DESCRIPTION` - Documento (archivo)

## Manejo de Errores

### Validaciones Automáticas

El rol incluye validaciones automáticas:

```yaml
- name: "Validar webhook URL esté definida"
  ansible.builtin.assert:
    that:
      - input_webhook_url is defined
      - input_webhook_url | length > 0
    fail_msg: "ERROR: webhook URL no definida"
```

### Códigos de Respuesta

| Código | Significado | Acción |
|--------|-------------|--------|
| 200 | Éxito | Notificación enviada correctamente |
| 400 | Error de formato | Verificar template JSON |
| 401 | No autorizado | Verificar webhook URL |
| 404 | No encontrado | Verificar espacio de Chat |

## Debug y Troubleshooting

### Habilitar Debug

```bash
ansible-playbook webhook.yml -v \
  -e input_webhook_url="https://..."
```

### Variables de Debug

El rol muestra información adicional con `-v`:

```yaml
- name: Debug - Mostrar payload generado
  ansible.builtin.debug:
    var: chat_payload
  when: ansible_verbosity >= 1
```

### Errores Comunes

#### Error: webhook URL no definida

```
TASK [cards : Validar webhook URL esté definida] ***
fatal: [localhost]: FAILED! => {
    "msg": "ERROR: webhook URL no definida"
}
```

**Solución:** Definir `input_webhook_url`

#### Error: JSON malformado

```
TASK [cards : Enviar notificación] ***
fatal: [localhost]: FAILED! => {
    "status": 400,
    "msg": "Bad Request"
}
```

**Solución:** Verificar sintaxis del template JSON

#### Error: Webhook inválido

```
TASK [cards : Enviar notificación] ***
fatal: [localhost]: FAILED! => {
    "status": 404,
    "msg": "Not Found"
}
```

**Solución:** Verificar URL del webhook

## Extensibilidad

### Agregar Nueva Plataforma

Para agregar soporte a Teams:

1. Crear `tasks/teams.yml`
2. Crear `templates/teams_card.json.j2`
3. Actualizar `tasks/main.yml`:

```yaml
- name: Include Teams notifications
  ansible.builtin.include_tasks: teams.yml
  when:
    - cards_notification_platform == "teams"
```

### Variables por Plataforma

```yaml
cards_teams:
  webhook_url: "{{ input_webhook_url }}"
  colors:
    success: "00FF00"
    failed: "FF0000"
```

## Dependencias

### Módulos Ansible Requeridos

- `ansible.builtin.uri` - Para envío HTTP
- `ansible.builtin.template` - Para procesamiento de templates
- `ansible.builtin.assert` - Para validaciones

### Conectividad

- Conectividad HTTPS a la plataforma de destino
- DNS funcional para resolución de nombres

## Desarrollo

### Testing Local

```bash
# Test con webhook de prueba
ansible-playbook webhook.yml \
  -e input_webhook_url="https://httpbin.org/post" \
  -v
```

### Validación de Template

```bash
# Generar solo el JSON sin enviar
ansible-playbook -i localhost, -c local webhook.yml \
  --tags never \
  -e input_webhook_url="dummy" \
  -v
```

## Changelog

### v1.0.0
- Implementación inicial con soporte para Google Chat
- Template básico de tarjetas
- Validaciones automáticas
- Manejo de errores

## Licencia

Este proyecto es para uso interno de automatización.

---

**Autor**: NTTDATA  
**Versión**: 1.0.0  
**Compatibilidad**: Ansible 2.9+  
**Mantenedor**: Equipo Automatización NTTDATA
