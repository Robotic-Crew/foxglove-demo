# Simulación del Hiwonder JetRover

ROS 2 Humble · Ignition Gazebo Fortress · MoveIt 2 · RViz · Foxglove · Docker

*Arquitectura de la simulación*

Tutorial paso a paso: instalación, conceptos, rutina diaria y solución de problemas

Diego Pedrazas · Octubre 2026 

**Índice**

## 1. Cómo usar esta guía

Esta guía documenta, paso a paso, cómo simular el robot **Hiwonder JetRover** (chasis Mecanum + brazo de 5 grados de libertad con pinza, cámara de profundidad en la muñeca y LiDAR) dentro de un contenedor Docker con **ROS 2 Humble**, **Ignition Gazebo Fortress**, **MoveIt 2**, **RViz** y **Foxglove**, sin tocar la instalación de ROS 2 Humble + Gazebo Classic que ya existe en el PC (TurtleBot3 / Nav2).

Está dividida en tres partes:

- **Parte A – Instalación (una sola vez):** clonar el código, construir la imagen Docker, aplicar dos parches y compilar.
- **Parte B – Rutina del día a día:** levantar la simulación en 4 terminales, mover el brazo y el chasis, y ver todo en Foxglove.
- **Parte C – Problemas y referencia:** errores reales que aparecieron durante el montaje, cómo se resolvieron y una chuleta de comandos.

### 1.1 Dónde se ejecuta cada comando

Lo más importante para no perderse: hay **dos mundos** — el PC (host) y el contenedor. El prompt de la terminal te dice en cuál estás:

| **Prompt** | **Dónde estás** | **Qué se hace ahí** |
| --- | --- | --- |
| trabajo@trabajo-Inspiron-15-5510:~$ | HOST (tu Ubuntu 22.04) | git, editar archivos, docker build, ./run.sh, docker exec, Foxglove |
| root@trabajo-Inspiron-15-5510:~# | CONTENEDOR jetrover | colcon build, source, ros2 launch, ros2 topic, ros2 run |

En los bloques de código verás una etiqueta encima: **HOST** (borde azul) o **CONTENEDOR** (borde naranja).

> **Atención.** Regla de oro: **Git en el host, ROS en el contenedor.** Dentro del contenedor `git` falla con "dubious ownership" porque la carpeta montada pertenece a tu usuario, no a root. Y en el host no hay Ignition ni MoveIt de esta simulación.

### 1.2 Las cuatro terminales

En el día a día se usan 4 terminales. Solo la primera crea el contenedor; las otras tres "entran" en el mismo contenedor con `docker exec`:

| **Terminal** | **Función** | **Cómo se abre** |
| --- | --- | --- |
| T1 | Contenedor + simulación (Gazebo, MoveIt, RViz) | ./run.sh |
| T2 | TF estático del LiDAR | docker exec -it jetrover bash |
| T3 | Puente de Foxglove | docker exec -it jetrover bash |
| T4 | Verificar, mover el chasis, depurar | docker exec -it jetrover bash |

## 2. Conceptos

### 2.1 El robot JetRover

*Figura 1. El JetRover real: chasis Mecanum, brazo con pinza, cámara de profundidad en la muñeca y LiDAR.*

| **Componente** | **En el robot real** | **En la simulación** |
| --- | --- | --- |
| Chasis | Mecanum (4 ruedas omnidireccionales) | Plugin MecanumDrive de Gazebo, se controla con /cmd_vel |
| Brazo | 5 DOF + pinza (servos de bus) | arm_controller + gripper_controller (ros2_control) y MoveIt 2 |
| Cámara | Cámara de profundidad en la muñeca | Sensor RGB-D: /depth_cam/image, /depth_cam/depth_image |
| LiDAR | Slamtec A1 (u otros) | Sensor gpu_lidar: /scan |
| IMU | IMU en la placa | Sensor imu: /imu |
| Cómputo | Jetson (Nano / Orin) | Tu PC dentro de Docker |

### 2.2 Cinemática Mecanum

Un robot diferencial (como el Create 3 o el TurtleBot3) solo puede ir adelante/atrás y girar. Las ruedas Mecanum tienen rodillos a 45° que permiten además **moverse de lado**. Por eso, en el mensaje `geometry_msgs/Twist` que se publica en `/cmd_vel`, el JetRover usa tres campos:

*Figura 2. Los tres movimientos del chasis Mecanum y su campo en /cmd_vel.*

- `linear.x` → adelante (+) / atrás (−), en m/s.
- `linear.y` → lateral izquierda (+) / derecha (−), en m/s. **Esto no existe en un robot diferencial.**
- `angular.z` → giro antihorario (+) / horario (−), en rad/s.

