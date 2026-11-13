# 🐧 SysAdmin Playbook & Linux Notes (AlmaLinux / RHEL)

Este repositorio contiene mi documentación personal, plantillas y lecciones aprendidas durante mi entrenamiento como Administrador de Sistemas Linux. Todo el contenido está orientado a **estándares de producción**, aplicando el Principio de Mínimo Privilegio (PoLP) y Cero Tiempo de Inactividad (Zero Downtime).

---

## 🏛️ 1. Las 4 Reglas de Oro de Producción
1. **Principio de Mínimo Privilegio (PoLP):** Nunca dar permisos globales si podemos dar permisos específicos. (Ej. Usar `httpd_can_network_relay` en lugar de `httpd_can_network_connect`).
2. **Idempotencia:** Un script debe poder ejecutarse 100 veces y dejar el sistema en el mismo estado. (Ej. Usar `mkdir -p`).
3. **Cero Tiempo de Inactividad (Zero Downtime):** Usar `systemctl reload` en lugar de `restart` siempre que sea posible.
4. **Observabilidad:** Si el servidor falla a las 3 AM, los logs (`journalctl`, `/var/log/`) deben decir exactamente qué pasó.

---

## 🔄 2. Git y Control de Versiones

### El flujo de trabajo correcto
**REGLA DE ORO:** NUNCA uses `sudo` con Git. Git debe trabajar con tu usuario en tu carpeta personal (`~`), no en carpetas del sistema (`/etc`, `/usr`). Si un archivo del sistema necesita ir a Git, se copia a tu carpeta de proyectos y se le cambia el dueño (`chown`).

### Configuración inicial de SSH (Solo una vez)
```bash
# Generar llave SSH
ssh-keygen -t ed25519 -C "felixbonillotrabajo@gmail.com"

# Iniciar el agente y añadir la llave
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copiar la llave pública y pegarla en GitHub (Settings -> SSH and GPG keys)
cat ~/.ssh/id_ed25519.pub

# Probar la conexión (Debe decir: "Hi felixbonillo! You've successfully authenticated")
ssh -T git@github.com
```

### Comandos diarios
```bash
git init                                  # Iniciar repositorio local
git remote add origin git@github.com:usuario/repo.git # Conectar a GitHub
git status                                # Ver archivos modificados
git add .                                 # Preparar todos los cambios
git commit -m "feat: mensaje descriptivo" # Guardar cambios con mensaje semántico
git push origin main                      # Subir a GitHub
git pull origin main                      # Traer cambios desde GitHub
```

---

## 🛠️ 3. Bash Scripting Avanzado

### Estructura Base
```bash
#!/bin/bash
set -euo pipefail # -e: sale si falla, -u: error si variable vacía, -o pipefail: error en tuberías

LOG_FILE="/var/log/mi_script.log"
fecha=$(date +%F)

imprimir() {
    echo "$1" | tee -a "$LOG_FILE"
}
```

### Comandos Esenciales
- `mkdir -p /ruta`: Crea carpetas sin dar error si ya existen.
- `tar -czf archivo.tar.gz /carpeta`: Comprimir sin ruido en pantalla (sin `-v`).
- `find /ruta -name "*.tar.gz" -mtime +7 -delete`: Buscar y borrar archivos de más de 7 días.
- `awk 'NR==2 {print $5}'`: Extraer la columna 5 de la fila 2 de un comando.

---

## ⚙️ 4. Systemd: Servicios y Timers

### Plantilla de Servicio (`/etc/systemd/system/mi_app.service`)
```ini
[Unit]
Description=Descripcion del servicio
After=network.target

[Service]
Type=simple           # oneshot para scripts que terminan rápido; simple para demonios
WorkingDirectory=/ruta/de/trabajo
ExecStart=/usr/bin/ejecutable --bandera

[Install]
WantedBy=multi-user.target
```

