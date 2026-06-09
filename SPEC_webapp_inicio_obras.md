# SPEC — Webapp "Generador de Correos de Inicio de Obras"
**Constructora Raíces / ATPK / Atipika**
Documento de instrucciones para Claude Code. Versión 1.0 — 09-06-2026

---

## 1. OBJETIVO

Construir una webapp sencilla tipo **wizard (asistente paso a paso)** que:

1. Solicite punto a punto la información de una obra nueva (datos esenciales + temáticas anexas).
2. Permita **avanzar y retroceder** libremente entre pasos, incluso si falta información (los campos vacíos se marcan como `[PENDIENTE]` en el texto final, nunca bloquean).
3. Al finalizar, **genere el texto completo del correo** (asunto + cuerpo + lista de destinatarios) listo para **copiar con un botón** y pegar directamente en Gmail.
4. Tenga una **sección de Configuración** donde el usuario pueda agregar, editar, desactivar o eliminar temáticas (líneas fijas y preguntas anexas) por área, y editar la lista de destinatarios, para mejorar el formato tipo con el tiempo **sin tocar código**.

El usuario final es el Coordinador de Obras (un solo usuario, uso interno). No requiere backend ni autenticación.

---

## 2. STACK Y RESTRICCIONES TÉCNICAS

- **Una sola página**: HTML + CSS + JavaScript vanilla en un único archivo `index.html` (o, si se prefiere, un proyecto Next.js mínimo de una sola ruta; el single-file es la opción preferida por simplicidad de despliegue).
- **Persistencia: `localStorage`** para (a) la configuración (catálogo de temáticas, líneas fijas, destinatarios) y (b) el borrador en curso (autosave en cada cambio, para no perder lo escrito si se cierra la pestaña).
- **Exportar/Importar configuración** como archivo JSON (botones en Configuración), para respaldo y para compartir el formato entre computadores.
- Sin dependencias externas obligatorias. Si se quiere estética rápida, CSS propio simple; nada de frameworks pesados.
- Idioma de toda la interfaz: **español de Chile**.
- Debe funcionar bien en desktop; responsivo básico para mobile es deseable pero secundario.
- Botón "Copiar" usa `navigator.clipboard.writeText()` con fallback a selección de textarea.

---

## 3. ARQUITECTURA DE LA INFORMACIÓN

La app maneja dos tipos de contenido, y esta distinción es la regla de oro de todo el diseño:

- **CAMPOS ESENCIALES**: datos duros que cambian obra a obra (nombre, código Manager, fechas, responsables...). Son inputs simples. Si quedan vacíos → el texto final muestra `[PENDIENTE: nombre del campo]` en rojo en la vista previa.
- **TEMÁTICAS** (configurables): cada temática pertenece a un ÁREA y es de uno de dos tipos:
  - `fija`: línea de procedimiento estándar que se incluye SIEMPRE en el correo (texto editable solo desde Configuración). En el wizard solo se muestran como lista informativa con checkbox "incluir" (pre-marcado).
  - `anexa`: pregunta condicional. En el wizard se presenta como **Sí / No / No aplica**. Si la respuesta es Sí, se despliega un campo de texto para detallar, con un texto de ayuda (placeholder) que muestra un ejemplo histórico real. Solo las respondidas "Sí" con detalle aparecen en el correo.

---

## 4. MODELO DE DATOS

Guardar en `localStorage` bajo dos claves: `iniciObras_config` y `iniciObras_borrador`.

