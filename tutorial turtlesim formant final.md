# Tutorial: Conectar Turtlesim (ROS 2) con Formant

Este tutorial documenta, paso a paso, cómo conectar un nodo de turtlesim corriendo en ROS 2 Humble (Ubuntu 22.04) con la plataforma de observabilidad y teleoperación Formant. Cada paso indica su objetivo, lo que necesitas antes de empezar, la ejecución detallada, y el resultado que confirma que puedes avanzar al siguiente.

> ⚠ **Nota:** este es un tutorial basado en una sesión real de configuración, incluyendo los problemas que aparecieron en el camino y cómo se diagnosticaron, desde errores de conectividad del agente hasta un filtrado silencioso por versión de ROS. El ciclo completo quedó confirmado funcionando de punta a punta al final del Paso 7.

## Paso 0: Instalar ROS 2 Humble y el paquete turtlesim

**Objetivo:** Dejar el sistema completamente listo desde cero — Ubuntu + ROS 2 Humble + turtlesim — para que cualquier persona, incluso sin nada instalado previamente, pueda llegar al Paso 1 en condiciones de arrancar la tortuga. Si ya tienes ROS 2 Humble funcionando, puedes saltar directo a la sección "Instalar turtlesim" más abajo.

**Requisitos:**

- Ubuntu 22.04 LTS (instalación nativa, máquina virtual o WSL2), con conexión a internet
- Usuario con permisos sudo
- Al menos 10 GB libres en disco (la instalación de escritorio de ROS 2 incluye RViz2 y varios paquetes de demostración)

### Instalar ROS 2 Humble (omite esta parte si ya lo tienes funcionando)

1. Configura el idioma del sistema (locale) en UTF-8, requisito de ROS 2:

   ```bash
   sudo apt update && sudo apt install locales
   sudo locale-gen en_US en_US.UTF-8
   sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
   export LANG=en_US.UTF-8
   ```

2. Habilita el repositorio "universe" de Ubuntu, necesario para instalar paquetes de ROS 2:

   ```bash
   sudo apt install software-properties-common
   sudo add-apt-repository universe
   ```

3. Agrega la clave GPG oficial y el repositorio apt de ROS 2:

   ```bash
   sudo apt update && sudo apt install curl -y
   sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
   ```

4. Instala ROS 2 Humble, versión "desktop" (incluye RViz2, demos; turtlesim se instala aparte más abajo):

   ```bash
   sudo apt update && sudo apt upgrade
   sudo apt install ros-humble-desktop
   ```

5. Instala las herramientas de desarrollo y configura rosdep (gestiona dependencias de paquetes ROS):

   ```bash
   sudo apt install ros-dev-tools
   sudo rosdep init
   rosdep update
   ```

6. Carga el entorno de ROS 2 automáticamente en cada terminal nueva, agregándolo a tu `.bashrc`:

   ```bash
   echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
   source ~/.bashrc
   ```

**Resultado:** Al abrir una terminal nueva, `ros2 doctor --report` no muestra errores de configuración base, y `ros2 topic list` devuelve al menos `/parameter_events` y `/rosout`. Esto confirma que ROS 2 Humble quedó instalado y el entorno se carga correctamente.

### Instalar turtlesim

1. Con ROS 2 ya instalado y el entorno cargado, instala el paquete de turtlesim:

   ```bash
   sudo apt update && sudo apt install ros-humble-turtlesim
   ```

2. Verifica que el paquete quedó disponible, listando sus ejecutables:

   ```bash
   ros2 pkg executables turtlesim
   ```

> ⚠ Si el comando anterior no muestra `turtlesim_node` ni `turtle_teleop_key`, revisa que el entorno esté cargado en esa terminal (`echo $ROS_DISTRO` debe mostrar `humble`) y que el paso de `apt install` haya terminado sin errores.

**Resultado:** `ros2 pkg executables turtlesim` lista comandos como `turtlesim_node`, `turtle_teleop_key` y `turtle_teleop_joy`. El sistema queda listo para continuar directamente con el Paso 1.

## Paso 1: Tener Turtlesim corriendo como fuente de datos

**Objetivo:** Contar con un nodo ROS 2 activo que publique datos reales (posición y comandos de velocidad) para que Formant tenga algo que ingerir. Turtlesim es ideal porque es liviano y viene incluido en cualquier instalación de ROS 2.

**Requisitos:**

- Ubuntu 22.04 LTS
- ROS 2 Humble instalado y funcional (`ros2 doctor --report` sin errores)
- Paquete turtlesim instalado: `sudo apt install ros-humble-turtlesim`

**Pasos:**

