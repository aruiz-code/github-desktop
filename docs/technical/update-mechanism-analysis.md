# Análisis del Mecanismo de Actualización de GitHub Desktop

## Índice

1. [Mecanismo de Actualización](#1-mecanismo-de-actualización)
2. [Elementos del Mecanismo](#2-elementos-que-forman-parte-del-mecanismo-de-actualización)
3. [Algoritmo Detallado](#3-algoritmo-detallado)
4. [Diagrama de Secuencia](#4-información-para-diagrama-de-secuencia)
5. [Mapa Mental](#5-información-para-mapa-mental)
6. [Mejores Prácticas Implementadas](#6-mejores-prácticas-implementadas)
7. [Recomendaciones para C# y .NET 10](#7-recomendaciones-para-implementación-en-c-y-net-10)

---

## 1. Mecanismo de Actualización

### Descripción General

GitHub Desktop implementa un sistema de actualización automática basado en la arquitectura de **Electron autoUpdater**, que a su vez utiliza **Squirrel** como motor de instalación/actualización en cada plataforma. El mecanismo sigue un patrón de **publicación-suscripción** (pub/sub) entre el proceso principal (main) y el proceso de renderizado (renderer) de Electron, comunicándose a través de canales IPC (Inter-Process Communication) fuertemente tipados.

### Arquitectura de Alto Nivel

```
┌──────────────────────────────────────────────────────────────────┐
│                    Proceso Renderer (UI)                         │
│                                                                  │
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────────┐  │
│  │ App.tsx      │───▶│  UpdateStore      │───▶│ UpdateAvailable│  │
│  │ (Scheduler)  │    │  (State Machine)  │    │ Banner (UI)    │  │
│  └─────────────┘    └──────────────────┘    └────────────────┘  │
│         │                    │ ▲                                  │
│         │                    │ │ IPC Events                      │
├─────────┼────────────────────┼─┼────────────────────────────────┤
│         │    IPC Bridge      │ │                                 │
│         │  (ipc-shared.ts)   │ │                                 │
├─────────┼────────────────────┼─┼────────────────────────────────┤
│         ▼                    ▼ │                                 │
│  ┌─────────────┐    ┌──────────────────┐                        │
│  │  main.ts    │───▶│  AppWindow       │                        │
│  │  (IPC Hub)  │    │  (autoUpdater)   │                        │
│  └─────────────┘    └──────────────────┘                        │
│                              │                                   │
│                    Proceso Main (Node.js)                        │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                               ▼
                  ┌──────────────────────┐
                  │  Electron autoUpdater│
                  │  ┌────────────────┐  │
                  │  │ Squirrel.Win   │  │  (Windows)
                  │  │ Squirrel.Mac   │  │  (macOS)
                  │  └────────────────┘  │
                  └──────────────────────┘
                               │
                               ▼
                  ┌──────────────────────┐
                  │ central.github.com   │
                  │ (Update Server)      │
                  └──────────────────────┘
```

### Plataformas Soportadas

| Plataforma | Motor de Actualización | Formato de Paquete | Actualización Automática |
|------------|----------------------|-------------------|------------------------|
| **Windows** | Squirrel.Windows | `.nupkg` (NuGet) | ✅ Sí |
| **macOS** | Squirrel.Mac (nativo Electron) | `.zip` con firma de código | ✅ Sí |
| **Linux** | N/A | N/A | ❌ No (instalación manual) |

---

## 2. Elementos que Forman Parte del Mecanismo de Actualización

### 2.1 Archivos Principales

| Archivo | Rol | Proceso |
|---------|-----|---------|
| `app/src/ui/lib/update-store.ts` | Máquina de estados de actualización | Renderer |
| `app/src/main-process/app-window.ts` | Configuración del autoUpdater de Electron | Main |
| `app/src/main-process/main.ts` | Registro de handlers IPC y manejo de eventos Squirrel | Main |
| `app/src/main-process/squirrel-updater.ts` | Manejo de eventos de ciclo de vida Squirrel (Windows) | Main |
| `app/src/ui/main-process-proxy.ts` | Proxy IPC para comunicación renderer → main | Renderer |
| `app/src/lib/ipc-shared.ts` | Definiciones tipadas de canales IPC | Compartido |
| `app/src/ui/banners/update-available.tsx` | Banner de UI para notificar actualizaciones | Renderer |
| `app/src/ui/installing-update/installing-update.tsx` | Diálogo de instalación en progreso | Renderer |
| `app/src/lib/get-updater-guid.ts` | Gestión de GUID para despliegue escalonado | Main |
| `app/src/lib/squirrel-error-parser.ts` | Parser de errores amigables (Windows) | Renderer |
| `app/src/lib/feature-flag.ts` | Feature flags para funcionalidades de actualización | Compartido |
| `app/src/lib/release-notes.ts` | Obtención y parseo de notas de versión | Renderer |
| `app/src/models/release-notes.ts` | Modelos de datos para notas de versión | Compartido |
| `script/dist-info.ts` | Configuración de URLs y paquetes de distribución | Build |
| `app/app-info.ts` | Variables globales de compilación (`__UPDATES_URL__`, etc.) | Build |
| `app/src/ui/app.tsx` | Scheduler de verificación periódica y lógica de banners | Renderer |

### 2.2 Canales IPC

#### Canales Simplex (Renderer → Main)
```typescript
'quit-and-install-updates': () => void        // Instalar y reiniciar
'will-quit': () => void                       // Notificación de cierre
'will-quit-even-if-updating': () => void      // Forzar cierre durante actualización
'cancel-quitting': () => void                 // Cancelar cierre
'show-installing-update': () => void          // Mostrar diálogo de instalación
```

#### Canales Simplex (Main → Renderer)
```typescript
'auto-updater-error': (error: Error) => void           // Error en actualización
'auto-updater-checking-for-update': () => void          // Verificando actualizaciones
'auto-updater-update-available': () => void             // Actualización disponible
'auto-updater-update-not-available': () => void         // Sin actualizaciones
'auto-updater-update-downloaded': () => void            // Descarga completada
'show-installing-update': () => void                    // Mostrar diálogo
```

#### Canales Duplex (Request-Response)
```typescript
'check-for-updates': (url: string) => Promise<Error | undefined>  // Verificar actualizaciones
```

### 2.3 Estados de Actualización (UpdateStatus)

```typescript
enum UpdateStatus {
  UpdateNotChecked,       // Estado inicial, no se ha verificado
  CheckingForUpdates,     // Verificación en progreso
  UpdateAvailable,        // Actualización disponible, descargando
  UpdateNotAvailable,     // No hay actualizaciones
  UpdateReady,            // Descargada, lista para instalar
}
```

### 2.4 Estado Completo (IUpdateState)

```typescript
interface IUpdateState {
  status: UpdateStatus                              // Estado actual
  lastSuccessfulCheck: Date | null                  // Última verificación exitosa
  isX64ToARM64ImmediateAutoUpdate: boolean          // Migración x64→ARM64
  newReleases: ReleaseSummary[] | null              // Notas de nuevas versiones
  prioritizeUpdate: boolean                         // Actualización prioritaria
  prioritizeUpdateInfoUrl: string | undefined       // URL de información prioritaria
}
```

### 2.5 Servidor de Actualizaciones

- **URL Base**: `https://central.github.com/api/deployments/desktop/desktop/`
- **Parámetros**:
  - `version`: Versión actual de la aplicación
  - `env`: Canal de release (`production`, `beta`, `test`, `development`)
  - `guid`: UUID persistente para despliegue escalonado
- **Arquitectura**: Segmento `/arm64/` para builds ARM64
- **Notas de versión**: `https://central.github.com/deployments/desktop/desktop/changelog.json`

---

## 3. Algoritmo Detallado

### 3.1 Fase de Inicialización

```
1. INICIO del proceso Main (main.ts)
2. SI plataforma = Windows Y argumentos_proceso.longitud > 1:
   a. arg ← argumentos_proceso[1]
   b. SI arg ∈ {--squirrel-install, --squirrel-updated, --squirrel-uninstall, --squirrel-obsolete}:
      - Ejecutar handleSquirrelEvent(arg)
      - Salir de la aplicación al completar
      - FIN (no continuar con la app normal)
3. Solicitar bloqueo de instancia única
4. SI es instancia duplicada: Salir
5. Crear ventana principal (AppWindow)
6. Ejecutar setupAutoUpdater() → registrar listeners en autoUpdater de Electron:
   - 'error' → enviar 'auto-updater-error' al renderer
   - 'checking-for-update' → enviar 'auto-updater-checking-for-update'
   - 'update-available' → marcar isDownloadingUpdate=true, enviar 'auto-updater-update-available'
   - 'update-not-available' → enviar 'auto-updater-update-not-available'
   - 'update-downloaded' → marcar isDownloadingUpdate=false, enviar 'auto-updater-update-downloaded'
7. Registrar handlers IPC:
   - 'check-for-updates' → AppWindow.checkForUpdates(url)
   - 'quit-and-install-updates' → AppWindow.quitAndInstallUpdate()
```

### 3.2 Fase de Programación de Verificaciones (Scheduler)

```
1. App.tsx cargada → componentDidMount() → performDeferredLaunchActions()
2. SI canal_release ∈ {production, beta}:
   a. Ejecutar checkForUpdates(inBackground=true) inmediatamente
   b. Programar setInterval(checkForUpdates(true), 4_horas)
3. SI canal_release ∈ {development, test}:
   a. SI isUpdateShowcase() = true:
      - Mostrar banner de showcase (notas de versión)
```

### 3.3 Fase de Búsqueda de Actualizaciones

```
FUNCIÓN checkForUpdates(inBackground, skipGuidCheck):
  1. SI plataforma = Linux: RETORNAR (sin soporte)
  2. SI canal = development: RETORNAR
  3. SI Windows < 10: REGISTRAR error, RETORNAR
  4. SI macOS < 11.0: REGISTRAR error, RETORNAR
  5. SI status = UpdateReady: RETORNAR (ya hay actualización lista)
  
  6. updatesUrl ← getUpdatesUrl(skipGuidCheck):
     a. url ← __UPDATES_URL__ (de compilación)
     b. SI enableUpdateFromEmulatedX64ToARM64() Y isRunningUnderARM64Translation():
        - Reemplazar ruta a /arm64/latest
        - SI soporta actualización inmediata:
          - Falsear versión = "0.0.64" (forzar actualización)
     c. SI NO skipGuidCheck:
        - guid ← getUpdaterGUID()
        - Agregar guid como parámetro de consulta
     d. RETORNAR url
  
  7. Enviar IPC 'check-for-updates' con updatesUrl al proceso Main
  8. Main: autoUpdater.setFeedURL({url: trySetUpdaterGuid(url)})
  9. Main: autoUpdater.checkForUpdates()
```

### 3.4 Fase de Descarga

```
CUANDO autoUpdater emite 'update-available':
  1. Main: isDownloadingUpdate ← true
  2. Main: Enviar IPC 'auto-updater-update-available' al renderer
  3. Renderer: UpdateStore.status ← UpdateAvailable
  4. Renderer: Emitir evento de cambio de estado
  
  [Electron/Squirrel descarga automáticamente en segundo plano]
  
CUANDO autoUpdater emite 'update-downloaded':
  1. Main: isDownloadingUpdate ← false
  2. Main: Enviar IPC 'auto-updater-update-downloaded' al renderer
  3. Renderer: UpdateStore.onUpdateDownloaded():
     a. Obtener resumen de notas de versión (generateReleaseSummary)
     b. Verificar migración ARM64:
        - SI supportsImmediateUpdateFromEmulatedX64ToARM64()
          Y newReleases tiene exactamente 1 entrada
          Y versión coincide con versión actual
          Y isRunningUnderARM64Translation():
          → isX64ToARM64ImmediateAutoUpdate ← true
     c. status ← UpdateReady
     d. Emitir evento de cambio
     e. Verificar estado de actualización prioritaria (updatePriorityUpdateStatus)
```

### 3.5 Fase de Verificación de Prioridad

```
FUNCIÓN updatePriorityUpdateStatus():
  1. SI plataforma = Linux: RETORNAR
  2. Construir URL de verificación con versión actual y canal
  3. Enviar petición HEAD a central.github.com
  4. Examinar headers de respuesta:
     - SI 'x-prioritize-update' = 'true':
       → prioritizeUpdate ← true
     - SI 'x-prioritize-update-info-url' presente:
       → prioritizeUpdateInfoUrl ← valor del header
  5. Emitir evento de cambio de estado
  6. SI error: Registrar error silenciosamente (no interrumpir al usuario)
```

### 3.6 Fase de Notificación al Usuario (UI)

```
CUANDO UpdateStore.state cambia:
  1. SI status = UpdateReady Y canal ≠ development:
     a. Mostrar banner de actualización disponible
     b. Banner renderiza según condiciones:
        - SI isX64ToARM64ImmediateAutoUpdate:
          → Mensaje: "Versión optimizada disponible para ARM64"
        - SI isUpdateShowcaseVisible:
          → Mensaje: "Nuevas funcionalidades añadidas" + "Ver novedades"
        - SI prioritizeUpdate:
          → Mensaje: "Faltan actualizaciones importantes" (no dismissable)
          → Icono: Advertencia
        - POR DEFECTO:
          → Mensaje: "Actualización disponible" + "Ver novedades"
          → Icono: Descarga
```

### 3.7 Fase de Instalación

```
CUANDO usuario hace clic en "Reiniciar ahora":
  1. Renderer: updateStore.quitAndInstallUpdate()
  2. Enviar IPC síncrono 'will-quit' → Main: quitting ← true
  3. Enviar IPC 'quit-and-install-updates'
  4. Main: AppWindow.quitAndInstallUpdate()
  5. Main: autoUpdater.quitAndInstall()
  6. [Squirrel/Electron maneja la instalación y reinicio]

SI usuario intenta cerrar ventana durante descarga:
  1. Main: Interceptar evento 'close'
  2. SI isDownloadingUpdate Y (quitting O no es macOS):
     a. Prevenir cierre (e.preventDefault())
     b. Enviar IPC 'show-installing-update'
     c. Mostrar ventana (focus)
     d. Renderer: Mostrar diálogo InstallingUpdate
  3. Diálogo muestra:
     - "No cierre GitHub Desktop mientras se instala la actualización"
     - Botón "Salir de todos modos" (destructivo)
  4. SI usuario elige "Salir de todos modos":
     a. Enviar IPC síncrono 'will-quit-even-if-updating'
     b. Main: quittingEvenIfUpdating ← true
     c. Forzar cierre de ventana
```

### 3.8 Manejo de Eventos Squirrel (Windows)

```
FUNCIÓN handleSquirrelEvent(eventName):
  SEGÚN eventName:
    '--squirrel-install':
      1. Crear acceso directo en Menú Inicio
      2. Crear acceso directo en Escritorio
      3. Instalar CLI:
         a. Crear directorio bin
         b. Escribir github.bat (trampoline batch)
         c. Escribir github (trampoline shell para WSL)
         d. Agregar directorio bin al PATH del sistema
    
    '--squirrel-updated':
      1. Actualizar acceso directo en Menú Inicio
      2. SI existía acceso directo en Escritorio: Actualizarlo
      3. Reinstalar CLI (misma lógica que install)
    
    '--squirrel-uninstall':
      1. Eliminar acceso directo en Menú Inicio
      2. Eliminar acceso directo en Escritorio
      3. Eliminar directorio bin del PATH
    
    '--squirrel-obsolete':
      → No hacer nada (no-op)
```

### 3.9 Manejo de Errores

```
CUANDO autoUpdater emite 'error':
  1. Main: isDownloadingUpdate ← false
  2. Main: Enviar IPC 'auto-updater-error' con objeto Error
  3. Renderer: UpdateStore.onAutoUpdaterError(error):
     a. SI plataforma = Windows:
        - parsedError ← parseError(error)
        - SI parsedError no es null: error ← parsedError
     b. status ← UpdateNotAvailable
     c. SI NO es tarea de fondo:
        - Emitir error envuelto en ErrorWithMetadata
     d. Emitir evento de cambio de estado

FUNCIÓN parseError(error) [Solo Windows]:
  - SI "Can not find Squirrel": → "Falta dependencia de Squirrel"
  - SI error DNS con 'central.github.com': → "No se puede contactar servidor"
  - SI timeout de conexión: → "Timeout verificando actualizaciones"
  - DE LO CONTRARIO: → null (usar error original)
```

---

## 4. Información para Diagrama de Secuencia

### 4.1 Secuencia Principal: Verificación y Descarga de Actualización

```
Participantes:
  - Usuario
  - App.tsx (Renderer - Scheduler)
  - UpdateStore (Renderer - State Machine)
  - main-process-proxy (Renderer - IPC Proxy)
  - ipcMain (Main - IPC Handler)
  - AppWindow (Main - Update Manager)
  - autoUpdater (Electron - Update Engine)
  - central.github.com (Servidor de Actualizaciones)

Secuencia:
  App.tsx → App.tsx: performDeferredLaunchActions()
  App.tsx → UpdateStore: checkForUpdates(inBackground=true)
  UpdateStore → UpdateStore: getUpdatesUrl(skipGuidCheck=false)
  UpdateStore → UpdateStore: Construir URL con versión, canal, GUID
  UpdateStore → main-process-proxy: checkForUpdates(updatesUrl)
  main-process-proxy → ipcMain: IPC invoke 'check-for-updates'
  ipcMain → AppWindow: checkForUpdates(url)
  AppWindow → AppWindow: trySetUpdaterGuid(url)
  AppWindow → autoUpdater: setFeedURL({url})
  AppWindow → autoUpdater: checkForUpdates()
  autoUpdater → central.github.com: GET /api/deployments/desktop/desktop/latest?version=X&env=Y&guid=Z
  central.github.com → autoUpdater: Respuesta con información de actualización
  
  alt [Actualización disponible]
    autoUpdater → AppWindow: evento 'update-available'
    AppWindow → AppWindow: isDownloadingUpdate = true
    AppWindow → UpdateStore: IPC 'auto-updater-update-available'
    UpdateStore → UpdateStore: status = UpdateAvailable
    autoUpdater → autoUpdater: Descargar actualización automáticamente
    autoUpdater → AppWindow: evento 'update-downloaded'
    AppWindow → AppWindow: isDownloadingUpdate = false
    AppWindow → UpdateStore: IPC 'auto-updater-update-downloaded'
    UpdateStore → UpdateStore: generateReleaseSummary()
    UpdateStore → UpdateStore: status = UpdateReady
    UpdateStore → App.tsx: onDidChange(state)
    App.tsx → App.tsx: setUpdateBannerVisibility(true)
    App.tsx → Usuario: Mostrar banner "Actualización disponible"
  end
  
  alt [Sin actualización]
    autoUpdater → AppWindow: evento 'update-not-available'
    AppWindow → UpdateStore: IPC 'auto-updater-update-not-available'
    UpdateStore → UpdateStore: status = UpdateNotAvailable
  end
  
  alt [Error]
    autoUpdater → AppWindow: evento 'error'
    AppWindow → UpdateStore: IPC 'auto-updater-error'
    UpdateStore → UpdateStore: parseError() [Windows]
    UpdateStore → UpdateStore: status = UpdateNotAvailable
    UpdateStore → App.tsx: onError(error)
    App.tsx → Usuario: Mostrar error (si no es tarea de fondo)
  end
```

### 4.2 Secuencia: Instalación de Actualización

```
Participantes:
  - Usuario
  - UpdateAvailableBanner (Renderer - UI)
  - UpdateStore (Renderer - State Machine)
  - main-process-proxy (Renderer - IPC Proxy)
  - ipcMain (Main - IPC Handler)
  - AppWindow (Main - Window Manager)
  - autoUpdater (Electron - Update Engine)

Secuencia:
  Usuario → UpdateAvailableBanner: Clic "Reiniciar ahora"
  UpdateAvailableBanner → UpdateStore: quitAndInstallUpdate()
  UpdateStore → main-process-proxy: sendWillQuitSync()
  main-process-proxy → ipcMain: IPC sync 'will-quit'
  ipcMain → AppWindow: quitting = true
  ipcMain → main-process-proxy: return true
  UpdateStore → main-process-proxy: quitAndInstallUpdate()
  main-process-proxy → ipcMain: IPC 'quit-and-install-updates'
  ipcMain → AppWindow: quitAndInstallUpdate()
  AppWindow → autoUpdater: quitAndInstall()
  autoUpdater → autoUpdater: Cerrar app y aplicar actualización
  autoUpdater → Usuario: Reiniciar aplicación actualizada
```

### 4.3 Secuencia: Prevención de Cierre Durante Descarga

```
Participantes:
  - Usuario
  - AppWindow (Main - Window Manager)
  - InstallingUpdateDialog (Renderer - UI)
  - Dispatcher (Renderer - Actions)

Secuencia:
  Usuario → AppWindow: Intentar cerrar ventana
  AppWindow → AppWindow: Verificar isDownloadingUpdate
  
  alt [Descarga en progreso]
    AppWindow → AppWindow: e.preventDefault()
    AppWindow → InstallingUpdateDialog: IPC 'show-installing-update'
    AppWindow → AppWindow: this.show() (traer al frente)
    InstallingUpdateDialog → Usuario: "No cierre mientras se actualiza"
    
    alt [Usuario acepta esperar]
      InstallingUpdateDialog → InstallingUpdateDialog: Esperar cambio de estado
      UpdateStore → InstallingUpdateDialog: onDidChange()
      InstallingUpdateDialog → Dispatcher: quitApp(false)
    end
    
    alt [Usuario fuerza cierre]
      Usuario → InstallingUpdateDialog: Clic "Salir de todos modos"
      InstallingUpdateDialog → Dispatcher: quitApp(true)
      Dispatcher → main-process-proxy: sendWillQuitEvenIfUpdatingSync()
      main-process-proxy → AppWindow: quittingEvenIfUpdating = true
      AppWindow → AppWindow: Permitir cierre de ventana
    end
  end
```

### 4.4 Secuencia: Verificación de Actualización Prioritaria

```
Participantes:
  - UpdateStore (Renderer)
  - central.github.com (Servidor)
  - UpdateAvailableBanner (Renderer - UI)

Secuencia:
  UpdateStore → UpdateStore: onUpdateDownloaded() completo
  UpdateStore → central.github.com: HEAD /latest?version=X&env=Y
  central.github.com → UpdateStore: Headers de respuesta
  
  alt [Actualización prioritaria]
    UpdateStore → UpdateStore: Leer header 'x-prioritize-update: true'
    UpdateStore → UpdateStore: Leer header 'x-prioritize-update-info-url'
    UpdateStore → UpdateStore: prioritizeUpdate = true
    UpdateStore → UpdateAvailableBanner: emitDidChange()
    UpdateAvailableBanner → Usuario: Banner con icono de advertencia (no dismissable)
  end
```

### 4.5 Secuencia: Migración x64 → ARM64

```
Participantes:
  - UpdateStore (Renderer)
  - main-process-proxy (Renderer)
  - autoUpdater (Main)
  - central.github.com (Servidor)

Secuencia:
  UpdateStore → main-process-proxy: isRunningUnderARM64Translation()
  main-process-proxy → UpdateStore: true
  UpdateStore → UpdateStore: Modificar URL → /arm64/latest
  UpdateStore → UpdateStore: Falsear versión = "0.0.64"
  UpdateStore → autoUpdater: checkForUpdates(urlModificada)
  autoUpdater → central.github.com: GET /arm64/latest?version=0.0.64&env=Y
  central.github.com → autoUpdater: Paquete ARM64 nativo
  autoUpdater → UpdateStore: update-downloaded
  UpdateStore → UpdateStore: isX64ToARM64ImmediateAutoUpdate = true
  UpdateStore → Usuario: Banner "Versión optimizada para Apple Silicon/ARM64"
```

---

## 5. Información para Mapa Mental

### Estructura del Mapa Mental

```
MECANISMO DE ACTUALIZACIÓN DE GITHUB DESKTOP
│
├── 🔧 SUB-MECANISMOS
│   │
│   ├── 📋 Carga de Actualización (Inicialización)
│   │   ├── main.ts: Registro de handlers IPC
│   │   ├── app-window.ts: setupAutoUpdater()
│   │   ├── squirrel-updater.ts: handleSquirrelEvent() [Windows]
│   │   ├── app.tsx: performDeferredLaunchActions()
│   │   └── Scheduler: setInterval cada 4 horas
│   │
│   ├── 🔍 Búsqueda de Actualizaciones
│   │   ├── update-store.ts: checkForUpdates()
│   │   ├── Construcción de URL dinámica
│   │   │   ├── URL base: central.github.com
│   │   │   ├── Parámetro version (versión actual)
│   │   │   ├── Parámetro env (canal de release)
│   │   │   ├── Parámetro guid (UUID persistente)
│   │   │   └── Segmento /arm64/ (si aplica)
│   │   ├── Validaciones previas
│   │   │   ├── Excluir Linux
│   │   │   ├── Excluir canal development
│   │   │   ├── Verificar versión SO mínima
│   │   │   └── Verificar si ya hay actualización lista
│   │   └── Comunicación IPC: 'check-for-updates'
│   │
│   ├── 📥 Descarga de Actualizaciones
│   │   ├── Electron autoUpdater maneja descarga
│   │   ├── Descarga automática en segundo plano
│   │   ├── Flag isDownloadingUpdate para protección
│   │   ├── Eventos: update-available → update-downloaded
│   │   └── Paquetes Windows: .nupkg (full + delta)
│   │
│   ├── 📦 Instalación de Actualizaciones
│   │   ├── Trigger: quitAndInstallUpdate()
│   │   ├── IPC síncrono will-quit (prevenir race conditions)
│   │   ├── autoUpdater.quitAndInstall()
│   │   ├── Squirrel.Windows: Update.exe [Windows]
│   │   │   ├── --squirrel-install: Crear atajos + CLI
│   │   │   ├── --squirrel-updated: Actualizar atajos + CLI
│   │   │   └── --squirrel-uninstall: Limpiar atajos + CLI
│   │   ├── Prevención de cierre durante descarga
│   │   │   ├── Interceptar evento 'close'
│   │   │   ├── Diálogo InstallingUpdate
│   │   │   └── Opción "Salir de todos modos"
│   │   └── Reinstalación CLI (trampolines batch/shell)
│   │
│   └── ✅ Verificación de Autenticidad/Corrupción
│       ├── Delegada a Electron/Squirrel (no código custom)
│       ├── Windows: Validación de firma NuGet por Update.exe
│       ├── macOS: Validación codesign nativa
│       ├── HTTPS obligatorio para canal de comunicación
│       ├── GUID persistente (crypto.randomUUID)
│       └── Verificación de prioridad vía headers HTTP
│
├── 🎯 FUNCIONES PRINCIPALES
│   │
│   ├── UpdateStore (Máquina de Estados)
│   │   ├── checkForUpdates(inBackground, skipGuidCheck)
│   │   ├── quitAndInstallUpdate()
│   │   ├── getUpdatesUrl(skipGuidCheck)
│   │   ├── updatePriorityUpdateStatus()
│   │   ├── isUpdateShowcase()
│   │   ├── onAutoUpdaterError()
│   │   ├── onAutoUpdaterCheckingForUpdate()
│   │   ├── onAutoUpdaterUpdateAvailable()
│   │   ├── onAutoUpdaterUpdateNotAvailable()
│   │   └── onAutoUpdaterUpdateDownloaded()
│   │
│   ├── AppWindow (Proceso Main)
│   │   ├── setupAutoUpdater()
│   │   ├── checkForUpdates(url)
│   │   ├── quitAndInstallUpdate()
│   │   └── trySetUpdaterGuid(url)
│   │
│   ├── SquirrelUpdater (Windows)
│   │   ├── handleSquirrelEvent(eventName)
│   │   ├── handleInstalled()
│   │   ├── handleUpdated()
│   │   ├── installWindowsCLI()
│   │   ├── uninstallWindowsCLI()
│   │   └── spawnSquirrelUpdate(commands)
│   │
│   └── UI Components
│       ├── UpdateAvailable.render()
│       ├── UpdateAvailable.updateNow()
│       ├── UpdateAvailable.showReleaseNotes()
│       ├── InstallingUpdate.onQuitAnywayButtonClicked()
│       └── InstallingUpdate.onUpdateStateChanged()
│
├── 📌 CASOS DE USO
│   │
│   ├── CU-01: Verificación automática periódica
│   │   └── Cada 4 horas en producción/beta
│   │
│   ├── CU-02: Verificación manual por el usuario
│   │   └── Desde menú o botón de la UI
│   │
│   ├── CU-03: Descarga silenciosa en segundo plano
│   │   └── Sin intervención del usuario
│   │
│   ├── CU-04: Notificación de actualización disponible
│   │   └── Banner con opciones: ver notas / instalar
│   │
│   ├── CU-05: Showcase de nueva versión
│   │   └── Banner especial para releases con pretext (≤15 días)
│   │
│   ├── CU-06: Actualización prioritaria
│   │   └── Banner no dismissable con advertencia
│   │
│   ├── CU-07: Instalación y reinicio
│   │   └── Quit + Install + Relaunch
│   │
│   ├── CU-08: Prevención de cierre durante actualización
│   │   └── Diálogo modal de protección
│   │
│   ├── CU-09: Migración x64 → ARM64
│   │   └── Detección + descarga inmediata de build nativo
│   │
│   ├── CU-10: Despliegue escalonado (staggered rollout)
│   │   └── GUID persistente para cohortes
│   │
│   ├── CU-11: Manejo de eventos post-instalación [Windows]
│   │   └── Atajos + CLI trampolines + PATH
│   │
│   └── CU-12: Desinstalación limpia [Windows]
│       └── Eliminar atajos + CLI + PATH
│
├── 🔑 PUNTOS CLAVE
│   │
│   ├── Comunicación IPC fuertemente tipada
│   ├── Máquina de estados basada en eventos (pub/sub)
│   ├── Separación clara main process / renderer process
│   ├── Verificación de versión mínima de SO
│   ├── Soporte multi-arquitectura (x64, arm64)
│   ├── IPC síncrono para operaciones críticas (quit)
│   ├── Feature flags para control de funcionalidades
│   ├── Despliegue escalonado basado en GUID
│   ├── Priorización de actualizaciones controlada por servidor
│   ├── Delta updates para reducir ancho de banda [Windows]
│   ├── Prevención de corrupción durante cierre
│   └── Notas de versión categorizadas y temporizadas
│
└── 🏗️ ELEMENTOS ARQUITECTÓNICOS
    │
    ├── Patrón Observer (event-kit Emitter)
    ├── Patrón Proxy (main-process-proxy)
    ├── Patrón State Machine (UpdateStatus)
    ├── Patrón Singleton (UpdateStore)
    ├── Patrón Strategy (por plataforma)
    ├── Patrón Facade (autoUpdater de Electron)
    ├── IPC Bridge (ipc-shared.ts tipado)
    └── Feature Flags (feature-flag.ts)
```

---

## 6. Mejores Prácticas Implementadas

### 6.1 Arquitectura y Diseño

| Práctica | Implementación | Ubicación |
|----------|---------------|-----------|
| **Separación de responsabilidades** | Proceso Main maneja autoUpdater, Renderer maneja UI y estado | `app-window.ts` vs `update-store.ts` |
| **Tipado fuerte en IPC** | Todos los canales IPC definidos con tipos en `ipc-shared.ts` | `app/src/lib/ipc-shared.ts` |
| **Máquina de estados explícita** | `UpdateStatus` enum con transiciones claras y predecibles | `update-store.ts:32-47` |
| **Patrón Observer** | `event-kit` Emitter para desacoplar emisores de consumidores | `update-store.ts` |
| **Inmutabilidad** | Estado `readonly` en interfaces React y tipos TypeScript | Todo el código |
| **Feature flags** | Control granular de funcionalidades por canal de release | `feature-flag.ts` |

### 6.2 Seguridad

| Práctica | Implementación | Ubicación |
|----------|---------------|-----------|
| **HTTPS obligatorio** | URL de actualización siempre usa `https://` | `dist-info.ts:138-145` |
| **Delegación de verificación de firma** | Squirrel/Electron manejan verificación criptográfica | Framework level |
| **Generación segura de GUID** | Usa `crypto.randomUUID()` (CSPRNG) | `get-updater-guid.ts:19` |
| **IPC síncrono para operaciones críticas** | `sendWillQuitSync()` evita race conditions en cierre | `main-process-proxy.ts:283-309` |
| **No importar ipcRenderer/ipcMain directamente** | Wrappers tipados previenen errores de canal | `ipc-shared.ts` |

### 6.3 Experiencia de Usuario

| Práctica | Implementación | Ubicación |
|----------|---------------|-----------|
| **Actualizaciones silenciosas** | Descarga en background sin interrumpir al usuario | `update-store.ts:200-224` |
| **Despliegue escalonado** | GUID persistente permite rollout gradual | `get-updater-guid.ts` |
| **Mensajes de error amigables** | Parser traduce errores técnicos a lenguaje usuario | `squirrel-error-parser.ts` |
| **Prevención de corrupción** | Bloquear cierre durante descarga/instalación | `app-window.ts:115-134` |
| **Actualizaciones prioritarias** | Server puede forzar actualización urgente vía headers | `update-store.ts:273-297` |
| **Showcase de versión** | Notas de versión con duración de 15 días y dismiss persistente | `update-store.ts:305-335` |
| **Notificación no intrusiva** | Banner en lugar de diálogo modal para actualizaciones normales | `update-available.tsx` |

### 6.4 Resiliencia y Tolerancia a Fallos

| Práctica | Implementación | Ubicación |
|----------|---------------|-----------|
| **Reintentos automáticos** | Verificación periódica cada 4 horas reintenta fallos anteriores | `app.tsx:375` |
| **Errores de background silenciosos** | Errores de verificación automática no interrumpen al usuario | `update-store.ts:173-176` |
| **Fallback graceful** | Si GUID falla, continúa sin GUID | `get-updater-guid.ts:18-21` |
| **Validación de plataforma** | Verificar SO compatible antes de intentar actualización | `app.tsx:621-637` |
| **Cancelación segura de quit** | `cancel-quitting` IPC revierte estado de cierre | `app-window.ts:110-113` |

### 6.5 Mantenibilidad

| Práctica | Implementación | Ubicación |
|----------|---------------|-----------|
| **Constantes globales de plataforma** | `__DARWIN__`, `__WIN32__`, `__LINUX__` para compilación condicional | `app-info.ts` |
| **URLs configurables** | `DESKTOP_E2E_UPDATES_URL` para pruebas E2E | `app-info.ts:28` |
| **Delta packages** | Solo para production/beta, reduce tamaño de descarga | `dist-info.ts:shouldMakeDelta()` |
| **Métodos de prueba** | `simulateUpdateReady()` y `simulateUpdateCheck()` para dev/test | `update-store.ts:339-374` |
| **Logging exhaustivo** | `log.info()` y `log.error()` en puntos críticos | Multiple archivos |

---

## 7. Recomendaciones para Implementación en C# y .NET 10

### 7.1 Arquitectura General Recomendada

```
┌─────────────────────────────────────────────────────────┐
│                 Aplicación C# / .NET 10                  │
│                                                          │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────┐  │
│  │ UpdateService│───▶│UpdateStateMgr │───▶│ UI Layer │  │
│  │ (Background) │    │(State Machine)│    │(WPF/MAUI)│  │
│  └──────────────┘    └───────────────┘    └──────────┘  │
│         │                    │                           │
│  ┌──────────────┐    ┌───────────────┐                  │
│  │ PlatformSvc  │    │ UpdateConfig  │                  │
│  │ (Strategy)   │    │ (appsettings) │                  │
│  └──────────────┘    └───────────────┘                  │
│         │                                                │
│  ┌──────────────┐                                       │
│  │SignatureVerif │                                       │
│  │ (Security)   │                                       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### 7.2 Componentes Recomendados

#### A) Servicio de Actualización (Background Service)

```
Equivalente a: UpdateStore + AppWindow.setupAutoUpdater()

Recomendación:
- Implementar como IHostedService o BackgroundService de .NET
- Usar Timer periódico (equivalente al setInterval de 4 horas)
- Ejecutar en thread separado del UI
- Usar HttpClient con HttpClientFactory para peticiones al servidor
- Implementar IDisposable para limpieza de recursos

Tecnologías sugeridas:
- Microsoft.Extensions.Hosting (BackgroundService)
- System.Threading.Timer o PeriodicTimer (.NET 6+)
- HttpClientFactory con policies de retry (Polly)
```

#### B) Máquina de Estados

```
Equivalente a: UpdateStatus enum + UpdateStore state transitions

Recomendación:
- Definir enum UpdateStatus con los mismos estados
- Implementar patrón State Machine con record types
- Usar INotifyPropertyChanged o IObservable<T> para notificaciones
- Considerar librería Stateless (https://github.com/dotnet-state-machine/stateless)
  para transiciones formales y validadas

Definición sugerida:
  public enum UpdateStatus
  {
      NotChecked,
      CheckingForUpdates,
      UpdateAvailable,
      Downloading,
      UpdateReady,
      Installing,
      Error
  }

  public record UpdateState(
      UpdateStatus Status,
      DateTime? LastSuccessfulCheck,
      IReadOnlyList<ReleaseSummary>? NewReleases,
      bool PrioritizeUpdate,
      string? PriorityInfoUrl,
      string? ErrorMessage
  );
```

#### C) Comunicación entre Capas

```
Equivalente a: IPC channels (ipc-shared.ts)

Recomendación para WPF:
- Usar MediatR para comunicación desacoplada (patrón Mediator)
- Usar IMessenger de CommunityToolkit.Mvvm
- Alternativamente, eventos fuertemente tipados con delegates

Recomendación para MAUI:
- WeakReferenceMessenger de CommunityToolkit.Mvvm
- MessagingCenter (built-in pero legacy)

Recomendación para aplicación de consola/servicio:
- System.Threading.Channels para comunicación productor/consumidor
- IObservable<T> con System.Reactive (Rx.NET)
```

#### D) Descarga y Verificación

```
Equivalente a: autoUpdater.checkForUpdates() + descarga automática

Recomendación:
- HttpClient con progress reporting (IProgress<T>)
- Implementar verificación de firma explícitamente (no delegarla)
- Usar System.Security.Cryptography para verificación SHA256/RSA
- Descargar a directorio temporal con nombre aleatorio
- Verificar hash ANTES de mover al directorio de instalación

Flujo recomendado:
  1. GET /api/updates/check?version=X&channel=Y&guid=Z
  2. Respuesta: { version, url, sha256, signature, releaseNotes }
  3. Descargar archivo a %TEMP%\{guid}\update.zip
  4. Calcular SHA256 del archivo descargado
  5. Comparar con hash del servidor
  6. Verificar firma digital (RSA/ECDSA)
  7. SI todo válido → mover a directorio de staging
  8. Notificar a UI: UpdateReady
```

#### E) Instalación

```
Equivalente a: autoUpdater.quitAndInstall()

Recomendación:
- Crear proceso externo para la instalación (updater.exe separado)
- El proceso principal notifica al updater y se cierra
- El updater espera que el proceso principal termine
- El updater aplica la actualización (reemplazar archivos)
- El updater reinicia la aplicación
- Usar named pipes o archivos temporales para comunicación

Tecnologías sugeridas para Windows:
- MSIX: Usar Windows.Management.Deployment para auto-actualización
- ClickOnce: Built-in con soporte limitado
- Squirrel.Windows para .NET: Clowd.Squirrel (fork mantenido)
- Custom: Process.Start() con actualización fuera de proceso

Tecnologías sugeridas cross-platform:
- NetSparkleUpdater (https://github.com/NetSparkleUpdater/NetSparkle)
- AutoUpdater.NET (https://github.com/ravibpatel/AutoUpdater.NET)
- Velopack (https://github.com/velopack/velopack) - sucesor moderno de Squirrel
```

#### F) Verificación de Autenticidad

```
Equivalente a: Verificación delegada a Electron/Squirrel

Recomendación MEJORADA (implementar explícitamente):
- Generar par de claves RSA/ECDSA para firmar releases
- Publicar clave pública embebida en la aplicación
- Firmar cada paquete de actualización con clave privada
- Verificar firma antes de instalar

Implementación sugerida:
  public interface IUpdateVerifier
  {
      Task<bool> VerifyHashAsync(string filePath, string expectedHash);
      Task<bool> VerifySignatureAsync(string filePath, byte[] signature, byte[] publicKey);
  }

  // Usar:
  // - SHA256 para integridad
  // - RSA-PSS o ECDSA para autenticidad
  // - Certificate pinning para comunicación con servidor
```

#### G) Despliegue Escalonado

```
Equivalente a: get-updater-guid.ts + parámetro guid en URL

Recomendación:
- Generar GUID persistente en primera ejecución
- Almacenar en directorio de datos de usuario
- Enviar como header o parámetro al servidor de actualizaciones
- Servidor decide si el cliente debe recibir actualización
- Permite rollback parcial si se detectan problemas

Almacenamiento sugerido:
  // Windows: %LOCALAPPDATA%\MiApp\.update-id
  // macOS: ~/Library/Application Support/MiApp/.update-id  
  // Linux: ~/.config/MiApp/.update-id
  
  var path = Path.Combine(
      Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
      "MiApp", ".update-id");
```

### 7.3 Configuración Recomendada

```json
// appsettings.json
{
  "UpdateSettings": {
    "UpdateServerBaseUrl": "https://updates.miapp.com/api",
    "CheckIntervalHours": 4,
    "EnableAutoUpdate": true,
    "ReleaseChannel": "production",
    "EnableStaggeredRollout": true,
    "MaxRetryAttempts": 3,
    "RetryDelaySeconds": 30,
    "DownloadTimeoutMinutes": 15,
    "PublicSigningKey": "BASE64_ENCODED_PUBLIC_KEY",
    "MinimumSupportedOsVersion": "10.0.17763",
    "AllowDeltaUpdates": true
  }
}
```

### 7.4 Patrones de Diseño Recomendados

| Patrón | Uso | Equivalente en GitHub Desktop |
|--------|-----|-------------------------------|
| **BackgroundService** | Verificación periódica | `setInterval(checkForUpdates, 4h)` |
| **State Machine** | Gestión de estado de actualización | `UpdateStatus` enum |
| **Strategy** | Lógica específica por plataforma | `__DARWIN__` / `__WIN32__` checks |
| **Observer** | Notificaciones de cambio de estado | `event-kit` Emitter |
| **Mediator** | Comunicación entre capas | IPC channels |
| **Factory** | Creación de verificadores/instaladores por plataforma | Compilación condicional |
| **Decorator** | Retry, logging, timeout en HTTP | `ErrorWithMetadata` wrapping |
| **Template Method** | Flujo general con pasos customizables | `checkForUpdates → download → verify → install` |

### 7.5 Estructura de Proyecto Recomendada

```
MiApp.Updater/
├── MiApp.Updater.Core/              # Lógica de negocio
│   ├── Models/
│   │   ├── UpdateStatus.cs          # Enum de estados
│   │   ├── UpdateState.cs           # Record del estado completo
│   │   ├── ReleaseSummary.cs        # Notas de versión
│   │   └── UpdateConfiguration.cs   # Configuración
│   ├── Services/
│   │   ├── IUpdateService.cs        # Interfaz principal
│   │   ├── UpdateService.cs         # Implementación
│   │   ├── IUpdateChecker.cs        # Verificar actualizaciones
│   │   ├── IUpdateDownloader.cs     # Descargar actualizaciones
│   │   ├── IUpdateInstaller.cs      # Instalar actualizaciones
│   │   └── IUpdateVerifier.cs       # Verificar integridad/firma
│   ├── StateMachine/
│   │   ├── UpdateStateMachine.cs    # Máquina de estados
│   │   └── UpdateTransitions.cs     # Transiciones válidas
│   └── Events/
│       ├── UpdateStateChangedEvent.cs
│       └── UpdateErrorEvent.cs
│
├── MiApp.Updater.Platform/          # Implementaciones por plataforma
│   ├── Windows/
│   │   ├── WindowsUpdateInstaller.cs
│   │   ├── WindowsShortcutManager.cs
│   │   └── WindowsCliInstaller.cs
│   ├── MacOS/
│   │   ├── MacOSUpdateInstaller.cs
│   │   └── MacOSArchitectureDetector.cs
│   └── Linux/
│       └── LinuxUpdateInstaller.cs
│
├── MiApp.Updater.Security/          # Verificación de seguridad
│   ├── HashVerifier.cs              # SHA256 verification
│   ├── SignatureVerifier.cs         # RSA/ECDSA verification
│   └── GuidManager.cs              # GUID persistente
│
├── MiApp.Updater.UI/                # Componentes de UI
│   ├── ViewModels/
│   │   ├── UpdateBannerViewModel.cs
│   │   └── InstallingUpdateViewModel.cs
│   └── Views/
│       ├── UpdateBanner.xaml        # Banner de actualización
│       └── InstallingUpdateDialog.xaml
│
└── MiApp.Updater.Tests/             # Tests unitarios e integración
    ├── UpdateServiceTests.cs
    ├── StateMachineTests.cs
    ├── VerifierTests.cs
    └── InstallerTests.cs
```

### 7.6 Dependencias NuGet Recomendadas

| Paquete | Versión Mínima | Propósito |
|---------|---------------|-----------|
| `Microsoft.Extensions.Hosting` | 10.0+ | BackgroundService |
| `Microsoft.Extensions.Http` | 10.0+ | HttpClientFactory |
| `Microsoft.Extensions.Options` | 10.0+ | Configuración tipada |
| `Polly` | 8.0+ | Retry policies |
| `Stateless` | 5.0+ | State machine (opcional) |
| `CommunityToolkit.Mvvm` | 8.0+ | MVVM + Messaging |
| `Serilog` | 4.0+ | Logging estructurado |
| `Velopack` | 0.0.500+ | Motor de actualización (alternativa a Squirrel) |

### 7.7 Consideraciones de Seguridad Específicas para .NET

1. **Certificate Pinning**: Implementar `HttpClientHandler.ServerCertificateCustomValidationCallback` para validar el certificado del servidor de actualizaciones.

2. **Firma de código**: Usar `SignTool.exe` o `signtool` para firmar el ejecutable y el instalador. En .NET 10, considerar `System.Security.Cryptography.X509Certificates` para validación en runtime.

3. **Protección contra downgrade**: Almacenar la versión instalada en un registro protegido y rechazar actualizaciones con versión inferior.

4. **Integridad del updater**: El ejecutable del actualizador externo debe estar firmado y verificado antes de ejecutarlo.

5. **Permisos mínimos**: El servicio de actualización debe ejecutar con los permisos mínimos necesarios. Solo elevar permisos (UAC) cuando se requiera escribir en `Program Files`.

6. **Aislamiento de proceso**: La descarga y verificación deben ejecutarse en un proceso aislado o AppDomain para prevenir manipulación en memoria.

### 7.8 Ejemplo de Interfaz Principal

```csharp
/// <summary>
/// Servicio principal de actualización.
/// Equivalente conceptual al UpdateStore de GitHub Desktop.
/// </summary>
public interface IUpdateService : IDisposable
{
    /// <summary>Estado actual de la actualización.</summary>
    UpdateState CurrentState { get; }
    
    /// <summary>Evento disparado cuando cambia el estado.</summary>
    event EventHandler<UpdateStateChangedEventArgs>? StateChanged;
    
    /// <summary>Evento disparado cuando ocurre un error.</summary>
    event EventHandler<UpdateErrorEventArgs>? ErrorOccurred;
    
    /// <summary>Verificar si hay actualizaciones disponibles.</summary>
    /// <param name="background">Si es true, errores no se muestran al usuario.</param>
    /// <param name="skipStaggerCheck">Si es true, ignora el despliegue escalonado.</param>
    Task CheckForUpdatesAsync(bool background = true, bool skipStaggerCheck = false);
    
    /// <summary>Descargar la actualización disponible.</summary>
    /// <param name="progress">Reporte de progreso de descarga.</param>
    Task DownloadUpdateAsync(IProgress<double>? progress = null);
    
    /// <summary>Verificar integridad y autenticidad del paquete descargado.</summary>
    Task<bool> VerifyUpdateAsync();
    
    /// <summary>Instalar la actualización y reiniciar la aplicación.</summary>
    Task InstallAndRestartAsync();
    
    /// <summary>Verificar si la actualización actual es prioritaria.</summary>
    Task<bool> IsPriorityUpdateAsync();
}
```

### 7.9 Checklist de Implementación

- [ ] **Infraestructura Base**
  - [ ] Definir modelos de estado (`UpdateStatus`, `UpdateState`, `ReleaseSummary`)
  - [ ] Implementar `UpdateStateMachine` con transiciones validadas
  - [ ] Configurar `BackgroundService` para verificación periódica
  - [ ] Implementar `HttpClientFactory` con políticas Polly

- [ ] **Búsqueda de Actualizaciones**
  - [ ] Implementar `IUpdateChecker` con construcción dinámica de URL
  - [ ] Agregar soporte para GUID persistente (despliegue escalonado)
  - [ ] Implementar validación de versión mínima de SO
  - [ ] Agregar soporte multi-arquitectura

- [ ] **Descarga**
  - [ ] Implementar `IUpdateDownloader` con reportes de progreso
  - [ ] Soportar descarga en background con cancelación
  - [ ] Implementar delta updates (opcional)
  - [ ] Prevenir cierre de aplicación durante descarga

- [ ] **Verificación de Seguridad**
  - [ ] Implementar verificación SHA256
  - [ ] Implementar verificación de firma digital (RSA/ECDSA)
  - [ ] Implementar certificate pinning
  - [ ] Agregar protección contra downgrade

- [ ] **Instalación**
  - [ ] Implementar proceso externo de actualización
  - [ ] Manejar eventos post-instalación (atajos, PATH, CLI)
  - [ ] Implementar rollback en caso de fallo
  - [ ] Soportar reinicio automático

- [ ] **UI**
  - [ ] Implementar banner de actualización disponible
  - [ ] Implementar diálogo de instalación en progreso
  - [ ] Implementar showcase de notas de versión
  - [ ] Implementar notificación de actualización prioritaria

- [ ] **Resiliencia**
  - [ ] Implementar reintentos automáticos con backoff exponencial
  - [ ] Manejar errores de red con mensajes amigables
  - [ ] Implementar logging exhaustivo
  - [ ] Agregar telemetría de actualizaciones

- [ ] **Testing**
  - [ ] Tests unitarios para máquina de estados
  - [ ] Tests unitarios para verificación de seguridad
  - [ ] Tests de integración para flujo completo
  - [ ] Métodos de simulación para desarrollo/testing

---

## Apéndice: Referencia Rápida de Archivos

| Archivo | Líneas Clave | Descripción |
|---------|-------------|-------------|
| `update-store.ts:32-47` | Enum `UpdateStatus` | Definición de estados |
| `update-store.ts:99-145` | Event handlers | Manejo de eventos autoUpdater |
| `update-store.ts:200-224` | `checkForUpdates()` | Punto de entrada de verificación |
| `update-store.ts:226-262` | `getUpdatesUrl()` | Construcción de URL con ARM64 |
| `update-store.ts:273-297` | `updatePriorityUpdateStatus()` | Verificación de prioridad |
| `update-store.ts:305-335` | `isUpdateShowcase()` | Lógica de showcase |
| `app-window.ts:115-134` | Close handler | Prevención de cierre |
| `app-window.ts:396-433` | `setupAutoUpdater()` | Registro de listeners |
| `app-window.ts:435-447` | `checkForUpdates()` / `quitAndInstall()` | Acciones principales |
| `main.ts:140-151` | Squirrel startup | Manejo de eventos al inicio |
| `main.ts:514-520` | IPC handlers | Registro de handlers |
| `squirrel-updater.ts:1-173` | Completo | Ciclo de vida Squirrel |
| `get-updater-guid.ts:1-25` | Completo | GUID persistente |
| `squirrel-error-parser.ts:1-37` | Completo | Parser de errores |
| `update-available.tsx:1-186` | Completo | Banner de UI |
| `installing-update.tsx:1-93` | Completo | Diálogo de instalación |
| `app.tsx:206-212` | Constante intervalo | 4 horas |
| `app.tsx:305-327` | State listener | Manejo de cambios de estado |
| `app.tsx:360-398` | `performDeferredLaunchActions()` | Scheduler |
| `app.tsx:617-640` | `checkForUpdates()` | Validaciones pre-check |
| `dist-info.ts:138-145` | `getUpdatesURL()` | URL de actualización |
| `feature-flag.ts:61-70` | ARM64 feature flag | Control de migración |
| `ipc-shared.ts` | Canales IPC | Definiciones tipadas |