```javascript
// ===== CONFIG (editable en sección Configuración) =====
config = {
  version: 1,
  empresas: ["Constructora Raíces", "ATPK", "Atipika"],
  destinatarios: {
    // listas por empresa; el wizard concatena la lista de la empresa elegida
    "Constructora Raíces": [
      "cespinoza@scraices.cl", "aespinoza@scraices.cl", "mabuid@scraices.cl",
      "rlagos@scraices.cl", "autocontrol@scraices.cl", "amartinez@scraices.cl",
      "mguinez@scraices.cl", "csaravia@scraices.cl", "vnahuel@scraices.cl",
      "vecheverria@scraices.cl", "adquisiciones@scraices.cl", "ebodega@scraices.cl",
      "Nbascur@scraices.cl", "nmunoz@scraices.cl", "rrhh@scraices.cl",
      "prevencion@scraices.cl"
    ],
    "ATPK": [ /* misma base, rrhh@atpk.cl en vez de rrhh@scraices.cl */ ],
    "Atipika": [ /* misma base */ ]
  },
  areas: [
    { id: "autocontrol", nombre: "Autocontrol / Residente" },
    { id: "egr",         nombre: "EGR – Legal" },
    { id: "proyectos",   nombre: "Área Proyectos / Residente" },
    { id: "logistica",   nombre: "Logística / Despacho" },
    { id: "admin",       nombre: "Administración / RRHH / Prevención de Riesgos" },
    { id: "contabilidad",nombre: "Consideración Contabilidad" },
    { id: "generales",   nombre: "Generales" }
  ],
  tematicas: [
    // { id, areaId, tipo: "fija"|"anexa", activa: true,
    //   texto (para fijas) | pregunta + placeholder (para anexas), orden }
    // ===== SEED COMPLETO EN SECCIÓN 8 — USAR TAL CUAL =====
  ]
}

// ===== BORRADOR (autosave del wizard en curso) =====
borrador = {
  esenciales: {
    empresa: "", nombreObra: "", comuna: "", codigoManager: "",
    fechaActaTerreno: "", fechaInicioFisico: "", plazoDias: null,
    fechaTermino: "",          // autocalculada: fechaInicioFisico + plazoDias (editable)
    tipologias: "", bodegaRC: "", totalGrupo: "",
    residente: "", autocontrol: "", capataz: "",
    asistenteSocial: "", fto: "", sat: "",
    linkGantt: ""
  },
  fijasIncluidas: { /* tematicaId: true/false (default true) */ },
  anexas: { /* tematicaId: { respuesta: "si"|"no"|"na"|null, detalle: "" } */ },
  pasoActual: 0
}
```

---

## 5. FLUJO DEL WIZARD (PANTALLAS)

Navegación: barra de progreso superior con los pasos clicables (se puede saltar a cualquier paso), botones **« Atrás** y **Siguiente »** abajo. NUNCA bloquear el avance por campos vacíos; en cambio, mostrar un contador discreto de pendientes (ej. "3 campos pendientes") clicable que lista qué falta y lleva al paso correspondiente.

**Paso 0 — Inicio**
- Botones: "Nueva obra" (limpia borrador) / "Continuar borrador" (si existe) / "Configuración".

**Paso 1 — Identificación**
- Empresa (select desde config), Nombre de la obra, Comuna, Código Manager.
- Vista previa en vivo del ASUNTO generado bajo los campos.

**Paso 2 — Fechas y plazos**
- Fecha acta de entrega de terreno (date), Fecha inicio físico (date), Plazo en días (number).
- Fecha de término: se autocalcula (inicio + plazo) y se muestra editable.

**Paso 3 — Alcance**
- Tipologías (text), Bodega/RC (text), Total del grupo (text).

**Paso 4 — Responsables**
- Residente, Autocontrol, Capataz en terreno, Asistente Social, FTO, SAT (6 inputs).

**Paso 5 — Cronograma**
- Link a Carta Gantt (url).

**Pasos 6 a 6+N — Un paso por ÁREA (en el orden de config.areas)**
Cada paso de área tiene dos secciones:
1. **"Lineamientos estándar"**: lista de temáticas `fijas` activas del área, cada una con checkbox marcado por defecto (desmarcar = no incluir en este correo puntual).
2. **"Consultas específicas de esta obra"**: las temáticas `anexas` activas del área, presentadas una a una como tarjetas con botones [Sí] [No] [No aplica]. Al elegir Sí se despliega textarea con placeholder de ejemplo. Las no respondidas cuentan como pendientes "suaves" (se listan en el contador, pero si quedan sin responder simplemente se omiten del correo, sin marca PENDIENTE).

**Paso final — Revisión y generación**
- Resumen de pendientes esenciales (si hay), con links a cada paso.
- **Vista previa del correo completo** (ver formato exacto en sección 6), con los `[PENDIENTE: ...]` destacados en rojo.
- Tres bloques copiables por separado, cada uno con su botón "Copiar":
  1. **PARA:** lista de destinatarios separada por coma
  2. **ASUNTO**
  3. **CUERPO** (texto plano con la estructura de la sección 6)
- Botón "Copiar todo" (concatena los tres con encabezados).
- Botón "Finalizar y limpiar borrador".

---

## 6. FORMATO EXACTO DEL TEXTO GENERADO