1. En una terminal, lanza el nodo de simulación:

   ```bash
   ros2 run turtlesim turtlesim_node
   ```

   Se abre una ventana gráfica con una tortuga sobre fondo azul.

2. En una segunda terminal, lanza el control por teclado:

   ```bash
   ros2 run turtlesim turtle_teleop_key
   ```

3. Con el foco en la terminal del teleop, usa las flechas del teclado para mover la tortuga.

*(Captura: turtlesim en movimiento, controlado por teclado)*

**Resultado:** La tortuga se mueve y gira dejando un rastro visual. Esto confirma que existen dos tópicos activos y publicando: `/turtle1/pose` (tipo `turtlesim/msg/Pose`) y `/turtle1/cmd_vel` (tipo `geometry_msgs/msg/Twist`).

## Paso 2: Crear cuenta, dispositivo y agente de Formant

**Objetivo:** Registrar tu computadora como un "dispositivo" dentro de la organización de Formant, e instalar el software (agente) que va a leer los tópicos ROS 2 locales y enviarlos a la nube.

**Requisitos:**

- Cuenta creada en app.formant.io
- Docker instalado (opcional, el instalador puede usar método nativo)
- Conexión a internet estable

**Pasos:**

1. En la app de Formant, ve a **Devices → Add Device**.
2. Asigna un nombre (ej. `turtlesim-dev`) y selecciona **ROS 2** como tipo de sistema.
3. Formant genera un comando de instalación con un token único (válido 72 horas).
4. Ejecuta ese comando en tu terminal de Ubuntu (pedirá contraseña de sudo):

   ```bash
   bash <(wget -qO - https://app.formant.io/install-agent.sh) <TU_TOKEN>
   ```

5. Verifica que el servicio quedó activo:

   ```bash
   sudo systemctl status formant-agent
   ```

**Resultado:** El servicio `formant-agent` aparece como `active (running)`. Los logs confirman la detección de ROS 2, aunque con 0 tópicos configurados todavía — eso se resuelve en el siguiente paso.

```bash
sudo journalctl -u formant-agent | grep -i version
```

```
ROS2 bridge: python3    Topic config: 0 topics (0 telemetry, 0 teleop, 0 filtered by rosVersion)
```

## Paso 3: Configurar los streams (tópicos) a ingerir

**Objetivo:** Decirle a Formant explícitamente qué tópicos de ROS 2 debe leer y bajo qué categoría/tipo de dato, para que empiece a aparecer telemetría real.

**Requisitos:**

- Dispositivo ya creado y agente corriendo (Paso 2)
- Turtlesim corriendo (Paso 1), para confirmar tipos de mensaje

**Pasos:**

1. Ve a **platform.formant.io → Devices → (tu dispositivo) → Streams**. Los streams se organizan por tarjetas de categoría (Video, Joystick, Odometry, Other, etc.).

   *(Captura: pantalla de configuración de streams, organizada por tarjetas de categoría)*

2. Para `/turtle1/pose` (tipo `turtlesim/msg/Pose`, mensaje personalizado), usa la tarjeta **"Other" → ADD**.

   *(Captura: diálogo "Add Other Stream" con el topic /turtle1/pose configurado)*

   Completa: `Topic Name = /turtle1/pose` · `Topic Type = turtlesim/msg/Pose` · `ROS Version = ROS 2` (campo crítico, ver Paso 5).

3. Para `/turtle1/cmd_vel` (tipo `geometry_msgs/msg/Twist`, tipo estándar), usa la tarjeta **"Joystick" → ADD**, con los mismos cuidados de nombre y versión de ROS.

**Resultado:** Ambos streams aparecen listados debajo de sus tarjetas, con el nombre del tópico y el modo "Observe". Esto confirma que la configuración quedó guardada en la nube, pendiente de sincronizarse con el agente local.

## Paso 4: Verificar la sincronización agente ↔ nube

**Objetivo:** Confirmar que la configuración guardada en la web realmente llegó al agente que corre en tu Ubuntu — el paso donde más problemas pueden aparecer.

**Requisitos:**

- Streams configurados (Paso 3)
- Acceso a terminal con sudo

**Pasos:**

1. Observa los logs del agente en vivo:

   ```bash
   sudo journalctl -u formant-agent -f
   ```

2. Busca la línea de configuración de tópicos:

   ```bash
   sudo journalctl -u formant-agent | grep -i "topic config" | tail -5
   ```

3. Si ves errores repetidos de timeout, verifica conectividad general antes de seguir:

   ```
   WARN  Calling EventTriggerApi. Request timed out (context deadline exceeded)
   WARN  Calling AuthApi. Request timed out (context deadline exceeded)
   ```