### 2.3 El software de simulación

Hiwonder publica el código del robot real (rama `Jetson_Orin_ros2` del repo oficial `Hiwonder/JetRover`), pero ese código está pensado para el hardware, no para simular. Usamos el repositorio comunitario **luongdev-bot/ros2_ws**, que toma el URDF oficial y añade todo lo necesario para Gazebo:

| **Pieza** | **Versión / valor** | **Para qué** |
| --- | --- | --- |
| ROS 2 | Humble | Mismo ROS que tu host y que la Jetson real |
| Simulador | Ignition Gazebo Fortress 6.x | Física, sensores, cámara y LiDAR |
| Planificador | MoveIt 2 | Planificar trayectorias del brazo sin colisiones |
| Control | gz_ros2_control + ros2_controllers | Ejecutar las trayectorias en las articulaciones |
| Paquetes del repo | 15 | Simulación, navegación, SLAM, periféricos, apps… |
| Paquetes compilados | 7 | Solo los necesarios para Gazebo + MoveIt |

Los 7 paquetes que se compilan con `--packages-up-to jetrover_gazebo jetrover_moveit_config` son: `jetrover_description`, `jetrover_gazebo`, `jetrover_gazebo_plugins`, `jetrover_moveit_config`, `navigation`, `slam` y `peripherals`.

> **Concepto.** Variables de entorno del repo: `MACHINE_TYPE=JetRover_Mecanum` elige el chasis (es el único que se puede conducir en Gazebo). `LIDAR_TYPE=A1` elige el modelo de LiDAR del URDF. Ambas quedan fijadas en el Dockerfile, así que no tienes que exportarlas cada día.

### 2.4 Arquitectura: host y contenedor

*Figura 3. Todo lo de ROS corre en el contenedor; el host solo aporta la pantalla, la GPU, los volúmenes y Foxglove.*

- **Imagen** `jetrover-humble:v2`: la "plantilla" con ROS, Ignition, MoveIt y dependencias (≈6,4 GB). Se construye una vez.
- **Contenedor** `jetrover`: la instancia en ejecución. Se crea con `./run.sh` y se borra al salir (`--rm`).
- **Volumen** `ws/`: tu código y lo compilado viven en el host (`~/docker/jetrover/ws`), así que no se pierden al borrar el contenedor.
- **Volumen** `gz_cache/`: modelos que Ignition descarga la primera vez (`/root/.ignition`).
- `--net=host`: el contenedor comparte la red del PC; por eso Foxglove conecta a `ws://localhost:8765`.
- `--device /dev/dri` + X11: Gazebo y RViz usan la GPU Intel (Mesa) y se dibujan en tu escritorio.

#### ROS_DOMAIN_ID = 42

Como el contenedor comparte la red, sus nodos ROS 2 verían (y serían vistos por) cualquier nodo Humble del host — por ejemplo agentes como Formant o un TurtleBot3. `ROS_DOMAIN_ID=42` crea una "sala" DDS aparte: solo se hablan nodos con el mismo ID. Todas las terminales que entran con `docker exec` heredan este valor.

### 2.5 Flujo de datos

*Figura 4. Qué camino siguen las órdenes (arriba) y los datos de sensores (abajo).*

#### ros2_control y los controladores

`gz_ros2_control` es un plugin de Gazebo que expone las articulaciones simuladas como si fueran hardware real. Encima se cargan tres controladores mediante "spawners":

| **Controlador** | **Tipo** | **Qué hace** |
| --- | --- | --- |
| joint_state_broadcaster | Broadcaster | Publica /joint_states (posición de cada articulación) |
| arm_controller | JointTrajectoryController | Ejecuta trayectorias del brazo (acción follow_joint_trajectory) |
| gripper_controller | Controlador de la pinza | Abre / cierra la pinza |

> **Nota.** Si los spawners no terminan, no hay `/joint_states` ni acción `follow_joint_trajectory`, y MoveIt dirá "Action client not connected". Es el fallo más común; ver Parte C.

#### MoveIt 2 y el octomap

`move_group` recibe un objetivo (desde el panel MotionPlanning de RViz), planifica una trayectoria libre de colisiones para el grupo **arm** y la envía a `arm_controller`. Además lee `/depth_cam/depth_image` y construye un **octomap** (los cubos azules de RViz): un mapa 3D de obstáculos vistos por la cámara de la muñeca, que MoveIt evita al planificar. Para eso hace falta el plugin `DepthImageOctomapUpdater` (paquete `ros-humble-moveit-ros-perception`).

### 2.6 TF: el árbol de coordenadas

*Figura 5. Árbol TF simplificado. Los frames rojos son nombres "estilo Gazebo" que hay que enlazar.*

