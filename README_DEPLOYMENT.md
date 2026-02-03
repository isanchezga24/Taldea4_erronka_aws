# Guía de Despliegue en AWS EC2

Este documento describe los pasos exactos para desplegar la aplicación Laravel+React+Inertia en una instancia EC2 de AWS.

## Cambios Realizados en Este Fork

- **`vite.config.ts`**: Cambiado `input` de `resources/js/app.tsx` a `resources/js/index.jsx` (entrada correcta)
- Eliminada referencia a `ssr: 'resources/js/ssr.tsx'` (archivo inexistente)
- Ajustado HMR host a `localhost` (en lugar de IP específica)
- **`resources/js/index.jsx`**: Mejorado el resolver de Inertia con `import.meta.glob` y `eager: true`

## Pasos para Desplegar en EC2

### 1. Preparación de la Instancia EC2

Conecta por SSH a tu instancia Ubuntu:

```bash
ssh -i "tu-key.pem" ubuntu@TU_IP_PUBLICA
```

### 2. Instalar Dependencias Base

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx git unzip curl build-essential
sudo apt install -y php-fpm php-cli php-mbstring php-xml php-curl php-zip php-mysql php-bcmath
```

### 3. Instalar Composer y Node.js

**Composer:**
```bash
php -r "copy('https://getcomposer.org/installer','composer-setup.php');"
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
php -r "unlink('composer-setup.php');"
```

**Node.js (versión 18+):**
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

### 4. Clonar el Repositorio

```bash
sudo mkdir -p /var/www/taldea4
sudo chown $USER:$USER /var/www/taldea4
git clone https://github.com/isanchezga24/Taldea4_erronka_aws.git /var/www/taldea4
cd /var/www/taldea4
```

### 5. Instalar Dependencias PHP y Node

```bash
composer install --no-dev --optimize-autoloader
cp .env.example .env
php artisan key:generate

npm ci
npm run build
```

### 6. Configurar Archivo `.env`

Edita `/var/www/taldea4/.env` con tus valores:

```bash
nano /var/www/taldea4/.env
```

Campos críticos:
- `APP_URL=http://TU_IP_PUBLICA` (o tu dominio)
- `DB_HOST=` (dirección del servidor de BD)
- `DB_DATABASE=` (nombre de la BD)
- `DB_USERNAME=` (usuario de BD)
- `DB_PASSWORD=` (contraseña de BD)

### 7. Migraciones y Caché

```bash
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan storage:link
```

### 8. Ajustar Permisos

```bash
sudo chown -R www-data:www-data /var/www/taldea4
sudo find /var/www/taldea4 -type f -exec chmod 644 {} \;
sudo find /var/www/taldea4 -type d -exec chmod 755 {} \;
```

### 9. Configurar Nginx

Crea `/etc/nginx/sites-available/taldea4`:

```bash
sudo nano /etc/nginx/sites-available/taldea4
```

Contenido:

```nginx
server {
    listen 80;
    server_name TU_IP_PUBLICA;

    root /var/www/taldea4/public;
    index index.php index.html;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Activa la configuración:

```bash
sudo ln -s /etc/nginx/sites-available/taldea4 /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 10. Reiniciar Servicios

```bash
sudo systemctl restart php8.1-fpm
sudo systemctl restart nginx
sudo systemctl enable nginx php8.1-fpm
```

### 11. Verificar el Despliegue

Accede a `http://TU_IP_PUBLICA` en el navegador.

Si hay problemas, revisa los logs:

```bash
# Logs de Laravel
sudo tail -n 200 /var/www/taldea4/storage/logs/laravel.log

# Logs de Nginx
sudo tail -n 200 /var/log/nginx/error.log
sudo tail -n 200 /var/log/nginx/access.log

# Logs de PHP-FPM
sudo journalctl -u php8.1-fpm -n 200
```

## Solución de Problemas

### Página en blanco (500 error)
- Revisa `storage/logs/laravel.log` primero
- Verifica permisos de `storage/` y `bootstrap/cache/`
- Comprueba que las extensiones PHP están instaladas

### Assets no cargan (404)
- Verifica que `npm run build` fue ejecutado sin errores
- Comprueba que `public/build/` existe con contenido
- Revisa que `public/manifest.json` está actualizado

### Error de conexión a BD
- Verifica credenciales en `.env`
- Comprueba que la BD está accesible desde EC2
- Si usas RDS, abre puertos en el Security Group

### HMR/Hot Module Reload no funciona
- Esto no es crítico en producción
- En desarrollo local, asegúrate que el puerto 5173 está abierto

## Notas Importantes

- **No ejecutes `npm run dev`** en producción; usa solo `npm run build`
- **Siempre** ejecuta migraciones con `--force` en producción
- **Backup** de BD antes de migraciones
- Usa un **dominio SSL** cuando sea posible (Let's Encrypt + Certbot)

---

¿Necesitas ayuda? Revisa los logs mencionados arriba.
