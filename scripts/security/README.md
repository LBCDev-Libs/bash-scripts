# Scripts de Seguridad

## web_security_laravel.sh

Endurece la seguridad de aplicaciones Laravel a nivel de sistema de archivos.

### Descripción

Configura automáticamente permisos y propietarios de archivos para minimizar riesgos de seguridad en aplicaciones Laravel en producción. Implementa el principio de mínimo privilegio, permitiendo al servidor web solo los permisos estrictamente necesarios.

**Ideal para:** Servidores de producción, VPS, entornos compartidos, pipelines de CI/CD.

### Uso

```bash
sudo ./web_security_laravel.sh [APP_PATH] [OPCIONES]
```

### Parámetros

#### Argumentos Posicionales

- `APP_PATH` (opcional): Ruta a la aplicación Laravel
  - Default: **Directorio actual** donde se ejecuta el script
  - Puede ser ruta absoluta o relativa

#### Opciones

- `--web-user USER`: Usuario del servidor web
  - Default: `www-data`
  - Ejemplos: `nginx`, `apache`, `www-data`
  - Debe ser un usuario existente en el sistema

- `--owner USER`: Usuario propietario de los archivos
  - Default: **Usuario que ejecuta sudo** (o usuario actual si no se usa sudo)
  - Debe ser un usuario existente en el sistema
  - Automáticamente detecta el usuario real cuando se usa sudo

- `--force`: Omite la confirmación interactiva
  - Útil para scripts automatizados, pipelines de CI/CD, o despliegues automáticos
  - **⚠️ PRECAUCIÓN**: Asegúrate de estar en el directorio correcto antes de usar esta opción

- `-h, --help`: Muestra la ayuda del script

### Ejemplos

```bash
# Aplicar en el directorio actual (detecta automáticamente tu usuario)
cd /var/www/myapp
sudo ./web_security_laravel.sh

# Especificar solo la ruta de la aplicación
sudo ./web_security_laravel.sh /var/www/myapp

# Cambiar usuario web (útil si usas nginx en lugar de apache)
sudo ./web_security_laravel.sh /var/www/myapp --web-user nginx

# Especificar todos los parámetros (ej: servidor con usuario deploy)
sudo ./web_security_laravel.sh /var/www/myapp --web-user nginx --owner deploy

# Los parámetros pueden ir en cualquier orden
sudo ./web_security_laravel.sh --owner john --web-user www-data /var/www/app

# Ejecutar sin confirmación (ideal para automatización)
sudo ./web_security_laravel.sh /var/www/myapp --force

# Combinar --force con otros parámetros
sudo ./web_security_laravel.sh /var/www/myapp --web-user nginx --owner deploy --force

# Ver ayuda completa
./web_security_laravel.sh --help
```

### Instalación en VPS (Recomendado)

Para usar este script en múltiples servidores de producción:

```bash
# 1. Clonar el repositorio en tu VPS
ssh usuario@tu-vps.com
git clone https://github.com/LBCDev-Libs/bash-scripts.git ~/.scripts-repo

# 2. Crear enlace simbólico
mkdir -p ~/bin
ln -s ~/.scripts-repo/scripts/security/web_security_laravel.sh ~/bin/harden-laravel

# 3. Añadir al PATH (opcional)
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 4. Usar desde cualquier directorio
cd /var/www/mi-aplicacion
sudo harden-laravel

# 5. Actualizar el script cuando haya cambios
cd ~/.scripts-repo && git pull
```

### Validaciones y Seguridad

El script incluye múltiples capas de validación para prevenir errores:

#### Validaciones Automáticas

- ✅ Verifica que el directorio de la aplicación existe
- ✅ Convierte rutas relativas a absolutas
- ✅ Verifica que se ejecuta con permisos de root/sudo
- ✅ Verifica que el usuario web existe en el sistema
- ✅ Verifica que el usuario propietario existe en el sistema

#### Detección de Estructura Laravel

El script analiza el directorio para verificar que es una aplicación Laravel:

- Busca directorios típicos: `app`, `bootstrap`, `config`, `database`, `public`, `resources`, `routes`, `storage`
- Verifica archivos clave: `artisan`, `composer.json`
- **Advertencia**: Si faltan más de 3 directorios típicos, muestra un warning antes de continuar

#### Confirmación Interactiva

Antes de aplicar cambios, el script:

1. **Muestra la ruta completa** donde se aplicarán los permisos
2. **Repite el warning de Laravel** si el directorio no parece ser una aplicación Laravel (para máxima visibilidad)
3. **Lista todos los cambios** que se realizarán (propietarios, permisos)
4. **Solicita confirmación explícita** del usuario (debe escribir "si")
5. **Permite cancelar** en cualquier momento sin hacer cambios

**Nota sobre --force**: Al usar el parámetro `--force`, se omite el paso de confirmación interactiva. El script mostrará toda la información de configuración y advertencias, pero procederá automáticamente sin solicitar confirmación.

Si el directorio **NO** parece Laravel, el warning se muestra **dos veces**:

- Una vez durante la verificación inicial
- **Otra vez justo antes del prompt de confirmación** (para que sea imposible pasarlo por alto)

### Qué Hace el Script

1. **Muestra configuración**: Presenta los parámetros que se aplicarán
2. **Establece propietarios**: Usuario del sistema como dueño, servidor web como grupo
3. **Permisos base**: 755 para directorios, 644 para archivos (solo lectura para web)
4. **Excepciones Laravel**: 775 en `storage/` y `bootstrap/cache/` (escritura necesaria)
5. **Protección .env**: 640 (solo propietario y grupo pueden leer)
6. **Manejo especial SQLite**: Si detecta archivos `.sqlite`, les aplica 664 (escritura necesaria)

#### ⚠️ Nota sobre SQLite

Si tu aplicación usa SQLite, el script automáticamente:

- Detecta archivos `.sqlite` en `database/` y `storage/database/`
- Les aplica permisos `664` (rw-rw-r--) para permitir escritura por el servidor web
- Aplica permisos `775` al directorio `database/` (en lugar de `755`) para permitir archivos temporales
- Esto es necesario porque SQLite necesita:
  - Escribir en el archivo de base de datos
  - Crear archivos temporales de lock en el mismo directorio

**Ubicaciones verificadas:**

- `database/*.sqlite`
- `storage/database/*.sqlite`

**Permisos especiales para SQLite:**

- Archivos `.sqlite`: `664` (rw-rw-r--)
- Directorio `database/`: `775` (rwxrwxr-x) cuando contiene SQLite, `755` (rwxr-xr-x) en caso contrario

### Configuración Adicional Requerida en Nginx

**⚠️ CRÍTICO PARA SEGURIDAD**: Después de ejecutar el script, debes configurar Nginx para completar el endurecimiento.

El script proporciona al final las instrucciones exactas de Nginx que debes añadir. A continuación se explica en detalle:

#### Configuración Completa Recomendada

Añade estas reglas a tu configuración de Nginx (`/etc/nginx/sites-available/tu-sitio`, dentro del bloque `server`):

```nginx
server {
    listen 80;
    server_name tudominio.com;
    root /var/www/tu-app/public;

    index index.php index.html;

    # --- SEGURIDAD WEB ---

    # 1. PERMITIR ÚNICAMENTE el punto de entrada de Laravel
    # El uso de "=" da prioridad máxima y exclusividad.
    location = /index.php {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By; # Oculta que usas PHP
    }

    # 2. BLOQUEAR CUALQUIER OTRO ARCHIVO .php
    # Cualquier intento de ejecutar otro archivo .php en public o subcarpetas
    # morirá aquí con un 403, protegiéndote de WebShells subidas.
    location ~ \.php$ {
        deny all;
        return 403;
    }

    # --- BLOQUEO DE ARCHIVOS SENSIBLES Y OCULTOS ---

    # Bloquear archivos que empiezan por punto (.env, .git, .htaccess, etc.)
    # Exceptuamos .well-known para que Certbot pueda renovar certificados.
    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Bloquear extensiones peligrosas o de backup
    location ~* \.(env|log|sql|git|sh|bak|config|php~)$ {
        deny all;
    }

    # Configuración normal de Laravel
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

#### Explicación de las Reglas de Seguridad

##### **Regla 1: Solo permitir index.php**

```nginx
location = /index.php {
    fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    include fastcgi_params;
    fastcgi_hide_header X-Powered-By;
}
```

- El operador `=` da máxima prioridad y hace que esta regla sea exclusiva
- Solo `index.php` puede ejecutarse como PHP
- `fastcgi_hide_header X-Powered-By` oculta información sobre PHP en las cabeceras HTTP

##### **Regla 2: Bloquear todos los demás archivos PHP**

```nginx
location ~ \.php$ {
    deny all;
    return 403;
}
```

- Cualquier archivo `.php` que no sea `index.php` será bloqueado
- Protege contra:
  - WebShells subidos maliciosamente
  - Scripts de prueba olvidados (test.php, info.php, phpinfo.php)
  - Exploits que intentan ejecutar PHP en subdirectorios de `public/`

##### **Regla 3: Bloquear archivos ocultos (excepto .well-known)**

```nginx
location ~ /\.(?!well-known).* {
    deny all;
}
```

- Bloquea acceso a archivos que empiezan con punto: `.env`, `.git`, `.htaccess`, etc.
- Permite `.well-known` para que Certbot pueda renovar certificados SSL

##### **Regla 4: Bloquear extensiones peligrosas**

```nginx
location ~* \.(env|log|sql|git|sh|bak|config|php~)$ {
    deny all;
}
```

- Bloquea archivos de configuración, logs, backups y scripts
- Protege información sensible

#### Aplicar los cambios

```bash
# Verificar sintaxis de Nginx
sudo nginx -t

