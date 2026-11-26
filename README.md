# Obsidian MCP for Home Assistant

[English](#english) · [Español](#español) · [Releases](https://github.com/sergey08/obsidian-mcp-integration/releases)

## English

Read, search and edit Markdown notes in your Obsidian vault from ChatGPT or another compatible MCP client. The integration runs in Home Assistant Core. The current stable release is **v0.3.1**.

[Release notes for v0.3.1](docs/releases/v0.3.1.md) describe the annotations now available for all nine MCP tools.

### Choose how Home Assistant sees your vault

| Your setup | Storage mode | Where you enter the path | Who synchronizes the files |
| --- | --- | --- | --- |
| The Markdown files already exist under Home Assistant's `/share`. You may edit them over a network share, or another process already keeps them synchronized. | `local` | **Local vault**. Example `/share/obsidian-vault`. | Your existing process, if any. Leave **Enable LiveSync** off. |
| Obsidian already uses **Self-hosted LiveSync** and you want Home Assistant to mirror the same CouchDB database. | `livesync` | Leave both paths blank for the integration-managed vault at `/config/obsidian_mcp/vault`. For an existing shared Markdown vault under `/share`, enter it in **Shared LiveSync vault** and enable LiveSync. | The integration runs LiveSync CLI. CouchDB and the Obsidian plugin run separately. |

A folder on your phone or PC is only usable in local mode if Home Assistant can see the Markdown files under its own `/share`. Install **Self-hosted LiveSync** in the Obsidian clients if you want CouchDB replication. You can add ChatGPT access to either storage mode independently.

### Install and set up

1. Open **HACS / Marketplace**, search for **Obsidian MCP**, and download the integration from the catalog. Restart Home Assistant if prompted. Requires Home Assistant **2026.9.0 or later**.

   **Alternative (custom repository):** If the integration does not appear in the catalog yet, open **HACS / Marketplace → Custom repositories**, add [this repository](https://github.com/sergey08/obsidian-mcp-integration) as **Integration**, then search for **Obsidian MCP** and download it.
2. Open **Settings → Devices & services → Add integration → Obsidian MCP**. Choose a mode from the table and leave the OpenAI tunnel and remote MCP disabled until you have configured them.
3. For **local** mode, create or mount the folder under `/share` first, make it writable by Home Assistant Core, and enter its path in **Local vault**. For **LiveSync** mode, start with both path fields empty and LiveSync disabled, then follow the setup below.
4. Open **Configure** to import credentials or run checks. Open **Reconfigure** in the integration's menu to change paths, enable services or set the tunnel ID.

### Set up a synchronized vault

1. Set up CouchDB and verify that Self-hosted LiveSync works in Obsidian. On Home Assistant OS you can use a separate app such as [Obsidian Sync Server SSL](https://github.com/alexbelgium/hassio-addons/tree/master/obsidian_syncserver_ssl). Use a certificate trusted by Home Assistant Core and a CouchDB account with enough rights to replicate the target database.
2. Obtain a **LiveSync CLI settings JSON** for that database. Reuse a working CLI configuration or follow the [upstream CLI setup guide](https://github.com/vrtmrz/obsidian-livesync/tree/1.0.39/src/apps/cli) to prepare one from the setup information of your Obsidian client. In **Configure → Import LiveSync settings**, paste the JSON. The form accepts CLI JSON, saves it to `/config/obsidian_mcp/settings/livesync-operativo.json` and protects an existing file from overwrite. Keep the JSON and any setup URI private, especially when the vault uses encryption or path obfuscation.
3. Download `livesync-ha-amd64.tar.gz` (x86-64) or `livesync-ha-arm64.tar.gz` (AArch64) and the matching `.sha256` from the [release](https://github.com/sergey08/obsidian-mcp-integration/releases/tag/v0.3.1). These bundles target Alpine-based Home Assistant Core. The integration expects `/config/obsidian_mcp/bin/livesync-cli`. On a different platform, provision a compatible LiveSync CLI wrapper with the same path, arguments and `LIVESYNC_DB_PATH` environment variable.
4. From a shell with write access to the Home Assistant configuration directory, verify and extract the downloaded archive. For AMD64, from the folder holding the two downloads:

   ```sh
   sha256sum -c livesync-ha-amd64.tar.gz.sha256 &&
     tar -xzf livesync-ha-amd64.tar.gz -C /config/obsidian_mcp
   ```

   Substitute `arm64` in both filenames for ARM64, or replace `/config` if your installation uses a different configuration directory. Fresh installs create the destination when the integration first loads. For upgrades, stop the LiveSync process before replacing its runtime.
5. In **Configure → Check CouchDB**, verify access. In **Reconfigure**, select `livesync`, enable LiveSync, and use the default **60-second** interval unless you need another value. Leave both paths empty for the managed vault. Only use **Shared LiveSync vault** for an existing, populated Markdown folder under `/share`, with any previous writer stopped. Start with a test vault and confirm a new disposable note reaches another Obsidian client.

### Connect ChatGPT or another MCP client

- **ChatGPT.** Create or reuse a [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels) in your OpenAI Platform organization and obtain its tunnel ID and a runtime API key authorized for it. Select a verified, platform-compatible executable from the [official tunnel-client releases](https://github.com/openai/tunnel-client/releases). Place it at `/config/obsidian_mcp/tunnel/bin/tunnel-client` and make it executable. Use **Configure → Save OpenAI tunnel runtime key** to store the key. In **Reconfigure**, enter the ID and enable **Private OpenAI tunnel**. Select that tunnel in your ChatGPT workspace's custom MCP app setup, where available. Test by listing notes. The tunnel uses outbound connectivity and a private local MCP listener.
- **Other Streamable HTTP clients.** In **Reconfigure**, enable **Authenticated remote MCP** and provide a unique, random Bearer token of at least 32 characters. Connect the client to `https://<your-ha-host>/api/obsidian_mcp/mcp` over trusted HTTPS with `Authorization: Bearer <token>`. Leave this option off if you only use the OpenAI tunnel. With remote MCP enabled, leaving the token field blank in **Reconfigure** retains the current token.
- The nine tools cover listing, searching, reading, reading with revision, obtaining a revision, writing, appending, renaming and deleting. Deleted notes go to recoverable private trash under `/config/obsidian_mcp/mcp-trash`.

### Configure, troubleshoot and recover

| Configure menu | When to use it |
| --- | --- |
| **Show vault and service status** | Check the active vault and whether LiveSync and the tunnel processes are running. Confirm replication and tunnel connectivity with a real request. |
| **Check the CouchDB connection** | Check database access and credentials from Core. Use a test note to confirm replication and encryption compatibility. |
| **Import LiveSync settings** | Provision a CLI JSON during installation or recovery. |
| **Save OpenAI tunnel runtime key** | Provision a runtime key during installation or recovery. |

If a note fails to sync, confirm the active vault path and LiveSync status, check CouchDB, and inspect Home Assistant and Obsidian LiveSync logs for TLS, configuration or replication errors. If ChatGPT cannot list notes, check the executable and its permissions, the runtime key's permissions, and the tunnel ID and workspace association. Only share redacted diagnostics.

### Migrate from the Home Assistant app

1. Record the active vault path and tunnel ID. Back up the vault, CouchDB, app settings, recoverable trash and LiveSync state. On the new integration, first test with the default isolated vault and optional services off.
2. While the app still runs, provision the CLI runtime, copy the private LiveSync settings, and copy the verified tunnel client and runtime key to the corresponding integration paths. Use **Check the CouchDB connection** before the cutover.
3. Stop the old app before enabling the integration's LiveSync or reusing its tunnel ID. For a shared vault, use the same path in **Shared LiveSync vault**. For an app-managed vault, copy the Markdown files to the integration-managed vault after stopping the app. On an existing installation with compatible CLI/database formats, the local LiveSync database can be copied while the old CLI is stopped to preserve replication state. Retain the original data for recovery.
4. Enable LiveSync and the private tunnel in **Reconfigure**. Verify a read and a test write from ChatGPT, then open the new note in an Obsidian client. Keep the old app stopped with manual startup during the transition.
5. To roll back, disable the integration first, then restart the app. If you switched to a copy of the vault, reconcile edits made since switching to the integration before returning to the original folder.

## Español

Lee, busca y edita notas Markdown de tu bóveda de Obsidian desde ChatGPT u otro cliente MCP compatible. La integración se ejecuta dentro de Home Assistant Core. La versión estable actual es **v0.3.1**.

Las [notas de v0.3.1](docs/releases/v0.3.1.md) describen las anotaciones disponibles para las nueve herramientas MCP.

### Elige cómo accede Home Assistant a tu bóveda

| Tu situación | Modo de almacenamiento | Dónde se indica la ruta | Quién sincroniza los archivos |
| --- | --- | --- | --- |
| Los archivos Markdown ya están en `/share` de Home Assistant. Puedes editarlos mediante una carpeta de red o quizá otro proceso ya los sincroniza. | `local` | **Bóveda local**. Por ejemplo, `/share/obsidian-vault`. | El proceso que ya uses, si lo hay. Deja **Activar LiveSync** desmarcado. |
| Obsidian ya usa **Self-hosted LiveSync** y quieres que Home Assistant mantenga una copia sincronizada con el mismo CouchDB. | `livesync` | Deja ambas rutas vacías para usar `/config/obsidian_mcp/vault`. Si ya tienes una bóveda Markdown compartida en `/share`, escríbela en **Bóveda compartida de LiveSync** y activa LiveSync. | La integración ejecuta LiveSync CLI. CouchDB y el complemento de Obsidian funcionan por separado. |

Una carpeta del móvil o del PC solo sirve en modo local si Home Assistant puede acceder a sus archivos Markdown dentro de su propio `/share`. Si quieres replicar por CouchDB, instala **Self-hosted LiveSync** en tus clientes de Obsidian. Puedes conectar ChatGPT con cualquiera de los dos modos.

### Instala y configura la integración

1. Abre **HACS / Marketplace**, busca **Obsidian MCP** y descarga la integración desde el catálogo. Reinicia Home Assistant si te lo pide. Requiere Home Assistant **2026.9.0 o posterior**.

   **Alternativa (repositorio personalizado):** Si la integración aún no aparece en el catálogo, abre **HACS / Marketplace → Repositorios personalizados**, añade [este repositorio](https://github.com/sergey08/obsidian-mcp-integration) como **Integración**, busca **Obsidian MCP** y descárgala.
2. Ve a **Ajustes → Dispositivos y servicios → Añadir integración → Obsidian MCP**. Elige un modo de la tabla y deja el túnel de OpenAI y el acceso MCP remoto desactivados hasta configurarlos.
3. En modo **local**, crea o monta antes la carpeta en `/share`, da permiso de escritura a Home Assistant Core y pon su ruta en **Bóveda local**. En modo **LiveSync**, empieza con ambas rutas vacías y LiveSync desactivado. Sigue los pasos del apartado siguiente.
4. Usa **Configurar** para importar credenciales o ejecutar comprobaciones. Usa **Reconfigurar**, en el menú de la integración, para cambiar las rutas, activar servicios o indicar el ID del túnel.

### Prepara una bóveda sincronizada

1. Configura CouchDB y comprueba que Self-hosted LiveSync funciona en Obsidian. En Home Assistant OS puedes usar una aplicación aparte como [Obsidian Sync Server SSL](https://github.com/alexbelgium/hassio-addons/tree/master/obsidian_syncserver_ssl). Usa un certificado que Home Assistant Core pueda validar y una cuenta de CouchDB con permisos para replicar la base de datos.
2. Consigue el **JSON de configuración de LiveSync CLI** para esa base de datos. Reutiliza una configuración funcional o sigue la [guía del CLI oficial](https://github.com/vrtmrz/obsidian-livesync/tree/1.0.39/src/apps/cli) para prepararla a partir de la configuración de tu cliente Obsidian. Pega el JSON en **Configurar → Importar ajustes de LiveSync**. El formulario guarda el JSON del CLI en `/config/obsidian_mcp/settings/livesync-operativo.json` y protege el archivo existente frente a sobrescrituras. Mantén en privado el JSON y cualquier URI de configuración, especialmente si la bóveda usa cifrado u ofuscación de rutas.
3. Descarga `livesync-ha-amd64.tar.gz` (x86-64) o `livesync-ha-arm64.tar.gz` (AArch64) y su `.sha256` de la [release](https://github.com/sergey08/obsidian-mcp-integration/releases/tag/v0.3.1). Los paquetes son para Home Assistant Core sobre Alpine. La integración busca `/config/obsidian_mcp/bin/livesync-cli`. En otra plataforma, prepara un lanzador LiveSync CLI compatible con esa misma ruta, los argumentos y la variable `LIVESYNC_DB_PATH`.
4. Desde un terminal con acceso de escritura al directorio de configuración, verifica el hash y extrae el paquete. En AMD64, desde la carpeta donde guardaste las dos descargas:

   ```sh
   sha256sum -c livesync-ha-amd64.tar.gz.sha256 &&
     tar -xzf livesync-ha-amd64.tar.gz -C /config/obsidian_mcp
   ```

   Cambia `amd64` por `arm64` en ambos nombres si corresponde, o sustituye `/config` si tu instalación usa otra ruta de configuración. En una instalación nueva, la integración crea el directorio de destino en la primera carga. Para actualizar el runtime, detén antes el proceso de LiveSync.
5. En **Configurar → Comprobar la conexión con CouchDB**, verifica el acceso. En **Reconfigurar**, selecciona `livesync`, activa LiveSync y usa el intervalo de **60 segundos** salvo que necesites otro. Deja ambas rutas vacías para la bóveda que gestiona la integración. Usa **Bóveda compartida de LiveSync** solo con una carpeta Markdown existente y con notas en `/share`, tras detener cualquier otro proceso de escritura o sincronización de esa carpeta. Empieza con una bóveda de prueba y comprueba que una nota desechable llega a otro cliente Obsidian.

### Conecta ChatGPT u otro cliente MCP

- **ChatGPT.** Crea o reutiliza un [túnel privado de OpenAI](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels) en tu organización de OpenAI Platform y consigue su ID y una clave de ejecución autorizada para ese túnel. Descarga un ejecutable verificado y compatible con tu plataforma de las [releases oficiales de tunnel-client](https://github.com/openai/tunnel-client/releases). Colócalo en `/config/obsidian_mcp/tunnel/bin/tunnel-client` y dale permiso de ejecución. Guarda la clave en **Configurar → Guardar la clave del túnel de OpenAI**. En **Reconfigurar**, indica el ID y activa el túnel. Selecciona ese túnel al configurar la app MCP personalizada de tu espacio de ChatGPT, si esta opción está disponible. Prueba a listar las notas. El túnel usa una conexión saliente y un servidor MCP local privado.
- **Otros clientes Streamable HTTP.** En **Reconfigurar**, activa **MCP remoto autenticado** y proporciona un token Bearer aleatorio y exclusivo de al menos 32 caracteres. Configura el cliente con `https://<tu-host-ha>/api/obsidian_mcp/mcp` sobre HTTPS de confianza y la cabecera `Authorization: Bearer <token>`. Deja esta opción desactivada si solo usas el túnel de OpenAI. Con MCP remoto activo, dejar en blanco el token en **Reconfigurar** conserva el valor guardado.
- Las nueve herramientas permiten listar, buscar, leer, leer con revisión, obtener una revisión, escribir, añadir texto, renombrar y borrar. Las notas borradas se conservan en la papelera privada recuperable de `/config/obsidian_mcp/mcp-trash`.

### Menús, diagnóstico y recuperación

| Opción de Configurar | Cuándo usarla |
| --- | --- |
| **Ver el estado de la bóveda y los servicios** | Comprueba la ruta activa y si los procesos de LiveSync y del túnel están iniciados. Confirma la replicación y la conexión del túnel con una petición real. |
| **Comprobar la conexión con CouchDB** | Comprueba el acceso a la base de datos desde Core. Usa una nota de prueba para validar la replicación y la compatibilidad del cifrado. |
| **Importar ajustes de LiveSync** | Carga el JSON del CLI durante la instalación o recuperación. |
| **Guardar la clave del túnel de OpenAI** | Carga la clave de ejecución durante la instalación o recuperación. |

Si una nota no se sincroniza, comprueba la ruta activa, el estado de LiveSync y la conexión a CouchDB. Revisa en Home Assistant y Obsidian los errores de certificados, configuración o replicación. Si ChatGPT no puede listar notas, comprueba el ejecutable y sus permisos, los permisos de la clave, el ID del túnel y su asociación con el espacio de ChatGPT. Oculta los datos privados antes de compartir registros.

### Migra desde la aplicación de Home Assistant

1. Identifica la bóveda en uso y el ID del túnel. Haz una copia de la bóveda, CouchDB, los ajustes de la aplicación, la papelera recuperable y el estado de LiveSync. Prueba la integración primero con la bóveda aislada y los servicios opcionales desactivados.
2. Con la aplicación todavía iniciada, instala el runtime del CLI y copia de forma privada los ajustes de LiveSync, el cliente verificado del túnel y la clave de ejecución a las rutas correspondientes de la integración. Usa **Comprobar la conexión con CouchDB** antes del cambio.
3. Detén la aplicación anterior antes de activar LiveSync en la integración o reutilizar el mismo ID de túnel. Si usabas una bóveda compartida, pon la misma ruta en **Bóveda compartida de LiveSync**. Si usabas una bóveda interna de la aplicación, copia sus archivos Markdown a la bóveda gestionada por la integración con la aplicación detenida. Si los formatos del CLI y la base local son compatibles, puedes copiar la base de LiveSync con el CLI anterior detenido para conservar el estado de replicación. Conserva los datos originales para poder volver atrás.
4. Activa LiveSync y el túnel privado en **Reconfigurar**. Comprueba una lectura y una escritura de prueba desde ChatGPT y abre la nota nueva en un cliente de Obsidian. Mantén la aplicación anterior detenida y con arranque manual durante la transición.
5. Si necesitas volver atrás, desactiva primero la integración y después arranca la aplicación. Si migraste a una copia de la bóveda, reconcilia los cambios hechos desde el corte antes de volver a la carpeta original.