> ⚠ Si ping y curl responden bien pero el agente sigue fallando, reinícialo para forzar que relea el estado de red:
> ```bash
> sudo systemctl restart formant-agent
> ```

**Resultado:** Después de un reinicio exitoso, el log muestra el número de tópicos sincronizados. Sin embargo, un valor `filtered by rosVersion` mayor a cero es una señal de alerta que resolvemos en el siguiente paso.

```
Topic config: 2 topics (0 telemetry, 2 teleop, 1 filtered by rosVersion)
```

## Paso 5: Diagnosticar y corregir tópicos filtrados por versión de ROS

**Objetivo:** Este fue el problema de raíz más importante de todo el proceso: un stream puede estar bien configurado en nombre y tipo de mensaje, pero si el campo ROS Version no coincide con tu sistema real, el agente lo descarta silenciosamente.

**Requisitos:**

- Al menos un stream mostrando "filtered by rosVersion" en los logs (Paso 4)

**Pasos:**

1. Vuelve a **platform.formant.io → Devices → (tu dispositivo) → Streams**.
2. Localiza el stream problemático (bajo la tarjeta Joystick) y haz click en el ícono de lápiz para editarlo.

   *(Captura: streams ya agregados: /turtle1/cmd_vel bajo Joystick, /turtle1/pose bajo Other)*

3. En el diálogo de edición, revisa todos los campos, no solo el nombre del tópico.

   *(Captura: diálogo de edición mostrando ROS Version incorrectamente en "ROS 1")*

   El campo clave suele pasar desapercibido:

   - **Save to:** "RT ONLY" → cámbialo a **"BOTH"** (sirve tanto para telemetría histórica como teleoperación en vivo)
   - **ROS Version:** "ROS 1" → cámbialo a **"ROS 2"** — esta era la causa real del filtrado

4. Click en **UPDATE STREAM**. Repite para cualquier otro stream con la versión incorrecta.
5. Verifica el resultado — debe aparecer una nueva sección "TELEMETRY STREAMS" con tus tópicos normalizados.

   *(Captura: pantalla final mostrando ambos streams correctamente configurados y la sección "Telemetry Streams")*

**Resultado:** Tras reiniciar el agente, el log final debe decir `0 filtered by rosVersion` — confirmando que el problema de raíz quedó resuelto.

```bash
sudo systemctl restart formant-agent
sudo journalctl -u formant-agent | grep -i "topic config" | tail -5
```

```
Topic config: 2 topics (2 telemetry, 2 teleop, 0 filtered by rosVersion)
```

## Paso 6: Confirmar telemetría básica del dispositivo

**Objetivo:** Antes de meterse con visualizaciones específicas de ROS, conviene confirmar que el ciclo agente → nube → interfaz web funciona en general, usando telemetría de sistema (CPU, memoria, red, disco).

**Requisitos:**

- Agente activo y sincronizado (Pasos 4 y 5)

**Pasos:**

1. Ve a **OBSERVABILITY** (barra superior de la app).
2. Cambia de vista MAP a LIST, y haz click en el nombre de tu dispositivo.

   *(Captura: vista general de Observability con el mapa de dispositivos)*

3. Entra al dispositivo — deberías ver paneles con datos reales del sistema.

   *(Captura: vista de dispositivo con CPU, memoria, disco y ubicación)*

**Resultado:** Se observan valores concretos (ej. `cpu util: 7.25%`, `mem util: 43.77%`) actualizándose — esto confirma que la conexión general dispositivo-nube-interfaz funciona de punta a punta.

## Paso 7: Crear un Dashboard para visualizar los streams de ROS

**Objetivo:** Armar una vista personalizada donde se muestren en vivo los valores de `/turtle1/cmd_vel` y `/turtle1/pose`, cerrando el ciclo completo: ROS 2 → agente → nube → visualización.

**Requisitos:**

- Streams sincronizados y sin filtrado (Paso 5)

**Pasos:**

1. Desde la vista de tu dispositivo, usa el botón **+** (arriba a la derecha) para iniciar la creación de una vista, y selecciona **Dashboard**.

   *(Captura: asistente de creación de vista en 3 pasos: dispositivos, streams, layout)*

2. **Choose Devices:** selecciona únicamente tu dispositivo (`turtlesim-dev`). Dejar varios dispositivos marcados puede reducir la lista de streams disponibles solo a los que comparten en común.

   *(Captura: selección de dispositivos con 2 dispositivos marcados)*