# Si todo está OK, recargar Nginx
sudo systemctl reload nginx
```

#### Verificar que funciona

```bash
# Intentar acceder a un PHP que no sea index.php (debe dar 403)
curl -I https://tudominio.com/test.php

# Intentar acceder a .env (debe dar 403)
curl -I https://tudominio.com/.env

# Intentar acceder a un archivo de backup (debe dar 403)
curl -I https://tudominio.com/config.bak

# Acceso normal a la aplicación (debe funcionar)
curl -I https://tudominio.com/
```

### Uso en CI/CD y Automatización

El parámetro `--force` hace que este script sea ideal para pipelines de CI/CD:

#### Ejemplo con GitHub Actions

```yaml
name: Deploy Laravel

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            
            # Aplicar permisos de seguridad automáticamente
            sudo /home/deploy/bin/harden-laravel --force
            
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
```

#### Ejemplo con GitLab CI/CD

```yaml
deploy:
  stage: deploy
  script:
    - ssh $DEPLOY_USER@$DEPLOY_HOST "
        cd /var/www/myapp &&
        git pull origin main &&
        composer install --no-dev --optimize-autoloader &&
        php artisan migrate --force &&
        sudo harden-laravel --force &&
        php artisan config:cache
      "
  only:
    - main
```

#### Script de Deploy Personalizado

```bash
#!/bin/bash
# deploy.sh - Script de despliegue automatizado

set -e

APP_PATH="/var/www/myapp"

echo "🚀 Iniciando despliegue..."

# Actualizar código
cd "$APP_PATH"
git pull origin main

# Instalar dependencias
composer install --no-dev --optimize-autoloader

# Migraciones
php artisan migrate --force

# Aplicar seguridad (sin confirmación)
sudo harden-laravel "$APP_PATH" --force

# Optimizaciones
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Reiniciar servicios
sudo systemctl reload php8.3-fpm
sudo systemctl reload nginx

echo "✅ Despliegue completado"
```

### Dependencias

- **sudo**: Permisos de superusuario
- **find**: Búsqueda de archivos (incluido en sistemas Unix)
- **chmod/chown**: Gestión de permisos (incluido en sistemas Unix)

### Características de Seguridad

- ✅ Principio de mínimo privilegio
- ✅ Servidor web sin permisos de escritura (excepto storage)
- ✅ Protección del archivo .env
- ✅ Prevención de ejecución de scripts maliciosos
- ✅ Separación de propietario y grupo
- ✅ Modo automatizado para CI/CD con `--force`

### Cuándo Usar

- ✅ Después de desplegar una aplicación Laravel
- ✅ Tras actualizar código en producción (deploy)
- ✅ Como parte de un proceso de hardening de servidor
- ✅ Cuando se detectan permisos incorrectos
- ✅ Al configurar un nuevo VPS para Laravel
- ✅ Después de clonar un repositorio en producción
- ✅ En pipelines de CI/CD (con `--force`)

### Flujo de Trabajo Típico en Producción

#### Despliegue Manual

```bash
# 1. Desplegar código (git pull, composer install, etc.)
cd /var/www/mi-app
git pull origin main
composer install --no-dev --optimize-autoloader

# 2. Aplicar permisos de seguridad
sudo harden-laravel
# El script mostrará:
# - Verificación de estructura Laravel
# - Ruta completa donde se aplicarán los cambios
# - Lista de permisos que se modificarán
# - Solicitud de confirmación (escribe 'si')

# 3. Limpiar caché de Laravel
php artisan config:cache
php artisan route:cache
php artisan view:cache