Ignition nombra los frames de los sensores como `modelo/enlace/sensor`. Por ejemplo, el mensaje `/scan` viene con `frame_id = jetrover/base_footprint/lidar`, pero ese frame **no existe** en el árbol TF del URDF. RViz lo tolera; Foxglove no puede dibujar los puntos. La solución es un TF estático que diga "ese frame está exactamente donde `lidar_frame`" (terminal T2).

> **Nota.** `gazebo.launch.py` ya incluye ese TF estático; `gazebo_moveit.launch.py` no. Por eso en la rutina diaria se lanza a mano en T2.

### 2.7 Tópicos principales

| **Tópico** | **Tipo** | **Descripción** |
| --- | --- | --- |
| /cmd_vel | geometry_msgs/Twist | Velocidad del chasis (x, y, giro z) |
| /odom | nav_msgs/Odometry | Odometría del chasis (desde Gazebo) |
| /tf, /tf_static | tf2_msgs/TFMessage | Transformaciones entre frames |
| /scan | sensor_msgs/LaserScan | LiDAR 2D (frame jetrover/base_footprint/lidar) |
| /imu | sensor_msgs/Imu | Acelerómetro + giroscopio |
| /depth_cam/image | sensor_msgs/Image | Imagen RGB de la cámara de la muñeca |
| /depth_cam/depth_image | sensor_msgs/Image | Profundidad (alimenta el octomap) |
| /depth_cam/camera_info | sensor_msgs/CameraInfo | Parámetros intrínsecos de la cámara |
| /joint_states | sensor_msgs/JointState | Posición de las articulaciones del brazo y pinza |
| /robot_description | std_msgs/String | URDF completo (lo usan RViz y Foxglove) |
| /filtered_depth, /model_depth… | sensor_msgs/Image | Imágenes de depuración de MoveIt (no hace falta verlas) |

### 2.8 Foxglove

Foxglove es una herramienta de visualización web/escritorio. A diferencia de RViz, no es un nodo ROS: se conecta por WebSocket a `foxglove_bridge`, que corre dentro del contenedor y traduce los tópicos. Permite armar **dashboards** con paneles 3D, imagen, gráficas, logs, etc., y es lo que se usa en producción para monitorizar robots reales (en vivo o desde grabaciones rosbag/MCAP).

## 3. Decisiones de diseño (por qué así)

| **Decisión** | **Motivo** |
| --- | --- |
| Docker en lugar de instalar en el host | Ignition Fortress y su bridge chocan con Gazebo Classic 11. Al probar a instalarlos, apt quería eliminar ros-humble-gazebo-* y TurtleBot3. En Docker el host queda intacto. |
| Humble + Fortress (no Jazzy + Harmonic) | El repo y la Jetson real usan Humble; Fortress es el Ignition oficial para Humble. |
| Repo comunitario luongdev-bot/ros2_ws | El repo oficial de Hiwonder no trae simulación para Ignition ni configuración de controladores para Gazebo. |
| Compilar solo 7 de 15 paquetes | Los demás (apps, drivers del hardware real) no son necesarios y tienen dependencias que no existen en un PC. |
| --skip-keys "gazebo_ros_control warehouse_ros_mongo" | Son dependencias de Gazebo Classic / MongoDB que no se usan y no tienen paquete para esta combinación. |
| Imagen v2 (ros2-controllers + moveit-ros-perception) | La v1 no tenía los controladores ("Loader for controller … not found") ni el plugin de octomap. |
| Parche NVIDIA → Mesa | El autor tenía GPU NVIDIA; los launch forzaban __GLX_VENDOR_LIBRARY_NAME=nvidia. Con la Intel Iris Xe eso rompe el render. |
| ROS_DOMAIN_ID=42 | Aislar la simulación de nodos ROS del host. |
| Volúmenes ws/ y gz_cache/ | Conservar código, compilación y modelos descargados entre sesiones. |

## Parte A – Instalación (una sola vez)

Esta parte se hace **una sola vez**. Si ya la completaste, salta a la Parte B. Solo se repite algo si cambias el Dockerfile (reconstruir imagen) o el código de `ws/` (recompilar).

### A1. Comprobar Docker

**HOST:**
```bash
docker context use default
docker --version
docker run --rm hello-world
```

> **Nota.** Si tienes Docker Desktop instalado, el contexto puede quedar en `desktop-linux` y dar "Cannot connect to the Docker daemon". `docker context use default` usa el Docker Engine nativo.

### A2. Crear la carpeta y clonar el código