El cuerpo se genera en **texto plano** (se pega en Gmail y se formatea a mano si se quiere). Reglas:
- Campos vacíos → `[PENDIENTE: <etiqueta>]`.
- Áreas sin ninguna línea (todas las fijas desmarcadas y ninguna anexa en Sí) → se omite el encabezado del área completo.
- Las anexas con respuesta Sí se insertan al final de la sección de su área, con el detalle escrito por el usuario.

```
ASUNTO:
INICIO OBRA {NOMBRE_OBRA en mayúsculas} – {COMUNA} / Código Manager: {CODIGO} / Fecha Inicio: {FECHA_INICIO dd-mm-aaaa} / TODOS LEER COMPLETO

CUERPO:
Estimados,

En relación con el inicio de la obra {Nombre Obra}, Código Manager: {Código}, se informa lo siguiente:

- Fecha de inicio: Acta de entrega de terreno: {fecha acta}; INICIO OBRAS: {fecha inicio físico} (corresponde a la fecha de inicio físico de la obra).
- Plazo de ejecución: {plazo} días, fecha de término {fecha término}.
- Tipología: {tipologías}. Bodega: {bodega/RC}.
- Total grupo: {total grupo}.
- Cronograma: se adjunta cronograma de inicio interno y plan de trabajo. Debe ser revisado por todas las áreas para ajustar sus funciones a esta planificación.
  {link Gantt}

Responsables:
- Residente de obra: {residente}
- Autocontrol: {autocontrol}
- Capataz en terreno: {capataz}
- Asistente Social: {asistente social}
- FTO: {fto}
- SAT: {sat}

Lineamientos de trabajo:

{POR CADA ÁREA con contenido:}
{Nombre del Área}:
- {línea fija incluida 1}
- {línea fija incluida 2}
- {detalle de anexa respondida Sí, tal como lo escribió el usuario}
...

Quedo atento a cualquier comentario o rectificación a la información entregada. Se solicita acusar recibo.

Saludos,
Alfredo Espinoza
Coordinador de Obras
```

---

## 7. SECCIÓN CONFIGURACIÓN

Accesible desde el Paso 0 y desde un ícono ⚙ permanente. Contiene tres pestañas:

**7.1 Temáticas**
- Tabla/lista agrupada por área. Cada fila muestra: tipo (fija/anexa), texto o pregunta, switch Activa/Inactiva, botones Editar y Eliminar, drag o flechas para reordenar.
- Botón "+ Agregar temática": formulario con Área (select), Tipo (fija/anexa), Texto (si fija) o Pregunta + Placeholder de ejemplo (si anexa).
- Desactivar ≠ eliminar: una temática inactiva no aparece en el wizard pero se conserva.
- También permitir agregar/renombrar/eliminar ÁREAS.

**7.2 Destinatarios**
- Por empresa: textarea editable con un correo por línea. Validación visual simple de formato email.

**7.3 Respaldo**
- "Exportar configuración" → descarga `config_inicio_obras.json`.
- "Importar configuración" → file input, valida `version`, reemplaza config previa confirmación.
- "Restaurar valores de fábrica" → recarga el SEED de la sección 8 previa confirmación.

---

## 8. SEED DE TEMÁTICAS (CARGAR TAL CUAL EN PRIMERA EJECUCIÓN)

Extraído del análisis de los correos históricos reales (Ñuke Mapu 17-2025 y El Maitén 5-2026, principalmente). Los `placeholder` de las anexas son ejemplos reales para guiar la redacción.

### Área: Autocontrol / Residente (`autocontrol`)
**Fijas:**
1. Construcción de planilla de m³ para revisión y envío a FTO. Confirmar que ha sido derivada.
2. Levantar requerimientos especiales de materiales, herramientas u otros.
3. Planilla de MO: se considera trabajar con cuadrillas propias; en caso de eventual subcontrato, confirmar cómo se procederá en el pago y solicitar contrato de subcontrato y seguimiento de responsabilidades según normativa y formato, con especial cuidado en calidad.
4. Actualizar pre-levantamientos y comentarios en la App para requerir habilitaciones, cumpliendo procedimientos de revisión y registro (rutas, emplazamientos, observaciones técnicas).
5. Establecer calendario inmediato de toma de muestras de hormigones.
6. Actualizar libro de obras en relación con homologaciones y cambios relevantes del proyecto.

**Anexas:**
1. ¿Hay temas eléctricos críticos (presupuestos vs factibilidades, empalmes, normativa nueva)?
   placeholder: "CRÍTICO: Verificar de forma detallada los presupuestos eléctricos y factibilidades, contrastando ambas informaciones antes de solicitar habilitaciones de terreno. Coordinar visitas con [instalador] para evaluación de conectividad. Aplica normativa eléctrica nueva."
