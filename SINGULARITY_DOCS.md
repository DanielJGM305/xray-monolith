# PROJECT SINGULARITY — Documentación Interna
> Motor: X-Ray Monolith (fork: themrdemonized/xray-monolith)  
> Equipo: Daniel + Claude + Gemini  
> Última actualización: Abril 2026

---

## ROADMAP

| Fase | Nombre | Estado |
|---|---|---|
| 1 | Campamento Base (Entorno + Compilación) | ✅ COMPLETADA |
| 2 | Reconocimiento (Mapeo de sistemas) | ✅ COMPLETADA |
| 2.5 | Documentación Interna | ✅ COMPLETADA |
| 3 | Multihilo Fase 1 (Optimizaciones Seguras) | ⬜ Pendiente |
| 4 | Multihilo Fase 2 + Renderizado (Cirugía Mayor) | ⬜ Pendiente |
| 5 | El Jefe Final (Mundo Abierto sin Pantallas de Carga) | ⬜ Pendiente |

---

## RESULTADO DE FASE 1

- **Compilador:** Visual Studio 2022, configuración DX10, arquitectura x64
- **Resultado:** 32/32 proyectos compilados, 0 errores
- **Tiempo de compilación:** ~16 minutos (primera vez)
- **Ejecutable generado:** `_build\_game\bin_dbg\AnomalyDX10.exe`
- **Nota sobre DX10:** El nombre "DX10" en la configuración NO significa que el juego corra en DX10. Es solo el nombre de la build. El motor compila todos los renderers incluyendo R4 (DX11).
- **Prueba de ejecución:** El exe arrancó y pasó el splash screen. Falló por incompatibilidad de shaders con mods externos (SSFX) — problema de entorno, no del motor.

---

## ARQUITECTURA DEL MOTOR — SISTEMAS MAPEADOS

### 1. Sistema de Dispatching — `pure.h`
**Ubicación:** `src/xrEngine/Engine/pure.h` y `pure.cpp`

**¿Qué hace?**  
Es el sistema de eventos central del motor. Cada sistema (render, física, A-Life) se registra con una prioridad y se ejecuta en orden cada frame.

**Clase clave:** `CRegistrator<T>`

**Puntos de registro disponibles:**
- `Frame` — cada fotograma (corazón del main loop)
- `Render` — cuando toca dibujar
- `AppStart / AppEnd` — inicio y cierre del juego
- `AppActivate / AppDeactivate` — foco de ventana
- `DeviceReset` — cambio de resolución o pérdida del dispositivo DX
- `ScreenResolutionChanged` — cambio de resolución

**Prioridades disponibles:**
```cpp
REG_PRIORITY_LOW      = 0x11111111
REG_PRIORITY_NORMAL   = 0x22222222
REG_PRIORITY_HIGH     = 0x33333333
REG_PRIORITY_CAPTURE  = 0x7fffffff  // bloquea todo lo demás
REG_PRIORITY_INVALID  = 0xffffffff  // marcado para eliminar
```

**Cuello de botella identificado:**
```cpp
void Process(RP_FUNC* f) {
    in_process = true;
    for (u32 i = 0; i < R.size(); i++)
        f(R[i].Object);  // ← Todo ejecuta secuencial, un solo hilo
    in_process = false;
}
```

**Plan Fase 3:** Reemplazar el loop secuencial con dispatch paralelo usando `std::for_each` con política de ejecución paralela para sistemas que no tengan dependencias entre sí.

---

### 2. Scheduler — `xrSheduler.cpp`
**Ubicación:** `src/xrEngine/Engine/Core/xrSheduler.cpp` y `xrSheduler.h`

**¿Qué hace?**  
Administra cuándo se ejecuta cada sistema del juego. Usa una cola de prioridad basada en tiempo — cada sistema declara cada cuántos ms quiere actualizarse y el scheduler lo ejecuta cuando corresponde.

**Dos tipos de objetos:**
- **ItemsRT (Real Time):** se ejecutan CADA frame sin excepción (ej: física del jugador)
- **Items (Normal):** se ejecutan según su intervalo de tiempo configurado (ej: A-Life lejano)