**HOST:**
```bash
mkdir -p ~/docker/jetrover/gz_cache
cd ~/docker/jetrover
git clone https://github.com/luongdev-bot/ros2_ws.git ws

cd ws
git log --oneline -1                     # ver la versión clonada
find src -name package.xml | wc -l       # debe dar 15
grep -Rn "NV_PRIME\|GLX_VENDOR" src      # ver los ajustes NVIDIA
```

Estructura final de la carpeta:

```
~/docker/jetrover/
├── .dockerignore      # excluye ws y gz_cache del build
├── Dockerfile         # receta de la imagen
├── run.sh             # arranca el contenedor
├── gz_cache/          # → /root/.ignition (modelos descargados)
└── ws/                # → /root/ws (código + build + install)
    └── src/simulations/jetrover_{description,gazebo,moveit_config,…}
```

### A3. Crear .dockerignore y Dockerfile

**HOST:**
```bash
cd ~/docker/jetrover
printf "ws\ngz_cache\n" > .dockerignore
nano Dockerfile
```

Contenido del Dockerfile:

```dockerfile
FROM osrf/ros:humble-desktop
ENV DEBIAN_FRONTEND=noninteractive
ENV MACHINE_TYPE=JetRover_Mecanum
ENV LIDAR_TYPE=A1

RUN apt-get update && apt-get install -y \
      ros-humble-ros-gz \
      ros-humble-ros2-controllers \
      ros-humble-moveit-ros-perception \
      ros-humble-foxglove-bridge \
      python3-colcon-common-extensions \
      python3-vcstool \
      git \
      mesa-utils \
    && rm -rf /var/lib/apt/lists/*

# Instala las dependencias que declaran los package.xml del repo
RUN mkdir -p /tmp/deps && cd /tmp/deps && \
    git clone --depth 1 https://github.com/luongdev-bot/ros2_ws.git . && \
    apt-get update && rosdep update && \
    rosdep install -r -y --ignore-src \
      --from-paths src/simulations src/navigation src/slam src/peripherals \
      --skip-keys "gazebo_ros_control warehouse_ros_mongo" && \
    rm -rf /tmp/deps /var/lib/apt/lists/*

RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc
WORKDIR /root
CMD ["bash"]
```

> **Concepto.** Qué hace cada bloque: `FROM osrf/ros:humble-desktop` → base oficial con ROS 2 Humble + RViz. `ros-humble-ros-gz` → trae Ignition Fortress y el bridge ROS↔Gazebo. `ros2-controllers` → arm_controller, joint_state_broadcaster… `moveit-ros-perception` → plugin de octomap. El bloque `rosdep` clona el repo temporalmente solo para leer sus dependencias e instalarlas; luego lo borra (el código real llega por el volumen).

### A4. Construir la imagen

**HOST:**
```bash
cd ~/docker/jetrover
docker build -t jetrover-humble:v2 .
docker images | grep jetrover            # ~6.4 GB
```

> **Nota.** Tarda 10–20 minutos la primera vez. El `.dockerignore` evita copiar `ws/` al contexto de build. Si existe una imagen `jetrover-humble:v1`, puedes borrarla con `docker rmi jetrover-humble:v1`.

### A5. Parches al código (en el host)

#### Parche 1: NVIDIA → Mesa (3 launch de Gazebo)

**HOST:**
```bash
cd ~/docker/jetrover/ws
sed -i "s/'__NV_PRIME_RENDER_OFFLOAD', '1'/'__NV_PRIME_RENDER_OFFLOAD', '0'/; s/'__GLX_VENDOR_LIBRARY_NAME', 'nvidia'/'__GLX_VENDOR_LIBRARY_NAME', 'mesa'/" src/simulations/jetrover_gazebo/launch/*.launch.py

# Verificar: deben salir '0' y 'mesa' en los 3 launch
grep -Rn "NV_PRIME\|GLX_VENDOR" src/simulations
```

#### Parche 2: robot_description como string en sim_display

Sin esto, `sim_display.launch.py` falla con "Unable to parse the value of parameter robot_description as yaml".

**HOST:**
```bash
cd ~/docker/jetrover/ws
F=src/simulations/jetrover_description/launch/sim_display.launch.py

# 1) Envolver el resultado de xacro como string
sed -i "s/robot_description = Command(\['xacro ', urdf_path\])/robot_description = ParameterValue(Command(['xacro ', urdf_path]), value_type=str)/" $F

# 2) Añadir el import (solo si no existe)
grep -q "import ParameterValue" $F || sed -i "1i from launch_ros.parameter_descriptions import ParameterValue" $F

# 3) Comprobar
grep -n "ParameterValue" $F
python3 -m py_compile $F && echo "sintaxis OK"
git diff --stat
```

