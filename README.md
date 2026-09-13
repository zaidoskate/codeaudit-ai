# CodeAudit AI

**CodeAudit AI** es una plataforma de análisis de código de dos capas (estático local + LLM asíncrono) diseñada bajo restricciones de recursos gratuitos, exponiendo sus funcionalidades mediante una CLI en Go y un Dashboard en Next.js.

## Estructura del Monorepo

- `/backend`: Capa API, dominio, integración con proveedores LLM y workers (Go).
- `/cli`: Herramienta de línea de comandos (Go).
- `/frontend`: Dashboard de usuario (Next.js, App Router, Tailwind, Shadcn).
- `/docs`: Documentación y diseño de arquitectura.

## CI/CD

El repositorio está configurado para integración continua vía GitHub Actions, con despliegue automatizado hacia Render (backend) y Vercel (frontend) desde la rama `main`.