3. **Choose Streams:** busca tu stream por categoría o con el buscador. Con los streams ya corregidos (Paso 5), el contador debería subir de 55 a 57 streams — los 2 nuevos son justamente `turtle1.cmd_vel` y `turtle1.pose`.

   *(Captura: lista de streams, ahora con 57 disponibles tras corregir la versión de ROS)*

   > ⚠ **Problema encontrado en esta sesión:** al principio, con ambos dispositivos de la organización seleccionados, la lista mostraba 55 streams mezclados y el buscador no filtraba bien por `cmd_vel`. Categorías como "Joystick" no aparecían en ningún punto. La causa no era el filtro de dispositivos, sino que estos dos streams personalizados (Twist y Pose, mensajes con varios campos anidados) no se clasifican bajo "Joystick" ni "Other" en esta lista, sino bajo la categoría **"Json"** — porque Formant serializa mensajes compuestos como estructuras JSON.

4. Baja hasta la categoría "Json" (en orden alfabético, después de "Image"). Ahí aparecen, ya marcados con el check, automáticamente.

   *(Captura: turtle1.cmd_vel y turtle1.pose encontrados dentro de la categoría "Json")*

5. Click en **NEXT** para pasar a "EDIT LAYOUT". Formant genera automáticamente un módulo por cada stream seleccionado — no hace falta arrastrar nada manualmente.
6. (Opcional) Usa el menú de tres puntos de cada tarjeta para eliminar los módulos que no interesan.
7. Click en **SAVE** (abajo a la derecha) para guardar el dashboard.

   *(Captura: dashboard mostrando datos JSON reales de cmd_vel y pose con timestamps)*

   ```
   turtle1.cmd_vel — turtlesim-dev — 2:26:30 pm
   {"angular":{"x":0,"y":0,"z":2},"linear":{"x":0,"y":0,"z":0}}

   turtle1.cmd_vel — turtlesim-dev — 2:26:27 pm
   {"angular":{"x":0,"y":0,"z":0},"linear":{"x":2,"y":0,"z":0}}

   turtle1.pose — turtlesim-dev — 2:35:26 pm
   {"angular_velocity":0,"linear_velocity":0,"theta":1.5568147,"x":5.622...}
   ```

   Cada entrada corresponde a una tecla presionada en el teleop: `linear.x=2` es "avanzar", `angular.z=2` es "girar". El panel `$.ros.node_online` confirma además que los nodos `/formant_ros2_bridge` y `/teleop_turtle` están activos y reportando.

🎉 **Ciclo completo confirmado:** ROS 2 (turtlesim) → Formant Agent → Nube → Dashboard visual, con datos reales actualizándose en tiempo real.

### Nota adicional: modo LIVE vs. histórico

Si al mover la tortuga no ves cambios inmediatos en el dashboard, revisa la barra de tiempo inferior: si el cursor está fijo en un punto pasado, el dashboard está en modo "reproducción histórica", no siguiendo el presente. Haz click en el botón **LIVE** (esquina inferior derecha) para volver al tiempo real. Recuerda además que `cmd_vel` solo publica un mensaje por cada tecla presionada — no es un flujo continuo como `pose`.

## Resumen del flujo completo

| Paso | Qué logra | Cómo se verifica |
| --- | --- | --- |
| 0. Instalación | Prepara el sistema: Ubuntu + ROS 2 Humble + turtlesim | `ros2 doctor --report` y `ros2 pkg executables turtlesim` sin errores |
| 1. Turtlesim | Genera datos ROS 2 reales | La tortuga se mueve, tópicos activos |
| 2. Cuenta + Agente | Vincula tu máquina a Formant | `systemctl status formant-agent` → active |
| 3. Configurar streams | Define qué tópicos ingerir | Streams listados en la web |
| 4. Sincronización | Confirma agente ↔ nube | Logs sin timeouts, `Topic config: N topics` |
| 5. Corregir ROS Version | Elimina el filtrado silencioso | `0 filtered by rosVersion` en logs |
| 6. Telemetría básica | Confirma el ciclo completo en general | CPU/memoria visibles en Observability |
| 7. Dashboard | Visualiza datos ROS específicos en vivo | Valores JSON de cmd_vel/pose actualizándose |

## Las dos lecciones más importantes de este proceso

1. Cuando un stream de ROS no aparece pese a estar "bien" configurado, revisa **todos** los campos del diálogo de edición — no solo el nombre del tópico y el tipo de mensaje, sino también el selector de **ROS Version**, que es fácil de pasar por alto y causa un filtrado silencioso sin mensajes de error explícitos en la interfaz web.

2. Al buscar un stream personalizado (como un mensaje Twist o Pose) dentro del asistente de creación de un Dashboard, no asumas que aparecerá bajo la categoría "obvia" (Joystick, Odometry, Other). Formant clasifica los mensajes con estructura compuesta bajo la categoría genérica **Json** — conviene revisar ahí primero si el buscador no da resultados.
