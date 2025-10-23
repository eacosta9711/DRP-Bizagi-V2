# Rol de Ansible: SQL Failover

Este rol automatiza el proceso de failover manual para bases de datos SQL Server configuradas con Mirroring.

## Funcionalidades

- **Validación de Prerrequisitos**: Comprueba la conectividad, reglas de firewall, y configuración de red de SQL Server antes de actuar.
- **Ejecución de Failover**: Orquesta los pasos para realizar el failover, incluyendo la verificación del estado de la base de datos y la reducción del log transaccional si es necesario.
- **Modularidad**: Todas las configuraciones importantes (puertos, credenciales, umbrales) son variables que se pueden sobrescribir.
- **Reversibilidad**: Permite ejecutar el failover de producción a contingencia y viceversa cambiando los parámetros de entrada.

## Requisitos

- Colección `community.windows` de Ansible: `ansible-galaxy collection install community.windows`
- Servidores de destino deben ser Windows y estar en un grupo `[windows_servers]` en el inventario.
- Credenciales de WinRM configuradas para conectar a los servidores Windows.

## Variables del Rol

Las variables por defecto se encuentran en `defaults/main.yml`.

### Variables Principales

- `db_name`: (Obligatorio) El nombre de la base de datos sobre la que se actuará.
- `primary_server`: (Obligatorio) El hostname del servidor principal actual (origen del failover).
- `secondary_server`: (Obligatorio) El hostname del servidor secundario (destino del failover).

### Variables de Autenticación

- `sql_auth_mode`: Modo de autenticación. `windows` (default) o `sql`.
- `sql_user`: Usuario para la autenticación SQL. Requerido si `sql_auth_mode` es `sql`.
- `sql_password`: Contraseña para la autenticación SQL. Requerido si `sql_auth_mode` es `sql`.
- `sql_encrypt`: Si se debe usar cifrado en la conexión. `true` (default) o `false`.

### Variables de Configuración

- `sql_port`: Puerto de la instancia de SQL Server. Default: `1433`.
- `mirroring_port`: Puerto usado para el mirroring. Default: `5022`.
- `log_size_threshold_gb`: Umbral en GB para el log transaccional antes de ejecutar un `SHRINK`. Default: `2`.