> **Atención.** Aplica este parche **solo a sim_display.launch.py**. Los launch de Gazebo ya envuelven `robot_description` en `ParameterValue` más abajo; si lo envuelves dos veces aparece: `TypeError: value should be either a scalar, a substitutions…` Arreglo: `git checkout -- <archivo>` en el host y volver a aplicar el parche NVIDIA a ese archivo.

### A6. Crear run.sh

**HOST:**
```bash
cd ~/docker/jetrover
nano run.sh
chmod +x run.sh
```

```bash
#!/bin/bash
xhost +local:docker > /dev/null
docker run -it --rm \
  --name jetrover \
  --net=host \
  --ipc=host \
  --device /dev/dri \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e ROS_DOMAIN_ID=42 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/docker/jetrover/ws:/root/ws \
  -v $HOME/docker/jetrover/gz_cache:/root/.ignition \
  jetrover-humble:v2
```

| **Opción** | **Significado** |
| --- | --- |
| xhost +local:docker | Permite que el contenedor dibuje ventanas en tu escritorio |
| -it --rm | Terminal interactiva; borra el contenedor al salir |
| --name jetrover | Nombre fijo para poder hacer docker exec -it jetrover bash |
| --net=host / --ipc=host | Misma red y memoria compartida que el host (DDS, Foxglove) |
| --device /dev/dri | Acceso a la GPU Intel |
| -e DISPLAY, -v /tmp/.X11-unix | Ventanas gráficas (X11/XWayland) |
| -e ROS_DOMAIN_ID=42 | Aísla los nodos ROS del contenedor |
| -v …/ws:/root/ws | Código y compilación persistentes |
| -v …/gz_cache:/root/.ignition | Modelos de Ignition persistentes |

> **Atención.** Las líneas que terminan en `\` son UNA sola orden. Si pegas una línea suelta como `-v ...` en la terminal, bash responde "-v: orden no encontrada". Por eso todo va dentro de run.sh.

### A7. Primer arranque y comprobaciones

**HOST:**
```bash
cd ~/docker/jetrover
./run.sh
```

**CONTENEDOR:**
```bash
echo $MACHINE_TYPE $LIDAR_TYPE $ROS_DOMAIN_ID   # JetRover_Mecanum A1 42
glxinfo -B | grep -i "renderer"                 # debe mencionar Mesa / Intel
```

### A8. Compilar el workspace

**CONTENEDOR:**
```bash
cd /root/ws
colcon build --symlink-install --packages-up-to jetrover_gazebo jetrover_moveit_config
source /root/ws/install/setup.bash
```

> **Resultado esperado.** "Summary: 7 packages finished" en unos 15 segundos. Como `ws/` está en el host, esto **no** hay que repetirlo cada día; solo si cambias código.

> **Concepto.** `--symlink-install` enlaza los archivos Python/launch/YAML en vez de copiarlos: si editas un launch no hace falta recompilar.

### A9. Prueba 1 – Solo el modelo en RViz

**CONTENEDOR:**
```bash
ros2 launch jetrover_description sim_display.launch.py
```

*Figura 6. sim_display.launch.py: el URDF del JetRover en RViz con joint_state_publisher_gui.*

Sirve para comprobar que el URDF se genera bien. Cierra con Ctrl+C.

### A10. Prueba 2 – Gazebo con sensores

**CONTENEDOR:**
```bash
ros2 launch jetrover_gazebo gazebo.launch.py
```

*Figura 7. gazebo.launch.py: mundo con dos cajas, el JetRover y el panel "Visualize lidar".*

La primera vez Ignition descarga modelos a `gz_cache/` y puede tardar. En otra terminal (`docker exec`, ver B4) prueba `ros2 topic list` y mueve el chasis con `/cmd_vel` (B6). Cierra con Ctrl+C.

Con esto la instalación está completa. La prueba 3 (Gazebo + MoveIt) es la rutina diaria de la Parte B.

## Parte B – Rutina del día a día

Cada día, al encender el PC, no hay nada corriendo: el contenedor se borró al salir (`--rm`). Pero la imagen, el código compilado y los modelos siguen ahí, así que **no** hay que repetir la Parte A. Solo esto:

*Figura 8. Las cuatro terminales y su orden.*

### B1. T1 – Contenedor y simulación completa

**HOST:**
```bash
cd ~/docker/jetrover
./run.sh
```

**CONTENEDOR:**
```bash
source /root/ws/install/setup.bash
ros2 launch jetrover_gazebo gazebo_moveit.launch.py
```

Esto abre **Gazebo** (física + sensores), carga los **controladores**, arranca **MoveIt** y abre **RViz** con el panel MotionPlanning.

- En Gazebo, **cierra el panel "Visualize lidar"** (o quítalo del menú de plugins). Llena la terminal con "Lidar pointer is not set" y ralentiza el arranque de los spawners.
- **Espera 1–2 minutos** hasta que los spawners terminen. En T1 deben aparecer líneas como "Configured and activated joint_state_broadcaster", "… arm_controller", "… gripper_controller".
- Si tras 2–3 min siguen colgados en "waiting for service" o "asking for robot_description": Ctrl+C y vuelve a lanzar.

> **Atención.** Si `source` falla o `ros2 launch` dice "Package 'jetrover_gazebo' not found", te olvidaste del `source /root/ws/install/setup.bash` (o no compilaste, A8). Y si `./run.sh` no existe, no estás en `~/docker/jetrover`.

### B2. T2 – TF estático del LiDAR

**HOST:**
```bash
docker exec -it jetrover bash
```

**CONTENEDOR:**
```bash
source /root/ws/install/setup.bash
ros2 run tf2_ros static_transform_publisher \
  --frame-id lidar_frame \
  --child-frame-id jetrover/base_footprint/lidar
