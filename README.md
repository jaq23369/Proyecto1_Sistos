# Proyecto 1 — Chat TCP con Protobuf

**CC3064 Sistemas Operativos**

Aplicación de chat en tiempo real sobre TCP con arquitectura cliente-servidor. Usa [Protocol Buffers](https://protobuf.dev/) para la serialización de mensajes y un protocolo de framing propio de 5 bytes. El servidor maneja múltiples clientes de forma concurrente mediante hilos POSIX.

---

## Tabla de contenidos

1. [Estructura del repositorio](#estructura-del-repositorio)
2. [Dependencias](#dependencias)
3. [Compilación](#compilación)
4. [Uso](#uso)
5. [Protocolo de comunicación](#protocolo-de-comunicación)
6. [Arquitectura interna](#arquitectura-interna)
7. [Comandos del cliente](#comandos-del-cliente)
8. [Estados de usuario](#estados-de-usuario)
9. [Notas de compatibilidad](#notas-de-compatibilidad)

---

## Estructura del repositorio

```
Proyecto1_Sistos/
├── client/
│   └── client.cpp          # Lógica del cliente (entrada de usuario + hilo receptor)
├── server/
│   └── server.cpp          # Lógica del servidor (hilo por cliente + monitor de inactividad)
├── common/
│   └── framing.h           # Funciones de framing TCP compartidas (send/recv)
├── Makefile                # Reglas de compilación
└── .gitignore
```

> Los archivos `.proto` viven en el repositorio hermano `../Simple-Chat-Protocol/protos/`.
> El `Makefile` los busca allí y genera los headers/fuentes de C++ en `generated/`.

---

## Dependencias

| Herramienta | Versión mínima | Notas |
|---|---|---|
| `g++` | C++17 | Cualquier GCC/Clang moderno |
| `protobuf` | 3.x | Incluye `protoc` y la librería de runtime |
| `pkg-config` | cualquiera | Opcional; el Makefile usa fallback a `-lprotobuf` |
| `pthread` | — | Incluido en glibc / macOS SDK |

### Instalación de dependencias

**Ubuntu / Debian**
```bash
sudo apt install g++ protobuf-compiler libprotobuf-dev pkg-config
```

**macOS (Homebrew)**
```bash
brew install protobuf pkg-config
```

---

## Compilación

```bash
# Compilar servidor y cliente (también genera los archivos protobuf)
make

# Solo generar código protobuf
make protos

# Solo compilar el servidor
make chat_server

# Solo compilar el cliente
make chat_client

# Limpiar binarios y código generado
make clean
```

Tras compilar exitosamente se crean los binarios `chat_server` y `chat_client` en la raíz del proyecto.

---

## Uso

### Iniciar el servidor

```bash
./chat_server <puerto>
```

Ejemplo:
```bash
./chat_server 8080
```

El servidor imprime en consola cada conexión entrante, registro de usuarios y cambios de estado por inactividad.

### Conectar un cliente

```bash
./chat_client <username> <server_ip> <server_port>
```

Ejemplo:
```bash
./chat_client alice 127.0.0.1 8080
```

Al conectarse el cliente muestra un banner con su nombre de usuario e IP, y la lista de comandos disponibles.

---

## Protocolo de comunicación

### Framing TCP (`common/framing.h`)

Cada mensaje se transmite con el siguiente encabezado de **5 bytes** seguido del payload serializado con protobuf:

```
┌────────────┬──────────────────────────┬──────────────────────┐
│  type (1B) │  length big-endian (4B)  │  payload (N bytes)   │
└────────────┴──────────────────────────┴──────────────────────┘
```

- **`type`**: identifica el tipo de mensaje (ver tabla abajo).
- **`length`**: tamaño del payload en bytes (orden de red, big-endian).
- **`payload`**: mensaje serializado con Protocol Buffers.

### Tipos de mensaje

#### Cliente → Servidor

| Tipo | Nombre protobuf | Descripción |
|------|----------------|-------------|
| `1`  | `Register` | Registrar usuario con nombre e IP |
| `2`  | `MessageGeneral` | Enviar mensaje al chat general (broadcast) |
| `3`  | `MessageDM` | Enviar mensaje directo a otro usuario |
| `4`  | `ChangeStatus` | Cambiar el estado del usuario |
| `5`  | `ListUsers` | Solicitar lista de usuarios conectados |
| `6`  | `GetUserInfo` | Solicitar información de un usuario específico |
| `7`  | `Quit` | Desconectarse del servidor |

#### Servidor → Cliente

| Tipo | Nombre protobuf | Descripción |
|------|----------------|-------------|
| `10` | `ServerResponse` | Respuesta genérica (código de estado + mensaje) |
| `11` | `AllUsers` | Lista de usuarios con sus estados |
| `12` | `ForDm` | Entrega de un mensaje directo |
| `13` | `BroadcastDelivery` | Entrega de un mensaje del chat general |
| `14` | `GetUserInfoResponse` | Info de usuario (IP, nombre, estado) |

### Archivos `.proto` utilizados

```
Simple-Chat-Protocol/protos/
├── common.proto
├── cliente-side/
│   ├── change_status.proto
│   ├── get_user_info.proto
│   ├── list_users.proto
│   ├── message_dm.proto
│   ├── message_general.proto
│   ├── quit.proto
│   └── register.proto
└── server-side/
    ├── all_users.proto
    ├── broadcast_messages.proto
    ├── for_dm.proto
    ├── get_user_info_response.proto
    └── server_response.proto
```

---

## Arquitectura interna

### Servidor (`server/server.cpp`)

- **Un hilo por cliente** (`handle_client`): atiende toda la comunicación de un cliente desde su registro hasta su desconexión.
- **Hilo de inactividad** (`inactivity_monitor`): se ejecuta cada 10 segundos y cambia a `INVISIBLE` a cualquier usuario que lleve más de **60 segundos** sin enviar mensajes.
- **Tabla de usuarios** (`g_users`): `unordered_map<string, ClientInfo>` protegido con `g_users_mtx`. O(1) en búsqueda y eliminación.
- **Mutex por socket** (`g_sock_mtxs`): evita condiciones de carrera al escribir simultáneamente en el mismo stream TCP desde distintos hilos (broadcasts). El payload se serializa una sola vez y se envía a todos los clientes con `safe_send_raw`.

```
main()
  └─ accept() loop
       └─ thread: handle_client(fd)
            ├── case 1: register
            ├── case 2: broadcast → snapshot() → safe_send_raw(todos)
            ├── case 3: DM → safe_send(target)
            ├── case 4: cambio de estado
            ├── case 5: listar usuarios
            ├── case 6: info de usuario
            └── case 7: quit

thread: inactivity_monitor()  (detached)
```

### Cliente (`client/client.cpp`)

- **Hilo principal**: lee líneas de `stdin` y despacha comandos al servidor.
- **Hilo receptor** (`receiver_thread`): escucha mensajes entrantes del servidor y los imprime con colores ANSI.
- Ambos hilos comparten `g_running` (atomic bool) para coordinar el cierre.
- `g_send_mtx` serializa los envíos al servidor; `g_print_mtx` evita que los mensajes impresos por ambos hilos se entrelacen.

---

## Comandos del cliente

| Comando | Descripción |
|---------|-------------|
| `<mensaje>` | Envía al chat general (todos los usuarios conectados) |
| `/dm <usuario> <mensaje>` | Envía un mensaje directo al usuario indicado |
| `/status <estado>` | Cambia tu estado: `Activo`, `Ocupado` o `AFK` |
| `/list` | Muestra todos los usuarios conectados y su estado |
| `/info <usuario>` | Muestra IP, nombre y estado de un usuario |
| `/help` | Muestra la lista de comandos disponibles |
| `/quit` | Envía mensaje de salida y cierra la conexión |

---

## Estados de usuario

| Estado | Valor protobuf | Insignia | Descripción |
|--------|---------------|----------|-------------|
| `Activo` | `ACTIVE` | `[+]` verde | Usuario activo |
| `Ocupado` | `DO_NOT_DISTURB` | `[-]` amarillo | No molestar |
| `AFK` | `INVISIBLE` | `[ ]` tenue | Ausente / invisible |

> El servidor cambia automáticamente el estado a `AFK` (`INVISIBLE`) tras **60 segundos** de inactividad. El usuario puede restaurarlo manualmente con `/status Activo`.

---

## Notas de compatibilidad

- **Linux**: usa `MSG_NOSIGNAL` en `send()` para evitar `SIGPIPE` en escrituras sobre sockets cerrados.
- **macOS**: `MSG_NOSIGNAL` no existe; el código define su valor a `0` y usa `SO_NOSIGPIPE` a nivel de socket como alternativa.
- El `Makefile` detecta automáticamente los flags de compilación de protobuf mediante `pkg-config`, con fallback a `-lprotobuf` si no está disponible.
