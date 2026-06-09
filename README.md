# Generador de Correos de Inicio de Obras

Webapp tipo **wizard** para redactar el correo de inicio de obra de **Constructora Raíces / ATPK / Atipika**.

- **Single-file**: todo vive en `index.html` (HTML + CSS + JS vanilla, sin dependencias).
- **Persistencia local**: usa `localStorage` (no hay backend ni base de datos). La configuración
  (`iniciObras_config`) y el borrador en curso (`iniciObras_borrador`) quedan en el navegador del usuario.
- **Privacidad**: ningún dato sale del computador del usuario.

## Uso

1. Abre la app (local: doble clic en `index.html`; en línea: la URL de Vercel).
2. **Nueva obra** → completa el asistente paso a paso. Puedes saltar entre pasos; los campos
   vacíos salen como `[PENDIENTE: …]` en el correo, nunca bloquean.
3. En el **paso final** copia los bloques (PARA / ASUNTO / CUERPO) o "Copiar todo" y pégalos en Gmail.
4. **⚙ Configuración** permite editar temáticas, áreas y destinatarios sin tocar código,
   y exportar/importar la configuración como JSON para respaldo o para compartir el formato.

## Despliegue

Sitio estático en **Vercel** (build no requerido). El repositorio está conectado a Vercel:
cada `git push` a `main` genera un nuevo despliegue de producción automáticamente.

```bash
# despliegue manual opcional
vercel --prod
```

## Estructura

| Archivo        | Rol                                            |
|----------------|------------------------------------------------|
| `index.html`   | La aplicación completa.                         |
| `vercel.json`  | Config de hosting estático (headers, cleanUrls).|
| `README.md`    | Este archivo.                                   |

Especificación original: `SPEC_webapp_inicio_obras.md`.
