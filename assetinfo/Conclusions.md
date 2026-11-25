# SixNG format specification

GTA VI, Xbox Series build. Everything here was worked out from the data
itself: headers, statistics over many samples, and checking decoded output
by eye. Nothing has been confirmed against game code or official
documentation.

Numbers are **as of 2026-11-20**, from one retail Xbox Series install.

**Confidence levels** used in every table:

- **confirmed**: holds over many samples, and decoded output looks right
  when opened.
- **inferred**: consistent with the data, not proven.
- **unknown**: not worked out.

**Conventions:** offsets are hex from the start of the structure;
little-endian unless stated. Hashes are hex. "joaat" is the
Jenkins one-at-a-time hash used by RAGE; where a field name is known it is
the **case-sensitive** joaat of that name. "GTA V equivalent" names
(`CEntityDef`, `.ydr`, `.ynd` …) are the names the same structures have in
GTA V, used only to say what a structure corresponds to.

## Pages

**Containers**

- [1. RPF9 archives](01-rpf9.md)
- [2. Resources and pages](02-resources.md)

**Textures**

- [3. Texture dictionaries](03-texture-dictionaries.md)
- [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md)
- [5. `ttpf` texture stores](05-ttpf.md)
- [6. BCPack](06-bcpack.md)
  - [6A. BCPack tables](06a-bcpack-tables.md)

**Models**

- [7. Drawables, fragments and vehicles](07-models.md)

**World data**

- [8. Meta resources (type 0x7a)](08-meta.md)
- [9. Placements, LOD links, archetypes and batches](09-placements.md)
- [10. Interiors (MLO)](10-interiors.md)
- [13. Collision and surface categories](13-collision.md)
- [14. Navigation meshes](14-navmesh.md)
- [15. Lights](15-lights.md)
- [16. Road graph, junctions and flags](16-roads.md)

**Materials**

- [11. Vehicle paint and vehicle tables](11-vehicle-paint.md)
- [12. Material classes and texture slots](12-materials.md)

**Other**

- [17. Other formats](17-other.md)
- [18. Radio station audio and scheduling](18-radio.md)
- [19. Strand hair, cloth and head deltas (unnamed `packs.rpf` resources)](19-ped-physics-resources.md)

Each page ends with an **Open questions** list.

**Byte-level notes.** The [structure/](../structure/00-INDEX.md) pages
record layouts as raw byte facts with the test behind each one. They also
cover audio (`ADAT`), the PSO object graph and the shader cache, which these
pages only mention. Where the two sets disagree, both readings are given.

## Conclusiones de la revisión local (2026-11-24)

- `textures.rpf` tiene cabecera RPF9 y un `toc_end` dentro del archivo; el
  u32 de `+0x24` vale `0x16`, pero contar entradas a partir de ese campo es
  una inferencia. Como la TOC está cifrada, esta revisión no confirma los
  miembros ni que todo el contenido sea de texturas.
- La extensión identifica el contenedor, no necesariamente el tipo único
  de contenido. RPF9 requiere clasificar cada payload recuperado; TTPF es
  un almacén de texturas, pero sus recursos mezclan codecs y layouts. La
  aplicación debe despachar por firma/descriptores y codec, no usar un
  único decodificador por extensión.
