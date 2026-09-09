# Buscador de Imágenes Huérfanas (Pro)

> **AVISO IMPORTANTE.** Este módulo **borra ficheros de la carpeta `img/p/` de la tienda** y **elimina carpetas** de esa misma ruta. Antes de pulsar cualquier botón de la pestaña «3. Resultados y Limpieza» o de la pestaña «4. Limpieza Carpetas», haz una **copia de seguridad completa de `img/p/`** (y, por prudencia, de la base de datos). Lo que se borra no se puede recuperar desde el módulo: no hay papelera ni tabla de respaldo. En la sección 5 se detalla qué borra exactamente cada botón.

## 1. Qué hace y para quién

En cualquier tienda PrestaShop con unos años de vida, la carpeta `img/p/` acaba llena de ficheros que ya no pertenecen a nada: imágenes de productos que se borraron, restos de importaciones que fallaron a medias, miniaturas de tamaños que ya no existen, copias de seguridad restauradas por encima de otra tienda… Todo eso ocupa espacio en el disco, engorda las copias de seguridad y ralentiza cualquier migración de servidor, pero desde el back-office de PrestaShop no hay forma de verlo.

El Buscador de Imágenes Huérfanas recorre **el disco** (la carpeta `img/p/` completa, con todas sus subcarpetas), registra cada fichero de imagen que encuentra y lo contrasta con la base de datos: por el número con el que empieza el nombre del fichero (que en PrestaShop es siempre el `id_image`) comprueba si esa imagen existe en la tabla `ps_image` y, si existe, si el producto al que apunta existe en `ps_product`. Los ficheros que no pasan alguna de las dos comprobaciones se marcan como **huérfanos** y se listan en pantalla, donde se pueden borrar uno a uno o todos de golpe.

Como cuarto paso, incluye una limpieza de **carpetas vacías**: la estructura de PrestaShop (`img/p/1/2/3/`) deja carpetas que solo contienen el `index.php` de seguridad y el fichero `fileType` cuando se borran las imágenes que había dentro, y el módulo las elimina para dejar el árbol limpio.

Todo el trabajo se hace **por lotes desde el navegador** (peticiones AJAX encadenadas), así que sirve también para tiendas con decenas o cientos de miles de imágenes sin que el servidor agote el tiempo de ejecución de PHP. Está pensado para el administrador o el técnico que mantiene la tienda, no para el uso diario: se ejecuta de vez en cuando, con copia de seguridad hecha, y con calma.

### En una línea, frente a `ecom_cleanimages`

Son complementarios y van en sentidos opuestos: **`ecom_cleanimages` parte de la base de datos** (registros de `ps_image` cuyo fichero no existe, y los borra de la base de datos además de regenerar miniaturas), mientras que **este módulo parte del disco** (ficheros de `img/p/` que no tienen registro o cuyo producto no existe, y borra los ficheros). El manual de `ecom_cleanimages` está en `manuales/ecom_cleanimages/`.

## 2. Compatibilidad

