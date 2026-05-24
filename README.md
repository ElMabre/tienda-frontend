# Frontend — Tienda Perritos 

Dashboard web desarrollado con React 18 + Vite, contenedorizado con Docker y servido en producción mediante Nginx Alpine. Se despliega automáticamente en AWS EC2 a través de GitHub Actions.

---

## Tecnologías

| Componente | Detalle |
|---|---|
| React | 18 |
| Bundler | Vite |
| Estilos | Tailwind CSS |
| Servidor de producción | Nginx Alpine |
| Cliente HTTP | Axios |

---

## Dockerfile Multi-stage

El contenedor de producción se construye en dos etapas:

```dockerfile
# Stage 1: compilación del código estático
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: servidor web de producción
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

La etapa `builder` compila la aplicación y descarta todo lo que no es necesario en producción (node_modules, código fuente, herramientas de build). Al servidor Nginx solo llega la carpeta `dist`, lo que reduce el tamaño final de la imagen y la superficie de ataque del contenedor en la subred pública de AWS.

---

## Levantar en local

```bash
# Construir la imagen
docker build -t tienda-frontend-local .

# Levantar el contenedor
docker run -d -p 80:80 --name contenedor-frontend tienda-frontend-local
```

Luego abrir `http://localhost` en el navegador.

---

## Arquitectura de red y comunicación con el Backend

El Frontend consume dos APIs del backend directamente desde el navegador:

| Servicio | Puerto |
|---|---|
| API Ventas (`TableCompras.jsx`) | `8082` |
| API Despachos (`TableDespachos.jsx`) | `8081` |

El Security Group de la instancia EC2-Frontend expone únicamente el puerto 80 hacia Internet. El tráfico hacia los puertos 8081 y 8082 está restringido según las reglas de la subred privada.

**Nota sobre las IPs en AWS Academy:** Como el Learner Lab reasigna IPs públicas en cada reinicio, el endpoint del backend se configura directamente en los servicios de React antes de generar el build, evitando errores de variables no resueltas en tiempo de compilación.

---

## Pipeline CI/CD

El archivo `.github/workflows/deploy.yml` se activa con cada `push` a la rama `deploy`.

**Flujo:**

1. Checkout del código y autenticación en AWS usando los secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN`.
2. Build de la imagen Docker (compilación de React + empaquetado con Nginx).
3. Push de la imagen al repositorio privado en Amazon ECR bajo el tag `frontend-latest`.
4. Despliegue remoto vía AWS SSM: la instancia EC2-Frontend descarga la nueva imagen, detiene el contenedor anterior y levanta la versión actualizada.