- Los datos Bink documentados (30 fps, tabla de offsets y duración de audio
  dentro de un frame) son evidencia de formato/decodificación, no una
  prueba de sincronización en el programa. No existe fuente en `app/` ni
  harness de reproducción en esta carpeta; por tanto no se afirma que la
  sincronización en tiempo de ejecución haya sido verificada. La
  comprobación propuesta y el umbral A/V de un frame están en
  [Video: Runtime synchronization check](17-other.md#runtime-synchronization-check).
- No se adaptó `Bink2Scarlett.dll`: solo se confirmó que es PE x64 y la
  captura la asocia a otro ejecutable. Sin imports/exportaciones/API del
  programa actual y sin SDK, adaptar o redistribuirla sería especulativo.
- Tras integrar Vendetta, `cargo build --release -p victory --locked` se
  verificó correctamente y generó `target/release/victory.exe`. El juego no
  se lanzó para esta verificación.

## Observación de la cinemática en la terminal (2026-11-24)

**Observado en la captura más reciente:** el título mostró `Americas |
VIctory 0 fps 3233.7 ms/frame 1327 draws/frame`; la cinemática seguía
negra y no se reportó un crash. Una captura anterior mostró `0.4 fps`,
`2800.3 ms/frame` y `1459 draws/frame`; el desglose de esa ejecución fue
`912.3 ms` preparando, incluidos `486.9 ms / 167.1 MB` en copias y
`41.5 ms / 144.7 MB` en texturas, `1547.3 ms` en GPU y
`329.5 ms / 4.6 MB` copiando resultados de vuelta. Son ejecuciones
distintas y no deben combinarse como si fueran una sola medición.

**Interpretación:** `1327 draws/frame` no significa 1327 draws/ms. Los
`3233.7 ms/frame` equivalen a unos 0.31 fps, frente al intervalo de 33.33 ms
del menú. Esto confirma una lentitud grave durante la cinemática, pero no
demuestra por sí solo que la causa sean shaders, descompresión, orden de
lectura o mipmaps. El desglose anterior sitúa el mayor coste observado en
GPU y copia, no en la preparación de texturas; por sí solo no justifica
alterar formatos ni omitir mipmaps.

## Fallo posterior a la fusión (2026-11-24)

La nueva ejecución no llegó al menú principal. Primero reportó
`0.1 fps / 13684.4 ms/frame`, con `13450.0 ms` preparando y `3.6 ms` en GPU;
después alcanzó `31.0 fps / 32.3 ms/frame`. Más tarde el dispositivo D3D12
se eliminó al enviar una sola lista y devolvió `0x887A0001`
(`DXGI_ERROR_INVALID_CALL`). DRED no pudo obtener breadcrumbs (`0x887A0004`),
por lo que el log no señala la instrucción culpable.

La pista más concreta son los índices de heap:

- Victory informó índices de título `1245180`–`1245183` con
  `count=934464`. En [host.rs](../../crates/victory-gpu/src/descriptor/host.rs),
  cuando la creación del heap host
  grande falla, el fallback limita el heap visible a 1,000,000 slots y
  reserva 65,536 para slots dinámicos: quedan `934464` slots de título.
- Los shaders traducidos, sin embargo, limitan los índices de recursos hasta
  `HEAP_VIEWS = 2^21` ([shader.rs](../../crates/victory-shader/src/emit/shader.rs)). Los índices
  superiores al heap host no se reflejan (`mirror` devuelve `None`), y el
  acceso bindless directo puede acabar fuera del heap físico. La coincidencia
  de valores hace de este un **candidato fuerte**, no una causa confirmada:
  también hubo texturas que el dispositivo no pudo crear y el log carece de
  validación GPU que identifique la llamada inválida.
- El aviso `no room for the null buffer views` con `count=0 total=65536`
  pertenece a otro heap pequeño; por sí solo no prueba que ese heap se haya
  usado en la lista fallida.

Se compiló `target/diag/victory.exe` con `--features diagnostics`. La
siguiente comprobación debe repetir la misma entrada con
`VICTORY_D3D12_DEBUG=1`, `VICTORY_D3D12_GBV=1` y
`VICTORY_D3D12_DRED=1`; GPU-based validation puede ser muy lenta, pero debe
confirmar o descartar el acceso fuera del heap. No conviene bajar el límite
global de índices sin medir: podría evitar el acceso inválido a costa de
aliasar descriptors y ocultar recursos del título.

## Selección del adaptador D3D12

La línea `host: AMD Ryzen 7 5700G with Radeon Graphics` identifica el
procesador/APU del equipo, no demuestra qué adaptador creó el dispositivo
D3D12. El código anterior usaba `D3D12CreateDevice(None, ...)`, que deja esa
elección al adaptador predeterminado de Windows. La integración ahora
enumera los adaptadores de hardware, prefiere NVIDIA y registra el nombre,
vendor y memoria dedicada del adaptador escogido en el log. Esto permite
comprobar si realmente se seleccionó la RTX 5050.

La selección de NVIDIA no corrige automáticamente el posible límite del
heap: el fallback documentado en [host.rs](../../crates/victory-gpu/src/descriptor/host.rs)
puede seguir dejando fuera los índices altos. El próximo log debe confirmar
tanto `host D3D12 adapter selected` como si siguen apareciendo índices de
heap superiores a `count`. CUDA no participa en el renderizado ni en la
traducción de shaders de Victory, que entrega DXIL a D3D12.

## Mipmaps, modelos y secuencia de carga

- En la ruta de texturas de Victory, el primer uso sube todas las huellas
  mip/slice y, cuando cambia la memoria del título, el detiler visita todos
  los mip/slice. Los samplers conservan el MinLOD/MaxLOD del título; no hay
  una política de residencia que priorice mipmaps gruesos. Un clamp MinLOD
  aislado no evita las lecturas DirectStorage ni la preparación CPU de la
  textura.
- RPF9 es el contenedor. `.#dr` describe un drawable con su grupo LOD;
  `.#ft` es un fragmento que enlaza a un drawable y puede incluir esqueleto
  y partes hijas. Personajes, ropa y piezas de vehículos pueden depender de
  esos enlaces; edificios también pueden contener componentes drawable. No
  se encontró evidencia de un orden universal por extensión: deben
  preservarse los enlaces y dependencias de cada recurso. Véanse
  [modelos](07-models.md#runtime-dependency-notes-2026-11-24) y
  [ruta de mipmaps](03-texture-dictionaries.md#mip-storage-versus-runtime-upload-2026-11-24).
- La integración de Vendetta incorpora recursos indexados directamente
  desde el heap de descriptores (`bindless.rs`), seguimiento/copia de
  páginas de memoria del título (`memory.rs`) y preconstrucción de
  pipelines desde los shader packs (`apps/victory/src/prebuild.rs`). Son
  cambios relevantes para bindings, coherencia de memoria y compilación,
  pero todavía no se ha demostrado que resuelvan esta pantalla negra.
- El planificador de frames reserva cada tick bajo el lock del reloj y
  espera fuera del lock; dos waits del mismo tipo no terminan en el mismo
  tick. DirectStorage registra inicio/fin de solicitudes con cola,
  secuencia, offsets, tamaños y duración. Estas mediciones permiten
  investigar carreras y tiempos, pero no prueban que una carrera causara
  los `3233.7 ms/frame`.
- DirectStorage vuelve al paralelismo por defecto del hardware. Para una
  comparación secuencial, `VICTORY_DSTORAGE_WORKERS=1`; `auto` o dejar la
  variable sin definir usa el valor por defecto. Esto contrasta la hipótesis
  del orden de carga; no implica que los formatos exijan un orden global.

## Siguiente comprobación

Usar un build `diag` con `VICTORY_D3D12_DEBUG=1`,
`VICTORY_D3D12_DRED=1`, `VICTORY_GPU_TIMING=1` y
`VICTORY_LOG=victory_hle=trace,info` durante la misma escena. Comparar
el campo `textures X ms Y MB` del contador, las listas lentas y las
solicitudes DirectStorage (comparar `VICTORY_DSTORAGE_WORKERS=1` con
`auto`, y revisar el orden/tiempo de secuencia);
`VICTORY_D3D12_WARP=1` sirve como comparación
que no puede resetear el driver, aunque será lento.
`VICTORY_SHADER_LOOP_LIMIT` corta
bucles traducidos y cambia el resultado visual: sirve como prueba A/B para
identificar shaders costosos, no como solución de calidad estable.