```

Déjalo corriendo. Sin esto, Foxglove no puede situar los puntos de `/scan` (ver 2.6).

### B3. T3 – Puente de Foxglove

**HOST:**
```bash
docker exec -it jetrover bash
```

**CONTENEDOR:**
```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

> **Resultado esperado.** Aparece "Server listening on port 8765". Déjalo corriendo.

### B4. T4 – Verificar que todo está vivo

**HOST:**
```bash
docker exec -it jetrover bash
```

**CONTENEDOR:**
```bash
source /root/ws/install/setup.bash
ros2 topic list
ros2 topic echo /joint_states --once            # ¿se publican articulaciones?
ros2 action list | grep follow_joint_trajectory # ¿arm_controller listo?
ros2 topic hz /scan                             # ~ frecuencia del LiDAR (Ctrl+C)
ros2 topic echo /odom --once
```

Para revisar el log del launch de T1 sin hacer scroll:

**CONTENEDOR:**
```bash
LOG=$(ls -td /root/.ros/log/*/ | head -1)launch.log
grep -E "spawner|ERROR|activated" $LOG | tail -20
```

> **Nota.** Si `/joint_states` no publica o no aparece `follow_joint_trajectory`, los controladores no se cargaron: vuelve a B1.

### B5. Mover el brazo con MoveIt (RViz)

- En RViz, panel **MotionPlanning** → pestaña **Planning**. Comprueba que **Planning Group = arm**.
- Elige el objetivo: arrastra la bola/flechas interactivas del extremo del brazo, o en **Goal State** elige `random valid` o un estado predefinido.
- Pulsa **Plan**: verás la trayectoria "fantasma" en naranja.
- Pulsa **Execute**: el brazo se mueve en RViz **y** en Gazebo.
- Para ir más lento, baja **Velocity Scaling** / **Acceleration Scaling** (p. ej. 0.3).

*Figura 9. Brazo planificado y ejecutado con MoveIt. Los cubos azules son el octomap generado por la cámara de profundidad.*

> **Concepto.** Si el brazo "fantasma" sigue moviéndose en bucle tras ejecutar, no es un error: es la animación de la trayectoria planificada. En el panel Displays → MotionPlanning → **Planned Path**, desmarca **Loop Animation** (y **Show Trail** si quieres).

> **Nota.** Si el octomap acumula cubos viejos que estorban al planificar, en la pestaña **Context** / Planning Scene del panel usa **Clear octomap**.

### B6. Mover el chasis Mecanum (/cmd_vel)

Desde T4. `-r 10` publica 10 veces por segundo hasta que pulses Ctrl+C:

**CONTENEDOR:**
```bash
# Adelante
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"

# Lateral a la izquierda (solo Mecanum)
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {y: 0.2}}"

# Girar sobre sí mismo
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{angular: {z: 0.5}}"

# Diagonal
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2, y: 0.2}}"

# PARAR (siempre después de Ctrl+C)
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}"
```

> **Atención.** Al hacer Ctrl+C el robot puede seguir con la última velocidad recibida. Publica siempre el mensaje vacío `"{}"` para pararlo.

Opcional (no probado en esta configuración): si el paquete `teleop_twist_keyboard` está en la imagen, `ros2 run teleop_twist_keyboard teleop_twist_keyboard` permite conducir con el teclado; con **Shift** + teclas activa el modo holonómico (lateral).

### B7. Foxglove

#### Conectar

- Abre Foxglove (app de escritorio o navegador) en el host.
- **Open connection** → **Foxglove WebSocket** → URL `ws://localhost:8765` → Open.
- Carga o crea el layout **JetRover Dashboard**.

*Figura 10. Primera vista en Foxglove: el modelo y el frustum de la cámara.*