**Throttling automático:**
```cpp
// El scheduler se autoregula entre 3ms y 66ms por frame
psShedulerCurrent = 0.9f * psShedulerCurrent + 0.1f * psShedulerTarget;
clamp(psShedulerTarget, 3.f, 66.f);
```
Si está sobrecargado, reduce cuántos objetos procesa por frame automáticamente.

**Cuello de botella identificado:**
```cpp
bool m_processing_now;  // ← flag booleano simple, no thread-safe
```
Un solo booleano protege toda la ejecución. No hay sincronización real.

**Hallazgo histórico:** Hay código de fibras comentado en el archivo:
```cpp
/*
void CSheduler::Switch() {
    if (fibered) SwitchToFiber(fiber_main);
}
*/
```
Alguien intentó implementar esto con fibras antes y lo abandonó. Confirma que threading real es el camino correcto.

**Plan Fase 3:** Reemplazar `m_processing_now` con un `std::mutex` real y separar la ejecución de objetos RT en un hilo dedicado.

---

### 3. Main Loop y Threading — `device.cpp`
**Ubicación:** `src/xrEngine/device.cpp`

**¿Qué hace?**  
`CRenderDevice` es el núcleo del motor. Contiene el main loop, gestiona los hilos y coordina render con lógica de juego.

**El motor YA tiene multithreading parcial:**
```cpp
// En CRenderDevice::Run():
thread_spawn(mt_FreezeThread, "Freeze detecting thread", 0, 0);
thread_spawn(mt_Thread,       "X-RAY Secondary thread",  0, this);
thread_spawn(mt_DiscordThread,"X-RAY Discord thread",    0, 0);
```

**Flujo por frame en `on_idle()`:**
```
Hilo Principal (Primary):          Hilo Secundario (mt_Thread):
──────────────────────────         ──────────────────────────────
1. FrameMove()                     
   └─ seqFrame.Process()           
2. mt_csLeave.Enter() ──────────→  se despierta
3. seqRender.Process() (render)    seqParallel[] (tareas paralelas)
                                   seqFrameMT.Process()
                                   LuaGC (limpieza de memoria Lua)
4. mt_csEnter.Enter() ←──────────  se duerme
```

**Sincronización actual (frágil):**
```cpp
mt_csLeave.Enter();   // libera el hilo secundario
mt_csEnter.Leave();   // ... (render en curso) ...
mt_csEnter.Enter();   // congela el hilo secundario
mt_csLeave.Leave();   // completa el ciclo
```
Dos CriticalSection usadas como ping-pong. Funciona pero es limitante.

**Hallazgo importante:**
```cpp
// mem_compact() SOLO funciona en modo DEBUG:
void xrMemory::mem_compact() {
#ifdef DEBUG_MEMORY_MANAGER   // ← bug: en Release no hace nada
    _heapmin();
    HeapCompact(GetProcessHeap(), 0);
#endif
}
```

**Plan Fase 4:** Reemplazar el sistema de ping-pong con un `std::barrier` o similar para permitir más de 2 hilos sincronizados.

---

### 4. A-Life Update Manager — `alife_update_manager.cpp`
**Ubicación:** `src/xrGame/alife_update_manager.cpp`

**¿Qué hace?**  
Gestiona la simulación de todos los stalkers y mutantes en el mundo, incluyendo los que están fuera del área visible del jugador (offline). Es el sistema más costoso en CPU del juego.

**Estructura de herencia del A-Life:**
```
CALifeSimulator
├── CALifeUpdateManager    ← aquí vive el update loop
├── CALifeInteractionManager  ← interacciones entre NPCs
└── CALifeSimulatorBase    ← datos base
```

**HALLAZGO CRÍTICO — El paralelismo YA existe:**
```cpp
void CALifeUpdateManager::shedule_Update(u32 dt) {
    if (!m_first_time && g_mt_config.test(mtALife)) {
        Device.seqParallel.push_back(
            fastdelegate::FastDelegate0<>(this, &CALifeUpdateManager::update)
        );
        return;  // ← sale sin ejecutar, lo hace el hilo secundario
    }
    // primera vez: ejecuta en el hilo principal
    update();
}
```
El A-Life ya tiene soporte para correr en `seqParallel` (hilo secundario). El flag `mtALife` está activo por defecto. El problema no es activarlo sino que hay condiciones de carrera no resueltas.

**Plan Fase 3:** Identificar y resolver las condiciones de carrera que causan inestabilidad cuando `mtALife` está activo.

