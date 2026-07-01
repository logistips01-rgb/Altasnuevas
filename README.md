# Altas Nuevas · Control y Verificación

App interna de Aldelís para controlar el proceso de alta de nuevas referencias en
producción. Añade una capa de verificación física (stock real de bandeja, film,
etiqueta, caja, envase logístico, pruebas de impresión, alérgenos en etiqueta
impresa...) que el ERP actual no cubre, y da visibilidad tipo semáforo sobre
qué altas están en riesgo de fallar en el arranque.

Single-file, sin build step: HTML + CSS + JS vanilla, Firebase como backend
(Firestore + Auth + Hosting). Mismo patrón que otras apps internas de Aldelís
(ej. Aldelís Muelles).

## Archivos

- `public/altas-tracker.html` — la app completa.
- `firestore.rules` — reglas de seguridad de Firestore.
- `firestore.indexes.json` — índices (vacío; la app no necesita ninguno compuesto hoy).
- `firebase.json` / `.firebaserc` — configuración de despliegue.

## Puesta en marcha

1. Crea un proyecto de Firebase (o reutiliza uno existente, ver más abajo) con
   **Authentication** (método Email/contraseña) y **Firestore** habilitados.
2. Sustituye el bloque `firebaseConfig` al principio del `<script>` en
   `public/altas-tracker.html` por la config real de tu proyecto (Configuración
   del proyecto → General → tus apps → SDK setup and configuration).
3. Sustituye `TU_PROYECTO` en `.firebaserc` por el ID real del proyecto.
4. Crea manualmente los usuarios que necesiten acceso en Firebase Authentication
   (no hay registro público, solo login).
5. Despliega:

   ```bash
   npm install -g firebase-tools   # si no lo tienes
   firebase login
   firebase deploy --only firestore:rules,hosting
   ```

### Reutilizar el proyecto de Aldelís Muelles

Si prefieres no crear un proyecto Firebase nuevo, puedes usar el mismo proyecto
que `aldelis-muelles.web.app`:

- Esta app usa colecciones propias (`altas_tracker`, `stock_general`) que no
  chocan con las de Muelles.
- **Importante:** `firestore.rules` de este repo solo contiene las reglas de
  esta app y termina con un `match /{document=**} { allow read, write: if
  false; }` de cierre. Firestore solo permite un archivo de reglas activo por
  proyecto, así que si compartes proyecto con Muelles tendrás que **fusionar**
  los bloques `match /altas_tracker/{...}` y `match /stock_general/{...}` de
  aquí dentro del `firestore.rules` ya desplegado para Muelles, en vez de
  desplegar este archivo tal cual (o se romperían las reglas existentes de
  Muelles).
- Para el hosting, copia `public/altas-tracker.html` a la carpeta `public/`
  del proyecto de Muelles y haz `firebase deploy --only hosting` desde allí.

## Modelo de datos

### Colección `altas_tracker`

Un documento por alta nueva:

```
{
  ref: string,
  cliente: string,
  fecha_produccion_prevista: string (YYYY-MM-DD) | null,
  archivada: boolean,
  creado_por: string (email),
  creado_en: ISO string,
  archivada_por: string (email) | undefined,
  archivada_en: ISO string | undefined,

  comercial: { [campoId]: { valor, quien, cuando } },
  idi:       { [campoId]: { valor, quien, cuando } },
  it:        { [campoId]: { valor, quien, cuando } },
  lab:       { [campoId]: { valor, quien, cuando } },

  ingredientes: [ { nombre: string, porcentaje: number } ],

  verificacion_fisica: {
    [itemId]: {
      estado: "pendiente" | "verificado" | "fallo",
      comentario: string,
      quien: string,
      cuando: ISO string,
      // solo en ítems stockLinked:
      codigo_material: string,
      cantidad_necesaria: number
    }
  }
}
```

Los IDs de campos de cada departamento están en el JS (`CAMPOS_COMERCIAL`,
`CAMPOS_IDI`, `CAMPOS_IT`, `CAMPOS_LAB`), replicando la ficha real del ERP.
Los ítems de verificación física están en `VERIF_ITEMS`.

### Colección `stock_general`

Un documento por material, `doc.id = código`:

```
{
  descripcion: string,
  cantidad: number,
  unidad: string,
  actualizado_en: ISO string,
  actualizado_por: string (email)
}
```

Se sobrescribe por completo en cada importación de Excel (no se guarda
histórico de stock, solo la última foto).

## Qué hace la app hoy

1. Login con Firebase Auth.
2. Dashboard con altas activas ordenadas por urgencia (días hasta
   `fecha_produccion_prevista`), semáforo por tarjeta, contadores agregados y
   barra de progreso de verificación física. Incluye un filtro
   Activas/Archivadas.
3. Ficha de alta con 5 pestañas: Comercial, I+D, IT, Laboratorio (fieles a los
   campos del ERP) y Verificación física (checklist de 9 puntos con
   Pendiente/Verificado/Fallo + comentario; los puntos de stock comparan en
   vivo contra `stock_general`).
4. Vista de Stock general: importación por Excel (columnas Código,
   Descripción, Cantidad, Unidad), con aviso fila a fila si falta el código o
   si la cantidad no es numérica.
5. Semáforo automático: rojo si hay algún "Fallo" o quedan ≤2 días sin
   verificación completa; ámbar si quedan ≤5 días; verde si todo está
   verificado.
6. Archivar / reactivar una alta desde su ficha.
7. Exportar a Excel del dashboard.

## Seguridad

`firestore.rules` exige `request.auth != null` para leer o escribir en
`altas_tracker` y `stock_general`, y deniega todo lo demás por defecto. No hay
todavía separación de permisos por departamento (cualquier usuario logueado
puede editar cualquier pestaña) — ver "Pendiente" más abajo.

## Pendiente / mejoras futuras

- **Roles por departamento:** hoy cualquier usuario autenticado puede editar
  cualquier campo de cualquier pestaña (Comercial, I+D, IT, Laboratorio). Si
  se necesita restringir (ej. que I+D no pueda tocar campos de Comercial),
  habría que mantener una colección `usuarios/{uid}` con el rol de cada
  persona y comprobarlo tanto en las Firestore Rules como en el cliente —
  nunca solo en el cliente, que se puede saltar.
- **Paginación:** el dashboard carga todas las altas (activas y archivadas) de
  golpe vía `onSnapshot`. Bien para el volumen actual; si crece mucho conviene
  paginar o filtrar por rango de fechas.
- **Histórico de stock:** la importación de Excel sobrescribe cada material
  (`batch.set()`), así que solo se conserva la última foto. Si se quiere
  histórico, habría que guardar snapshots en una subcolección en vez de
  sobrescribir el documento.
- **Índices:** ninguna query actual necesita índice compuesto (no se combina
  `where` + `orderBy` contra Firestore; el orden se hace en JS). Si en el
  futuro se mueve el `orderBy` a la query de Firestore, revisar si pide crear
  uno.