# 4. Verificar que todo funciona
curl -I https://mi-app.com
```

#### Despliegue Automatizado

```bash
# 1. Desplegar código
cd /var/www/mi-app
git pull origin main
composer install --no-dev --optimize-autoloader

# 2. Aplicar permisos de seguridad (sin confirmación)
sudo harden-laravel --force

# 3. Limpiar caché de Laravel
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### Ejemplo de Ejecución

#### Caso 1: Aplicación Laravel Válida (Modo Normal)

```text
$ cd /var/www/mi-aplicacion
$ sudo harden-laravel

🔍 Verificando estructura de Laravel...
✅ Estructura de Laravel detectada correctamente

🛡️ Configuración de Endurecimiento de Seguridad
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📁 Ruta completa:  /var/www/mi-aplicacion
👤 Propietario:    usuario
🌐 Usuario Web:    www-data
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  Este script modificará los permisos de TODOS los archivos en:
   /var/www/mi-aplicacion

Los cambios que se aplicarán:
  • Propietario: usuario:www-data
  • Directorios: 755 (rwxr-xr-x)
  • Archivos: 644 (rw-r--r--)
  • storage/: 775 (rwxrwxr-x)
  • bootstrap/cache/: 775 (rwxrwxr-x)
  • .env: 640 (rw-r-----)

¿Deseas continuar? (escribe 'si' para confirmar): si

✅ Confirmado. Iniciando proceso...

👤 Ajustando propietarios a usuario:www-data...
🔒 Aplicando permisos 755/644 (Solo lectura para el servidor web)...
📂 Otorgando permisos de escritura solo en storage y cache...
🔑 Asegurando archivo .env...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Proceso de permisos completado. App asegurada a nivel de sistema.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  ¡ATENCIÓN: CONFIGURACIÓN REQUERIDA EN NGINX! ⚠️

Para completar el endurecimiento de seguridad, añade esta sección
a tu configuración de Nginx (dentro del bloque 'server'):
...
```

#### Caso 2: Aplicación Laravel Válida (Modo --force)

```text
$ cd /var/www/mi-aplicacion
$ sudo harden-laravel --force

🔍 Verificando estructura de Laravel...
✅ Estructura de Laravel detectada correctamente

🛡️ Configuración de Endurecimiento de Seguridad
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📁 Ruta completa:  /var/www/mi-aplicacion
👤 Propietario:    usuario
🌐 Usuario Web:    www-data
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  Este script modificará los permisos de TODOS los archivos en:
   /var/www/mi-aplicacion

Los cambios que se aplicarán:
  • Propietario: usuario:www-data
  • Directorios: 755 (rwxr-xr-x)
  • Archivos: 644 (rw-r--r--)
  • storage/: 775 (rwxrwxr-x)
  • bootstrap/cache/: 775 (rwxrwxr-x)
  • .env: 640 (rw-r-----)

✅ Modo --force activado. Procediendo sin confirmación...

👤 Ajustando propietarios a usuario:www-data...
🔒 Aplicando permisos 755/644 (Solo lectura para el servidor web)...
📂 Otorgando permisos de escritura solo en storage y cache...
🔑 Asegurando archivo .env...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Proceso de permisos completado. App asegurada a nivel de sistema.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
...
```

#### Caso 3: Directorio que NO parece Laravel

```text
$ cd /home/usuario/temporal
$ sudo harden-laravel

🔍 Verificando estructura de Laravel...
⚠️  ADVERTENCIA: Este directorio NO parece ser una aplicación Laravel
   Directorios típicos encontrados: 2 de 8
   Directorios faltantes: app config database public resources routes

🛡️ Configuración de Endurecimiento de Seguridad
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📁 Ruta completa:  /home/usuario/temporal
👤 Propietario:    usuario
🌐 Usuario Web:    www-data
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  ADVERTENCIA: Este directorio NO parece ser una aplicación Laravel
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Directorios típicos de Laravel encontrados: 2 de 8
   Directorios faltantes: app config database public resources routes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  Este script modificará los permisos de TODOS los archivos en:
   /home/usuario/temporal

Los cambios que se aplicarán:
  • Propietario: usuario:www-data
  • Directorios: 755 (rwxr-xr-x)
  • Archivos: 644 (rw-r--r--)
  • storage/: 775 (rwxrwxr-x)
  • bootstrap/cache/: 775 (rwxrwxr-x)
  • .env: 640 (rw-r-----)

¿Deseas continuar? (escribe 'si' para confirmar): n

❌ Operación cancelada por el usuario
```