---

### 5. Configuración de Multithreading — `mt_config.h`
**Ubicación:** `src/xrGame/mt_config.h`

**Todos los flags disponibles:**
```cpp
#define mtLevelPath      (1<<0)  // Pathfinding de nivel
#define mtDetailPath     (1<<1)  // Pathfinding detallado
#define mtObjectHandler  (1<<2)  // Manejo de objetos
#define mtSoundPlayer    (1<<3)  // Audio
#define mtAiVision       (1<<4)  // Visión de NPCs
#define mtBullets        (1<<5)  // Física de balas
#define mtLUA_GC         (1<<6)  // Garbage collector de Lua
#define mtLevelSounds    (1<<7)  // Sonidos del nivel
#define mtALife          (1<<8)  // Simulador A-Life ← el más importante
#define mtMap            (1<<9)  // El mapa
```

**Flags activos por defecto** (en `console_commands.cpp`, línea 277):
```cpp
Flags32 g_mt_config = {
    mtLevelPath | mtDetailPath | mtObjectHandler | mtSoundPlayer | 
    mtAiVision | mtBullets | mtALife | mtMap
};
// Nota: mtLUA_GC y mtLevelSounds están INACTIVOS por defecto
```

**Conclusión:** 8 de 10 sistemas ya tienen paralelismo declarado. La Fase 3 es estabilización, no construcción desde cero.

---

### 6. Sistema de Memoria — `xrMemory.cpp`
**Ubicación:** `src/xrCore/xrMemory.cpp`

**¿Qué hace?**  
Gestiona toda la memoria dinámica del motor usando un sistema de pools estáticos pre-allocados al inicio del juego.

**Arquitectura actual:**
```
xrMemory
├── mem_pools[] → pools de tamaño fijo (estático)
├── mem_copy    → implementación MMX o x86 según CPU
├── mem_fill    → idem
└── debug_info  → solo en DEBUG_MEMORY_MANAGER
```

**Bug crítico identificado:**
```cpp
void xrMemory::mem_compact() {
#ifdef DEBUG_MEMORY_MANAGER  // ← SOLO EN DEBUG, en Release no hace NADA
    _heapmin();
    HeapCompact(GetProcessHeap(), 0);
    if (g_pStringContainer) g_pStringContainer->clean();
#endif
}
```
En builds de release, `mem_compact()` es una función vacía. Esto significa que la memoria nunca se compacta en producción.

**Limitación fundamental para Fase 5:**  
Los pools son de tamaño fijo pre-allocados. No pueden crecer dinámicamente. Para hacer level streaming necesitamos un gestor que pueda:
1. Allocar y liberar regiones grandes dinámicamente
2. Compactar memoria en runtime (no solo en debug)
3. Manejar cargas asíncronas sin fragmentar el heap

**Plan Fase 5:** Reemplazar el sistema de pools estáticos por un allocator dinámico con soporte para zonas de memoria dedicadas al streaming.

---

## NOTAS TÉCNICAS IMPORTANTES

### Sobre DirectX
- La configuración "DX10" en VS2022 NO significa que el juego use DX10
- El motor compila 4 renderers: R1(DX8), R2(DX9), R3(DX10), R4(DX11)
- El juego usa DX11 (R4) por defecto en hardware moderno

### Sobre el repo
- Fork base: `themrdemonized/xray-monolith`
- Nuestro fork: `DanielJGM305/xray-monolith`
- Branch principal: `all-in-one-vs2022-wpo`
- Solución VS: `src/engine-vs2022.sln`

### Convenciones de código del motor
- `xr_` prefijo para tipos y funciones propias del engine
- `pure` prefijo para clases base de eventos (`pureFrame`, `pureRender`, etc.)
- `IC` = `inline` (`__forceinline` en release)
- `VERIFY()` = assert que funciona en debug y release
- `Shedule` (con una sola 'e') es un typo histórico — así está en TODO el código

### Archivos clave para referencia rápida
| Sistema | Header | Implementación |
|---|---|---|
| Dispatcher | `pure.h` | `pure.cpp` |
| Scheduler | `xrSheduler.h` | `xrSheduler.cpp` |
| Device/MainLoop | `device.h` | `device.cpp` |
| A-Life | `alife_simulator.h` | `alife_update_manager.cpp` |
| MT Flags | `mt_config.h` | `console_commands.cpp` (línea 277) |
| Memoria | `xrMemory.h` | `xrMemory.cpp` |