2. ¿Hay cambios de estructura/prefabricación que el residente deba dominar?
   placeholder: "Énfasis en cambio de estructura de bodega a acero, prefabricación en planta. Cambio de receta cuando se envíe ficha de paneles para creación de códigos Manager. Aclarar dudas constructivas de piloto casa+bodega."
3. ¿Hay pre-verificaciones pendientes de terminar?
   placeholder: "Terminar levantamiento de pre-verificación para revisión."

### Área: EGR – Legal (`egr`)
**Fijas:**
1. Seguimiento de inscripción de prohibiciones y toma de firmas (informar casos pendientes).
2. Subida de comentarios relevantes a la App. Todo comentario debe quedar con respuesta, subsanación o continuidad de gestiones.
3. Subir DJ firmadas.

**Anexas:**
1. ¿Hay casos legales o beneficiarios críticos a individualizar?
   placeholder: "Revisar comentarios legales y casos críticos a resolver: [nombres], casos con subdivisiones; dejar puntos aclarados en actas de entrega de terreno."
2. ¿Corresponde calendarizar entregas individuales de terreno?
   placeholder: "Calendarizar entregas individuales según viviendas que cuentan con observaciones de pre-verificación subsanadas."
3. ¿Está pendiente la reunión de selección de color y cerámica con el grupo?
   placeholder: "Reunión de selección de color y cerámica. URGENTE."

### Área: Área Proyectos / Residente (`proyectos`)
**Fijas:**
1. Actualización de planimetría en Drive (planos homologados).
2. Actualización de información técnica en la App.
3. Revisión de tipologías contra la calificación definitiva de los proyectos.
4. Cumplimiento de checklist de obras según hitos mínimos (instalación de letreros, cronogramas derivados a SERVIU, actualización de libro de obras).
5. Aclaraciones sobre pernos de anclaje, proyecto eléctrico, hojalatería y otras observaciones técnicas; las anotaciones deben quedar en el libro de obras.

**Anexas:**
1. ¿Hay factibilidades por actualizar (APR, eléctricas)?
   placeholder: "Actualización de factibilidades de agua en APR y eléctricas; verificar en factibilidades la ubicación física del poste de conexión versus presupuesto. Actualizar resoluciones de APR del SS pendientes."
2. ¿Hay proyectos de cambio (fachada, estructura de bodega) por entregar?
   placeholder: "Entrega de proyectos para cambio de fachada y estructura de bodega."

### Área: Logística / Despacho (`logistica`)
**Fijas:**
1. Inicio del cronograma de compras basado en recetas, priorizando materiales de fundaciones y de largo período de fabricación (ventanas, bodegas prefabricadas y cubiertas).
2. Receta definitiva cargada en la App. Una vez subidas, es responsabilidad de los residentes verificar sus recetas e informar ajustes o cambios.
3. Preparar anticipado mediante kit bajo receta (fundaciones, plantas, redes de agua), avanzando compras según cronograma informado.
4. Crear carpeta con certificados de materiales de proveedores.

**Anexas:**
1. ¿Hay cambios de receta, colores u otras decisiones de diseño que informar?
   placeholder: "IMPORTANTE: se unificó un solo color de vivienda según fotografía (GRIS OSCURO), revisar ajuste en receta. / Receta definitiva con cambios en fachada, colores y estructura de bodega antes del [fecha]."
2. ¿Hay fabricación anticipada en curso (estructuras, bodegas prefabricadas)?
   placeholder: "Estructuras de acero: 10 fabricadas. Bodegas se trabajarán prefabricadas en Planta Raíces, modelo 16 m²; código nuevo en Manager debe crearse por panel (confirmación de planta)."
3. ¿Hay cotizaciones u OC específicas que gatillar de inmediato?
   placeholder: "Generar cotizaciones y OC para [proveedor] por viviendas modelos [tipologías]."

### Área: Administración / RRHH / Prevención de Riesgos (`admin`)
**Fijas:**
1. Establecer el grupo F-30 y las geocercas para Surtrack del personal en terreno.
2. Generar capacitaciones, inducciones y entrega de implementos y vehículos según lo requerido.
3. Crear la obra como centro de costo en REX+.
4. Crear centros de trabajo para informes de RRHH y Prevención de Riesgos.
5. Preparar y entregar el archivador administrativo para seguimiento.
6. Actualización de contratos con anexos por obra nueva; el residente debe informar movimientos de personal.
7. Entrega de formatos de subcontrato (en caso de corresponder) y procedimiento de subcontratistas al residente de obras.
8. Revisar seguros asociados a la obra y entregar a EGR.