#### Configurar el panel 3D

- **Frame → Display frame = odom** (el robot se mueve respecto a odom).
- **Topics**: activa `/robot_description` (modelo URDF), `/tf` y `/scan`.
- En `/scan` sube **Point size** (3–5) y elige un color visible.
- Desactiva todos los tópicos de **imagen** en el panel 3D (usa el filtro "Show visible" para encontrarlos): `/depth_cam/*`, `/filtered_depth`, `/model_depth`, etc.
- Si cambiaste mucho, desconecta y vuelve a conectar para vaciar el buffer.

*Figura 11. Error "Live connection frame buffer limit (250.0 MB) reached": demasiadas imágenes suscritas.*

*Figura 12. Tras desactivar las imágenes y añadir el TF del LiDAR (T2): los puntos de /scan aparecen sobre el mundo.*

#### Paneles recomendados para el dashboard

| **Panel** | **Tópico** | **Para qué** |
| --- | --- | --- |
| 3D | /robot_description, /tf, /scan | Robot, LiDAR y odometría |
| Image | /depth_cam/image | Ver lo que ve la cámara de la muñeca (fuera del 3D) |
| Plot | /joint_states.position[…] | Ángulos de las articulaciones durante una trayectoria |
| Plot | /odom.twist.twist.linear.x / .y | Velocidad real del chasis vs. la ordenada |
| Raw Messages | /imu | Orientación y aceleraciones |
| Log | /rosout | Mensajes y errores de los nodos |

> **Atención.** Problema conocido aún sin resolver: en Foxglove el modelo puede verse "desarmado" (Links con error rojo) aunque en RViz se vea bien. RViz/MoveIt dibuja con /joint_states, mientras que Foxglove usa solo /tf, y el /tf que viene de Gazebo puede tener frames con nombres distintos. Ver Pendientes.

### B8. Apagar todo

- En T4, T3 y T2: Ctrl+C y `exit`.
- En T1: Ctrl+C (espera unos segundos) y `exit`. El contenedor se borra solo.
- Comprueba en el host: `docker ps` no debe listar `jetrover`.

> **Nota.** Al cerrar pueden salir "Segmentation fault" de VisualizeLidar o move_group. Son fallos en los destructores al apagar: inofensivos.

> **Concepto.** Si una terminal se colgó o quedó un contenedor "zombi": `docker stop jetrover` desde el host.

## Parte C – Problemas y soluciones

### C1. Errores encontrados durante el montaje