---

## PRÓXIMOS PASOS — FASE 3

**Objetivo:** Optimizaciones seguras de multithreading sin romper el juego.

**Tareas identificadas (orden sugerido):**

1. **Investigar condiciones de carrera en mtALife** — el flag ya está activo, pero hay inestabilidad. Identificar exactamente qué estructuras de datos se acceden desde múltiples hilos sin protección.

2. **Corregir mem_compact() en Release** — fix rápido y de alto impacto. Sacar el `#ifdef DEBUG_MEMORY_MANAGER` de las llamadas a `_heapmin()` y `HeapCompact()`.

3. **Integrar herramienta de profiling** — antes de optimizar más, necesitamos datos. Opciones: Very Sleepy (gratis) o el profiler integrado de VS2022.

4. **Limpiar código legacy** — modernizar partes antiguas de C++ sin cambiar funcionalidad (usar `nullptr` en lugar de `NULL`, `auto` donde corresponda, etc.).

---

*Documento generado durante Fase 2.5 de PROJECT SINGULARITY*  
*Equipo: Daniel (líder) + Claude (especialista C++/memoria) + Gemini (arquitectura)*

---

## FIXES COMPLETADOS — FASE 3

### Fix 1 — mem_compact() en Release
**Archivo:** `src/xrCore/xrMemory.cpp`  
**Estado:** ✅ Completado

Removido el `#ifdef DEBUG_MEMORY_MANAGER` para que `mem_compact()` funcione en builds de Release.

---

### Fix 2 — xrSheduler.cpp modernización
**Archivo:** `src/xrEngine/Engine/Core/xrSheduler.cpp`  
**Estado:** ✅ Completado

- `NULL` → `nullptr` en punteros reales
- Iteradores modernizados con range-based for loops

---

### Fix 3 — Discord Thread Event-Driven
**Archivos:** `src/xrEngine/device.h`, `src/xrEngine/device.cpp`  
**Estado:** ✅ Completado  
**Branch:** `phase3-threading-fixes`

**Problema:** `mt_DiscordThread` usaba `Sleep(1000)` en polling loop ciego. El profiling baseline (`profile_vanilla_01.sleepy`) mostró que la dirección `0x1400CACB0` era responsable de 210 segundos de `SleepEx` — ~35% del tiempo total de profiling.

**Solución:** Reemplazado por modelo event-driven con `WaitForMultipleObjects`:
- `hDiscordWakeEvent` (auto-reset) — despierta el hilo ante cambio de estado
- `hDiscordShutdownEvent` (manual-reset) — apaga el hilo limpiamente
- Cuando Discord está inactivo: duerme `INFINITE` en lugar de `Sleep(1000)`
- Cleanup limpio de handles al cerrar el engine

**Resultado verificado:** El segundo perfil (`profile_singularity_02.sleepy`) confirmó que el Discord thread desapareció completamente de los callers de `SleepEx`. El log de shutdown confirma operación correcta:
```
[Discord] Shutdown signal received, exiting thread
```
Carga del juego notablemente más rápida como efecto secundario.

---

## PROBLEMA CONOCIDO — Render negro en partida (R4/stub_default)

**Estado:** Abierto  
**Detectado:** 27/04/2026

### Síntoma
Al copiar el exe compilado (AnomalyDX11.exe) a Anomaly 1.5.3 y cargar una partida, el mundo 3D se ve negro. UI, inventario y audio funcionan correctamente.

### Causa identificada
El log muestra shaders faltantes reemplazados por stub_default (shader vacío):
```
DX10: ...shaders\r3\deffer_terrain_low_flat.ps is missing. Replace with stub_default.ps
```
Los shaders del gamedata de Anomaly 1.5.3 no son 100% compatibles con xray-monolith compilado desde source.

### Lo que SÍ funciona
- Engine compila y arranca correctamente (0 errores)
- UI renderiza bien
- Discord fix operando correctamente (confirmado por log)
- Profiling válido

### Próximos pasos para resolver
- Investigar qué shaders necesita xray-monolith vs los de Anomaly stock
- Posiblemente compilar shaders desde el SDK del repo
- Consultar comunidad de themrdemonized sobre compatibilidad
