# shadPS4 Performance Roadmap

Objetivo: mejorar rendimiento sin romper corrección, basándose en análisis del diseño actual.

## Contexto de diseño actual
- Ejecución nativa x86-64, sin JIT CPU.
- CPU patches para red-zone Windows y SSE4a fallback.
- MemoryManager con VMAs VMAType: Free/Reserved/PoolReserved/Pooled/Flexible/Direct/Stack/File/Code.
- Scheduler Vulkan con timeline semaphore + DeferOperation/DeferPriorityOperation.
- Shader recompiler basado en yuzu Hades.
- Fault manager para buffer cache con mapeo lazy por página.

## Mejoras priorizadas

### 0. Upscaling nativo FSR 2/3
**Estado actual**
- Existe FsrPass FSR 1 EASU+RCAS en src/video_core/renderer_vulkan/host_passes/fsr_pass.cpp
- Se usa en vk_presenter::PrepareFrame para upscaling espacial host.

**Objetivo**
- Reemplazar FSR 1 por FSR 2 temporal nativo, con opción a FSR 3.

**Requisitos**
- FSR 2 necesita color, depth y velocity a resolución de render. Reactive mask y exposure opcionales.
- FSR 2 reemplaza TAA; post-procesos que requieren anti-aliasing van post-upscale.

**Cómo integrarlo**
- Integrar FidelityFX SDK FSR 2.2.2.
- Generar velocity:
  * Opción A screen-space: guardar prevViewProj por frame y reconstruir posición 3D desde depth en compute pass.
  * Opción B mesh-based: modificar liverpool_to_vk y shaders para emitir motion vector.
- Crear Fsr2Pass análogo a FsrPass con inputs color/depth/velocity.
- Punto de inserción: vk_presenter::PrepareFrame antes de pp_pass.

**Archivos**
- src/video_core/renderer_vulkan/host_passes/fsr_pass.cpp
- src/video_core/renderer_vulkan/vk_presenter.cpp
- src/video_core/renderer_vulkan/liverpool_to_vk.cpp

**Riesgo**
Alto

### 1. Shader / Pipeline Cache Warm-up y persistencia
**Por qué**
Compilación on-demand causa stutter en primer frame.
**Acciones**
- Precargar shaders más usados al inicio del juego.
- Persistir pipeline cache por GameSerial con vkGetPipelineCacheData / vkCreatePipelineCache.
- Warm-up mejorado con lista de hashes conocidos.

**Archivos**
- src/video_core/renderer_vulkan/vk_pipeline_cache.cpp
- src/video_core/renderer_vulkan/vk_pipeline_serialization.h/cpp

**Métricas**
- Tiempo primer frame, num_new_pipelines, tiempo CompileModule.

**Riesgo**
Bajo

### 2. Contención MemoryManager
**Por qué**
SharedFirstMutex en VMAMap contiende en Map/Unmap/Protect.
**Acciones**
- Sharded locks por tipo VMA.
- Pool de páginas libres para evitar VirtualAlloc2 frecuente.
- Cache de FindVMA para accesos repetidos.

**Archivos**
- src/core/memory.cpp
- src/core/address_space.cpp
- src/core/memory.h

**Métricas**
- Latencia MapMemory/Protect, contención mutex.

**Riesgo**
Medio-bajo

### 3. Scheduler GPU Batching
**Por qué**
DeferOperation por draw genera overhead.
**Acciones**
- Batch dynamic state updates.
- Agrupar comandos por pipeline key.
- Aumentar tamaño worker command buffer.

**Archivos**
- src/video_core/renderer_vulkan/vk_scheduler.cpp
- src/video_core/renderer_vulkan/vk_rasterizer.cpp

**Métricas**
- Submit time por frame, nº DeferOperation/frame.

**Riesgo**
Bajo-medio

## Próximos pasos
1. Implementar warm-up de pipeline cache con métricas.
2. Perfilar MemoryManager con herramienta de profiling.
3. Evaluar scheduler batching en juego de referencia.
4. Prototipar Fsr2Pass y generación de velocity screen-space.