- **Ubicación en el repositorio:** está en `old modules\old\`. **Sigue siendo vigente** en cuanto a función: no existe ningún módulo más reciente de Ecom Experts que busque imágenes huérfanas en el sentido disco → base de datos (`ecom_cleanimages` hace el sentido contrario). Sí está **superado en forma**: el autor declarado es «Antigravity» y no «Ecom Experts», la cabecera de los ficheros es la genérica de PrestaShop, las cadenas usan el sistema de traducción antiguo (`mod=`) y no hay ficheros `index.php` en las subcarpetas. Funciona, pero no cumple el estándar actual de los módulos de la casa; si se va a entregar a un cliente conviene revisarlo antes.
- **PrestaShop:** declara compatibilidad de 1.7 en adelante. **Instalado y comprobado en PrestaShop 9.1.1** (capturas de este manual). No usa nada específico de una versión: solo `Module`, `Db`, `Tools` y la constante `_PS_PROD_IMG_DIR_`.
- **PHP:** 7.4 y 8.x. Sin dependencias, sin Composer, sin librerías externas.
- **Multitienda:** las tablas del módulo son globales y la carpeta `img/p/` es común a todas las tiendas, así que el resultado es el mismo desde cualquier contexto de tienda. No hay ninguna opción por tienda.
- **Idiomas:** la interfaz está escrita en castellano directamente en las plantillas y en el JavaScript. No hay ficheros de traducción; el editor de traducciones de PrestaShop no la cambia.
- **Front-office:** no toca nada de la tienda pública. Solo existe la pantalla del back-office.

## 3. Instalación

1. Sube la carpeta `ecom_imagesorfan` a `/modules/` (o instala el ZIP desde **Módulos > Gestor de módulos > Subir un módulo**).
2. Pulsa **Instalar** y después **Configurar**.

Qué crea al instalar:

| Elemento | Detalle |
|---|---|
| Tabla `ps_ecom_imagesorfan_files` | Un registro por fichero de imagen encontrado en `img/p/`: ruta relativa, nombre, `id_image` deducido del nombre y estado (`pending`, `ok`, `orphan_db`, `orphan_product`, `error`). |
| Tabla `ps_ecom_imagesorfan_queue` | Cola de carpetas pendientes de recorrer. Se vacía sola al terminar cada escaneo. |
| Tabla `ps_ecom_imagesorfan_clean_dirs` | Lista de carpetas candidatas a borrar. **No se crea al instalar**, sino la primera vez que se usa la pestaña «4. Limpieza Carpetas». |
| Hook | `actionAdminControllerSetMedia`: solo carga el CSS y el JavaScript del módulo cuando se está en su pantalla. |
| Pestañas de menú | Ninguna. Se accede desde **Módulos > Gestor de módulos > Configurar**. |
| Configuración | Ninguna clave en `ps_configuration`. El módulo no tiene opciones que guardar. |

Al **desinstalar** se eliminan las tablas `ps_ecom_imagesorfan_files` y `ps_ecom_imagesorfan_queue`. La tabla `ps_ecom_imagesorfan_clean_dirs`, si llegó a crearse, **se queda en la base de datos** (el desinstalador no la borra); se puede eliminar a mano sin ningún efecto sobre la tienda.

## 4. Configuración

El módulo **no tiene opciones de configuración**: la pantalla de «Configurar» es directamente el panel de trabajo. A la izquierda hay un menú con los cuatro pasos y un cuadro de estadísticas; a la derecha, el contenido del paso elegido.

![Panel de control del módulo](../../capturas/ecom_imagesorfan/9.1/config-unica.png)

Las estadísticas de la izquierda se leen de la tabla de ficheros del módulo:

| Estadística | Qué cuenta |
|---|---|
| **Total Ficheros** | Ficheros de imagen registrados en el último escaneo. |
| **Pendientes de verificar** | Ficheros registrados que todavía no se han contrastado con la base de datos. |
| **Huérfanas Detectadas** | Ficheros marcados como `orphan_db` o `orphan_product`. |

Los tres números se actualizan al abrir la pestaña «3. Resultados y Limpieza» o al recargar la página; durante el escaneo y la verificación no se refrescan solos.

## 5. Cómo se usa, día a día

El flujo es siempre el mismo y en este orden: **indexar → verificar → revisar resultados → (opcional) borrar → (opcional) limpiar carpetas**. Cada paso es una pestaña del menú de la izquierda.

### Paso 1. Indexar imágenes

Recorre `img/p/` carpeta a carpeta (una carpeta por petición AJAX) y guarda en la tabla del módulo cada fichero con extensión `jpg`, `jpeg`, `png`, `gif` o `webp`. Del nombre del fichero toma el número inicial como `id_image` (`123.jpg`, `123-home_default.jpg` y `123-small_default.webp` apuntan los tres a la imagen 123). Los ficheros cuyo nombre no empieza por un número se registran igualmente, pero después se dan por buenos sin comprobarlos.

| Botón | Efecto |
|---|---|
| **Iniciar Nuevo Escaneo (Reiniciar)** | Pide confirmación, **vacía las tablas del módulo** (no toca nada de la tienda) y empieza a recorrer `img/p/` desde la raíz. Mientras trabaja muestra una barra de progreso y un registro con cada carpeta procesada, cuántos ficheros y cuántas subcarpetas ha encontrado. |
| **Continuar Indexación** | Reanuda un escaneo interrumpido (cerraste la pestaña, se cayó la conexión) por la carpeta en la que iba, sin borrar lo ya registrado. |

Al terminar aparece el aviso «Indexación completada». Si una petición falla, el JavaScript reintenta a los 5 segundos por sí solo.

### Paso 2. Verificar huérfanas

Toma los ficheros pendientes en lotes de 100 y, para cada uno con `id_image` válido, hace dos comprobaciones contra la base de datos:

1. ¿Existe ese `id_image` en `ps_image`? Si no → **`orphan_db`** (imagen sin registro).
2. Si existe, ¿existe su `id_product` en `ps_product`? Si no → **`orphan_product`** (imagen de un producto borrado).

Si pasa las dos, el fichero queda como `ok`.

| Botón | Efecto |
|---|---|
| **Verificar Imágenes** | Empieza la verificación. La barra de progreso muestra «Procesados lote de 100…» en cada vuelta y «Verificación Completada.» al final. |
| **Pausar** | Detiene la verificación al terminar el lote en curso. Al volver a pulsar «Verificar Imágenes» continúa por donde iba. |

La verificación **no borra nada**: solo cambia el estado de los registros en la tabla del módulo. Es seguro ejecutarla y revisar los resultados sin tocar el disco.

### Paso 3. Resultados y limpieza

Muestra la lista de huérfanas de 20 en 20, con paginación: id interno, nombre del fichero, ruta relativa dentro de `img/p/`, razón (`orphan_db` u `orphan_product`) y un botón de papelera por fila. El contador rojo de la cabecera es el total de huérfanas.

| Botón | Efecto |
|---|---|
| **Refrescar** | Vuelve a cargar la primera página de resultados y actualiza las estadísticas. |
| **Papelera de una fila** | Pide confirmación («¿Eliminar fichero?») y, si se acepta, **borra ese fichero del disco** y quita la fila. |
| **Eliminar TODAS las mostradas (Batch)** | Pide confirmación y, si se acepta, **borra del disco todos los ficheros marcados como huérfanos**, no solo los 20 de la página: los recorre en lotes de 100 hasta agotarlos, mostrando un contador de eliminados. |

**Qué se borra y qué no.** Se borran únicamente ficheros físicos de `img/p/` cuyo nombre empieza por un `id_image` que no está en `ps_image` o cuyo producto no está en `ps_product`. No se toca ninguna tabla de PrestaShop: ni `ps_image`, ni `ps_image_shop`, ni `ps_product`. Y **no hay marcha atrás**: `unlink()` directo, sin papelera, sin tabla de respaldo. De ahí la copia de seguridad de `img/p/` antes de pulsar.

Dos precauciones más antes de borrar en masa:

- Si en la tienda hay algún módulo que guarde sus propias imágenes en `img/p/` con nombres numéricos (no es lo habitual, pero existe), sus ficheros saldrán como `orphan_db`. Revisa la lista antes de usar el borrado masivo.
- Si la tabla `ps_image` está a medio migrar o se restauró una copia antigua de la base de datos sobre un `img/p/` más nuevo, **todo** lo nuevo aparecerá como huérfano. En ese caso el problema es la base de datos, no los ficheros.

### Paso 4. Limpieza de carpetas vacías

Recorre de nuevo `img/p/` (esta vez solo carpetas, sin registrar ficheros), apunta todas las subcarpetas y después las procesa de la más profunda a la menos profunda. Borra una carpeta **solo si está vacía o contiene únicamente `index.php` y/o `fileType`**; si tiene cualquier otro fichero o alguna subcarpeta, la deja tal cual.

| Botón | Efecto |
|---|---|
| **Escanear y Eliminar Carpetas Vacías** | Pide confirmación y ejecuta las dos fases seguidas (mapa de carpetas y borrado). El registro de abajo va contando carpetas revisadas y borradas. |

Es el paso natural después de un borrado masivo del paso 3, que suele dejar la estructura `1/2/3/` vacía. No hay modo de solo listar: si se confirma, borra.

### Recomendación de uso

1. Copia de seguridad de `img/p/`.
2. Paso 1 (indexar) y paso 2 (verificar). Ninguno de los dos toca el disco.
3. Paso 3: mira la lista. Si las rutas y las razones tienen sentido (imágenes de productos que sabes que se borraron), adelante con el borrado; si aparecen miles de imágenes de productos vigentes, para y revisa la base de datos.
4. Paso 4 solo después del 3, y solo si de verdad quieres el árbol de carpetas limpio.

## 6. Qué ve el cliente en la tienda

Nada. El módulo no registra ningún hook del front-office ni modifica plantillas: su único efecto sobre la tienda pública es que las imágenes borradas dejan de existir en el disco, y por definición eran imágenes que ningún producto usaba.

## 7. Preguntas frecuentes y problemas

**El escaneo tarda mucho.** Es normal en tiendas grandes: con la estructura de carpetas de PrestaShop hay una carpeta por dígito del `id_image`, y cada carpeta es una petición AJAX. En la tienda de pruebas de este manual (unas 70.000 imágenes repartidas en unas 9.600 carpetas) el indexado son casi 10.000 peticiones seguidas. Puedes cerrar la pestaña y reanudar más tarde con «Continuar Indexación».

**«Error de conexión. Reintentando en 5s…» durante el escaneo.** Una petición no ha respondido (tiempo de ejecución de PHP, carga del servidor). El módulo reintenta solo; si se queda ahí de forma permanente, revisa el registro de errores de PHP del servidor.

**Las estadísticas no cambian mientras verifica.** Es así por diseño: solo se refrescan al abrir «3. Resultados y Limpieza», al pulsar «Refrescar» o al recargar la página.

**He borrado imágenes que no debía.** El módulo no tiene forma de recuperarlas: hay que restaurarlas desde la copia de seguridad de `img/p/`. Si además eran imágenes que un producto usaba de verdad, después conviene pasar `ecom_cleanimages` para regenerar miniaturas y limpiar los registros que hayan quedado sin fichero.

**Me sale todo como `orphan_db`.** Comprueba que la base de datos a la que está conectada la tienda es la correcta y que la tabla `ps_image` tiene registros. Si acabas de restaurar una copia antigua, las imágenes nuevas no tendrán registro y aparecerán todas como huérfanas.

**Un fichero con nombre raro no aparece.** Solo se registran las extensiones `jpg`, `jpeg`, `png`, `gif` y `webp`; y los que no empiezan por un número se dan por buenos sin comprobar. El `index.php` y el `fileType` de cada carpeta se ignoran.

**No aparece en el menú.** No crea pestaña: se abre desde **Módulos > Gestor de módulos**, buscando «Huérfanas» y pulsando **Configurar**.

**Se puede desinstalar y volver a instalar sin perder nada.** Sí; las tablas del módulo son solo de trabajo y se regeneran con cada escaneo.

## 8. Ficha técnica

| Concepto | Valor |
|---|---|
| Carpeta / clase | `ecom_imagesorfan` / `Ecom_ImagesOrfan` |
| Versión | 1.1.0 |
| Autor declarado | Antigravity (no lleva la firma Ecom Experts) |
| Ficheros | `ecom_imagesorfan.php`, `views/templates/admin/configure.tpl`, `views/js/admin.js`, `views/css/admin.css` |
| Hooks | `actionAdminControllerSetMedia` |
| Controladores | Ninguno. Todas las acciones AJAX entran por `getContent()` con el parámetro `ajax_action` sobre la URL de `AdminModules&configure=ecom_imagesorfan`. |
| Acciones AJAX | `start_scan`, `process_queue`, `check_files`, `get_results`, `delete_orphan`, `delete_all_orphans`, `clean_init`, `clean_map`, `clean_execute` |
| Tablas | `ps_ecom_imagesorfan_files`, `ps_ecom_imagesorfan_queue`, `ps_ecom_imagesorfan_clean_dirs` (esta última, creada al primer uso del paso 4 y no borrada al desinstalar) |
| Claves de configuración | Ninguna |
| Cron | No tiene. Todo se ejecuta desde el navegador. |
| Registro (logs) | No escribe ficheros de log. |
| Carpeta que recorre | `_PS_PROD_IMG_DIR_` (`img/p/`), con todas sus subcarpetas |
| Extensiones que considera | `jpg`, `jpeg`, `png`, `gif`, `webp` |
| Tamaño de lote | Indexado: una carpeta por petición, inserciones de 500 en 500. Verificación: 100 ficheros por petición. Listado: 20 por página. Borrado masivo: 100 por petición. Limpieza de carpetas: hasta 4 segundos por petición, 50 carpetas por lote en el borrado. |
| Traducciones | Sistema antiguo (`{l s='…' mod='ecom_imagesorfan'}`), sin ficheros `.xlf`; los textos salen en castellano fijo. |