### Plantilla de Timer (`/etc/systemd/system/mi_app.timer`)
```ini
[Unit]
Description=Ejecutar mi servicio diariamente

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### Ciclo de Vida (La diferencia crucial)
- `sudo systemctl daemon-reload`: **Obligatorio** al crear o editar un archivo `.service` o `.timer`.
- `sudo systemctl enable --now <servicio>`: Habilita al arranque e inicia ahora.
- `sudo systemctl reload <servicio>`: Recarga configuración **sin cortar el servicio** (Zero Downtime).
- `sudo systemctl restart <servicio>`: Mata y reinicia el proceso (Micro-cortes).
- `journalctl -xeu <servicio>`: Ver logs detallados de fallos.

---

## 🛡️ 5. SELinux (Seguridad MAC)

**REGLA DE ORO:** NUNCA desactives SELinux (`setenforce 0`). Se negocia con él.

### A. Contextos de Archivos (Permisos de lectura/escritura)
Cuando un servicio (ej. Nginx) necesita leer archivos en una carpeta no estándar:
```bash
# 1. Crear la regla de contexto
sudo semanage fcontext -a -t httpd_sys_content_t "/ruta/personalizada(/.*)?"
# 2. Aplicar la regla a los archivos existentes
sudo restorecon -Rv /ruta/personalizada
# 3. Verificar la etiqueta
ls -Z /ruta/personalizada
```

### B. Booleanos (Permisos de comportamiento)
Cuando un servicio necesita hacer algo que SELinux bloquea por defecto:
```bash
# Ver booleanos relacionados (ej. httpd)
getsebool -a | grep httpd
# Activar de forma permanente (-P)
sudo setsebool -P httpd_can_network_connect on
```
*Nota de Producción:* `httpd_can_network_relay` es más seguro que `connect` para Reverse Proxies.

### C. Puertos Personalizados
Si un servicio (ej. SSH) cambia de puerto, SELinux lo bloquea. Se debe declarar:
```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -a -t http_port_t -p tcp 8000
```

---

## 🌐 6. Redes, Firewalld y Nginx

### Firewalld (El Guardián)
```bash
sudo firewall-cmd --list-all                   # Ver reglas actuales
sudo firewall-cmd --add-service=http --permanent # Abrir puerto (permanente)
sudo firewall-cmd --add-port=2222/tcp --permanent # Abrir puerto específico
sudo firewall-cmd --reload                     # Aplicar cambios permanentes
```

### Nginx (El Servidor Web)
- **Archivo principal:** `/etc/nginx/nginx.conf` (No tocar si es posible).
- **Archivos de sitios:** `/etc/nginx/conf.d/*.conf`.
- **Verificar sintaxis:** `sudo nginx -t` (SIEMPRE antes de recargar).
- **Recargar:** `sudo systemctl reload nginx`.

**Solución de problemas comunes:**
- **Error 403 Forbidden:** Permisos de archivo mal o SELinux bloqueando lectura (`semanage fcontext`).
- **Error 502 Bad Gateway:** Nginx no puede hablar con el Backend (App caída o SELinux booleano bloqueando conexión).
- **Conflicto de puerto 80:** Revisar `nginx -T | grep server {` y buscar bloques duplicados.

---

## 📦 7. Logrotate (Gestión de Logs)
Evita que los logs llenen el disco.
Archivo de configuración: `/etc/logrotate.d/mi_servicio`
```text
/var/log/mi_log.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    dateext
}
```
- `sudo logrotate -d /etc/logrotate.d/mi_servicio`: Simular (debug).
- `sudo logrotate -f /etc/logrotate.d/mi_servicio`: Forzar rotación inmediata.

---

## 🔗 8. Enlaces Simbólicos (Symlinks)
```bash
# Crear enlace (SIEMPRE usar rutas absolutas)
sudo ln -s /ruta/al/archivo/real /usr/bin/comando_corto
# Ver a dónde apunta
ls -l /usr/bin/comando_corto
```