### Notas de Seguridad

- **No ejecutar en desarrollo**: Puede interferir con herramientas como `php artisan`
- **Solo para producción**: Diseñado para entornos de producción
- **Backup recomendado**: Haz backup antes de cambiar permisos masivamente
- **Verificar después**: Comprueba que la aplicación funciona correctamente
- **Usar --force con cuidado**: Asegúrate de estar en el directorio correcto antes de usar `--force`

### Solución de Problemas

#### **Error: Permission denied al escribir logs**

```bash
# Verificar permisos de storage
ls -la /ruta/app/storage
# Debe mostrar 775 y grupo www-data

# Si es necesario, volver a aplicar permisos
sudo harden-laravel /ruta/app --force
```

#### **Error: SQLSTATE[HY000]: attempt to write a readonly database (SQLite)**

Este error ocurre cuando usas SQLite y el archivo de base de datos no tiene permisos de escritura.

```bash
# El script automáticamente detecta y corrige esto, pero si persiste:

# 1. Verificar permisos del archivo SQLite
ls -la /ruta/app/database/*.sqlite
# Debe mostrar 664 (rw-rw-r--) y grupo www-data

# 2. Si no es correcto, ejecutar el script de nuevo
sudo harden-laravel /ruta/app --force

# 3. O corregir manualmente
sudo chmod 664 /ruta/app/database/database.sqlite
sudo chmod 775 /ruta/app/database
sudo chown usuario:www-data /ruta/app/database/database.sqlite
```

**¿Por qué SQLite necesita permisos especiales?**

- SQLite escribe directamente en el archivo de base de datos
- El servidor web (www-data) necesita permisos de escritura en el archivo
- También necesita permisos de escritura en el directorio (para archivos temporales y locks)
- Por eso se usa `664` en archivos y `775` en el directorio que los contiene

#### **Error: .env no se puede leer**

```bash
# Verificar permisos del .env
ls -la /ruta/app/.env
# Debe mostrar 640 y grupo www-data

# Verificar que PHP-FPM corre como www-data
ps aux | grep php-fpm
```

#### **La aplicación no funciona después del script**

```bash
# Verificar que el usuario web está en el grupo correcto
groups www-data

# Verificar logs de Laravel
sudo tail -f /ruta/app/storage/logs/laravel.log

# Verificar logs de Nginx
sudo tail -f /var/log/nginx/error.log

# Revertir permisos si es necesario (solo en emergencia)
sudo chmod -R 775 /ruta/app/storage
sudo chmod -R 775 /ruta/app/bootstrap/cache
```

#### **Nginx retorna 403 después de aplicar las reglas**

```bash
# Verificar que index.php existe
ls -la /ruta/app/public/index.php

# Verificar configuración de Nginx
sudo nginx -t

# Revisar logs de Nginx
sudo tail -f /var/log/nginx/error.log

# Verificar que la ruta 'root' en Nginx apunta a /public
# Debe ser: root /var/www/tu-app/public;
```

#### **El script no encuentra el directorio**

```bash
# Usar ruta absoluta
sudo harden-laravel /var/www/myapp

# O navegar al directorio primero
cd /var/www/myapp
sudo harden-laravel
```

### Preguntas Frecuentes

**¿Puedo usar este script en desarrollo local?**

No es recomendable. Este script está diseñado para entornos de producción. En desarrollo, necesitas permisos más permisivos para que herramientas como `php artisan` funcionen correctamente.

**¿Qué pasa si ejecuto el script dos veces?**

No hay problema. El script es idempotente, puedes ejecutarlo múltiples veces sin causar daños. Simplemente restablecerá los permisos a los valores correctos.

**¿Funciona con Apache en lugar de Nginx?**

Sí, los permisos de filesystem funcionan igual. Sin embargo, las reglas de seguridad web mostradas al final son específicas de Nginx. Para Apache, necesitarías configurar equivalentes en `.htaccess` o en la configuración del VirtualHost.

**¿Puedo personalizar los permisos?**

Los permisos están hardcodeados porque representan las mejores prácticas de seguridad para Laravel. Si necesitas permisos diferentes, tendrías que modificar el script.

**¿El parámetro --force es seguro?**

Es seguro si lo usas correctamente. Asegúrate siempre de:

- Estar en el directorio correcto antes de ejecutarlo
- Verificar la ruta con `pwd` o especificarla explícitamente
- Usarlo solo en scripts automatizados donde confías en el contexto de ejecución
-