**Anexas:**
1. ¿Hay campañas o verificaciones puntuales de Prevención/RRHH?
   placeholder: "VERIFICAR LA ACTUALIZACIÓN DE ENTREGAS DE EPP A CUADRILLAS PARA TRABAJO EN ALTURA."
2. ¿Se aprovechará la instancia para regularizaciones pendientes?
   placeholder: "Aprovechar esta instancia para actualizaciones de contratos pendientes, según actualización de Reglamento Interno y normativos."

### Área: Consideración Contabilidad (`contabilidad`)
**Fijas:** (ninguna por defecto)
**Anexas:**
1. ¿Hay factor de facturación u otra consideración contable que indicar?
   placeholder: "Facturación 2025 – 31 de diciembre de 2026: aplica 3,0785%."
2. ¿La empresa ejecutora es distinta de Raíces (separación de gastos)?
   placeholder: "Primera obra de la empresa [ATPK/Atipika]; por ende, contrataciones, suministros y gastos generales deben estar claramente separados."

### Área: Generales (`generales`)
**Fijas:**
1. Todos los cambios de material, cantidad o tipología deben canalizarse a través de la jefatura directa. Actualizar acuerdos en relación con la estructura del proyecto y dejar anotaciones en comentarios (cambios de presupuestos, presupuestos adicionales individuales).
2. Enviar solicitudes aprobadas de cambios o ajustes a ebodega@scraices.cl y adquisiciones@scraices.cl, con copia a aespinoza@scraices.cl, para su actualización en la App e informar al equipo de adquisiciones y despacho.
3. No se aceptarán cambios fuera de este procedimiento, para evitar problemas logísticos y contables.
4. Cualquier duda respecto a procesos internos y responsabilidades, aclarar por este medio. Se pide a coordinadores de áreas reforzar los procedimientos, fechas y medios de entrega de información.

**Anexas:**
1. ¿Hay pendientes conocidos que condicionan el inicio?
   placeholder: "Se reitera que están pendientes procesos importantes como cierre de receta completa para piloto, revisión de [tema], antes del inicio físico."
2. ¿Hay piloto definido?
   placeholder: "Piloto: estructura de acero, casa + bodega. Se inicia por viviendas piloto [nombres/ubicación]."

---

## 9. CRITERIOS DE ACEPTACIÓN

1. Puedo crear una obra nueva, llenar solo 4 de los 9 campos esenciales, saltar al paso final y obtener el correo con `[PENDIENTE: ...]` en los 5 restantes; luego retroceder, completarlos y regenerar.
2. Puedo responder "Sí" a una anexa, escribir detalle, y verla insertada en la sección de su área en el texto final; las respondidas "No"/"No aplica"/sin responder no aparecen.
3. Puedo desmarcar una línea fija en el wizard y desaparece solo de ese correo (la config no cambia).
4. En Configuración puedo: agregar una temática anexa nueva a Logística, desactivar una fija de Admin, reordenar, editar destinatarios — y el próximo correo refleja todo.
5. Cierro la pestaña a mitad de llenado, vuelvo a abrir, y "Continuar borrador" restaura todo, incluido el paso en que iba.
6. Exporto la config a JSON, restauro fábrica, importo el JSON, y recupero mis cambios.
7. El botón Copiar deja en el portapapeles exactamente el texto mostrado en la vista previa (sin HTML).
8. La fecha de término se autocalcula al ingresar inicio + plazo, en formato dd-mm-aaaa, y sigue siendo editable.

---

## 10. NOTAS DE DISEÑO

- Estética sobria y profesional, paleta neutra con un acento (sugerido: verde Raíces). Tipografía del sistema.
- Las tarjetas de preguntas anexas son el corazón de la UX: una pregunta visible a la vez por tarjeta, grandes botones Sí/No/No aplica, y el placeholder de ejemplo siempre visible como guía de redacción.
- El contador de pendientes debe distinguir: **pendientes esenciales** (rojo, aparecen como [PENDIENTE] en el texto) vs **anexas sin responder** (gris, simplemente se omiten).
- Mostrar en el header el nombre de la obra en curso apenas se escribe, para contexto permanente.