| **Síntoma / mensaje** | **Causa** | **Solución** |
| --- | --- | --- |
| Cannot connect to the Docker daemon … desktop-linux | Contexto de Docker Desktop activo | docker context use default |
| -v: orden no encontrada | Se pegó una línea suelta de docker run | Usar ./run.sh |
| ./run.sh: No existe el archivo | No estás en la carpeta | cd ~/docker/jetrover |
| Package 'jetrover_gazebo' not found | Falta source del workspace | source /root/ws/install/setup.bash (o compilar, A8) |
| Unable to parse … robot_description as yaml | xacro devuelve XML que ROS intenta leer como YAML | Parche 2 (A5) en sim_display |
| TypeError: value should be either a scalar… | ParameterValue aplicado dos veces | git checkout -- <archivo> (HOST) y reaplicar parche NVIDIA |
| fatal: detected dubious ownership | git dentro del contenedor | Usar git en el host |
| Loader for controller … not found | Imagen v1 sin ros2-controllers | Construir y usar jetrover-humble:v2 |
| Action client not connected: arm_controller/follow_joint_trajectory | Controladores no cargados | Esperar spawners / relanzar T1 / imagen v2 |
| DepthImageOctomapUpdater … does not exist | Falta moveit-ros-perception | Imagen v2 |
| Spawner colgado en "asking for robot_description" | Arranque lento; panel Visualize lidar saturando | Cerrar el panel, esperar 1–2 min, relanzar |
| /joint_states no publica | joint_state_broadcaster no activo | Ver log (B4) y relanzar T1 |
| Dos RViz superpuestos / datos raros | Lanzamientos previos aún vivos | Cerrar todo; docker stop jetrover; volver a B1 |
| Trayectoria en bucle en RViz | Loop Animation activado | Desmarcar Planned Path → Loop Animation |
| /scan no se ve en Foxglove | Frame jetrover/base_footprint/lidar sin TF | T2: static_transform_publisher |
| Frame buffer limit (250 MB) reached | Imágenes suscritas en el panel 3D | Desactivar tópicos de imagen y reconectar |
| grep: … Is a directory | Se pasó la carpeta del log, no el archivo | LOG=$(ls -td /root/.ros/log/*/ \| head -1)launch.log |
| Gazebo negro, muy lento o no abre | Render / X11 | Revisar parche NVIDIA, glxinfo -B, xhost +local:docker, --device /dev/dri |

### C2. Mensajes que se pueden ignorar

| **Mensaje** | **Por qué es inofensivo** |
| --- | --- |
| Lidar pointer is not set | Lo emite el panel Visualize lidar de Gazebo; ciérralo y desaparece |
| Segmentation fault al pulsar Ctrl+C | Destructores de VisualizeLidar / move_group al apagar |
| Avisos de QoS / durability incompatibles | Suscriptores y publicadores con perfiles distintos; los datos que importan llegan |
| KDL: root link has an inertia… | Aviso clásico de URDF; no afecta a la simulación |
| load_yaml deprecated | API antigua de MoveIt; sigue funcionando |
| Avisos de recognize_objects / plugins opcionales | Funcionalidades del robot real no usadas aquí |

### C3. Volver a un estado limpio

**HOST:**
```bash
cd ~/docker/jetrover/ws
git status                 # qué archivos cambiaron
git diff                   # ver los cambios exactos
git checkout -- <archivo>  # deshacer cambios en un archivo
# luego reaplicar los parches de A5 que correspondan
```

**CONTENEDOR:**
```bash
cd /root/ws
rm -rf build install log
colcon build --symlink-install --packages-up-to jetrover_gazebo jetrover_moveit_config
```

## Referencia rápida – comandos del día a día

| **Acción** | **Dónde** | **Comando** |
| --- | --- | --- |
| Arrancar contenedor | HOST | cd ~/docker/jetrover && ./run.sh |
| Abrir otra terminal en el contenedor | HOST | docker exec -it jetrover bash |
| Cargar el workspace | CONT. | source /root/ws/install/setup.bash |
| Simulación completa | CONT. | ros2 launch jetrover_gazebo gazebo_moveit.launch.py |
| Solo Gazebo + sensores | CONT. | ros2 launch jetrover_gazebo gazebo.launch.py |
| Solo modelo en RViz | CONT. | ros2 launch jetrover_description sim_display.launch.py |
| TF del LiDAR | CONT. | ros2 run tf2_ros static_transform_publisher --frame-id lidar_frame --child-frame-id jetrover/base_footprint/lidar |
| Puente Foxglove | CONT. | ros2 launch foxglove_bridge foxglove_bridge_launch.xml |
| Listar tópicos | CONT. | ros2 topic list |
| Ver un mensaje | CONT. | ros2 topic echo /joint_states --once |
| Frecuencia | CONT. | ros2 topic hz /scan |
| Acciones (controladores) | CONT. | ros2 action list |
| Nodos activos | CONT. | ros2 node list |
| Adelante | CONT. | ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}" |
| Lateral | CONT. | ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {y: 0.2}}" |
| Girar | CONT. | ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{angular: {z: 0.5}}" |
| Parar | CONT. | ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}" |
| Ver árbol TF (frames) | CONT. | ros2 run tf2_ros tf2_echo odom base_footprint |
| Último log del launch | CONT. | LOG=$(ls -td /root/.ros/log/*/ \| head -1)launch.log; tail -50 $LOG |
| Recompilar | CONT. | cd /root/ws && colcon build --symlink-install --packages-up-to jetrover_gazebo jetrover_moveit_config |
| Contenedores en marcha | HOST | docker ps |
| Parar el contenedor a la fuerza | HOST | docker stop jetrover |
| Imágenes y espacio | HOST | docker images ; docker system df |
| Reconstruir imagen | HOST | cd ~/docker/jetrover && docker build -t jetrover-humble:v2 . |

## Pendientes y próximos pasos

- **Modelo desarmado en Foxglove:** diagnosticar qué frames publica Gazebo en /tf (`timeout 5 ros2 topic echo /tf --field transforms | grep frame_id | sort | uniq -c`) y revisar el detalle de los errores de Links en Foxglove.
- **TF del LiDAR permanente:** añadir el `static_transform_publisher` dentro de `gazebo_moveit.launch.py` (como ya hace `gazebo.launch.py`) para no necesitar T2.
- **Dashboard de Foxglove completo:** panel Image con la cámara, Plots de articulaciones y velocidades, IMU y Log; guardar el layout.
- **Navegación y SLAM:** los paquetes `navigation` y `slam` ya se compilan; el siguiente paso natural es mapear el mundo con el LiDAR y navegar con Nav2.
- **Robot real:** con el mismo Humble, los mismos tópicos y Foxglove, el dashboard servirá casi sin cambios para el JetRover físico (la Jetson publica /cmd_vel, /scan, /odom, /joint_states con los drivers de Hiwonder).
