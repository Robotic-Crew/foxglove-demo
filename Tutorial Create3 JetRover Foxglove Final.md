# Create 3 + JetRover en un solo dashboard

*Simulación en Gazebo, control con RViz/MoveIt y visualización conjunta en Foxglove · Docker · ROS 2 Jazzy + Humble*

![Create 3 y JetRover en Foxglove](img/final_clock.png)

Tutorial paso a paso · Diego Pedrazas · Octubre 2026

## Índice

- [1. Introducción](#1-introduccion)
  - [1.1 Qué vas a conseguir](#11-que-vas-a-conseguir)
  - [1.2 Cómo está organizada](#12-como-esta-organizada)
  - [1.3 Dónde se ejecuta cada comando](#13-donde-se-ejecuta-cada-comando)
  - [1.4 Estructura de carpetas en el host](#14-estructura-de-carpetas-en-el-host)
- [2. Conceptos](#2-conceptos)
  - [2.1 Los dos robots](#21-los-dos-robots)
  - [2.2 Por qué dos contenedores Docker](#22-por-que-dos-contenedores-docker)
  - [2.3 Arquitectura completa](#23-arquitectura-completa)
  - [2.4 ROS_DOMAIN_ID: canales separados](#24-ros_domain_id-canales-separados)
  - [2.5 El relay (create3_relay.py)](#25-el-relay-create3_relaypy)
  - [2.6 Árbol TF combinado](#26-arbol-tf-combinado)
  - [2.7 Foxglove y las mallas 3D](#27-foxglove-y-las-mallas-3d)
  - [2.8 La carga del TF y el límite de Foxglove](#28-la-carga-del-tf-y-el-limite-de-foxglove)
  - [2.9 El reloj de Foxglove (use_sim_time)](#29-el-reloj-de-foxglove-use_sim_time)
  - [2.10 ros2_control, spawners y RTF](#210-ros2_control-spawners-y-rtf)
- [Parte A – Instalación (una sola vez)](#parte-a--instalacion-una-sola-vez)
  - [A1. Comprobar Docker](#a1-comprobar-docker)
  - [A2. Create 3: imagen y contenedor](#a2-create-3-imagen-y-contenedor)
  - [A3. JetRover: imagen, código y compilación](#a3-jetrover-imagen-codigo-y-compilacion)
  - [A4. Mallas del Create 3 para el bridge del JetRover](#a4-mallas-del-create-3-para-el-bridge-del-jetrover)
  - [A5. Archivos del dashboard](#a5-archivos-del-dashboard)
  - [A6. Instalar Foxglove](#a6-instalar-foxglove)
- [Parte B – Uso: levantar todo y trabajar](#parte-b--uso-levantar-todo-y-trabajar)
  - [B1. Opción automática: dashboard.sh](#b1-opcion-automatica-dashboardsh)
  - [B2. Opción manual: terminal por terminal](#b2-opcion-manual-terminal-por-terminal)
  - [B3. Configurar Foxglove](#b3-configurar-foxglove)
  - [B4. Mover los robots](#b4-mover-los-robots)
  - [B5. Apagar](#b5-apagar)
- [Parte C – Problemas y soluciones](#parte-c--problemas-y-soluciones)
  - [C1. Foxglove](#c1-foxglove)
  - [C2. Simulación y controladores](#c2-simulacion-y-controladores)
  - [C3. Docker, dominios y CLI de ROS](#c3-docker-dominios-y-cli-de-ros)
- [Parte D – Comandos esenciales del día a día](#parte-d--comandos-esenciales-del-dia-a-dia)
  - [D1. Arrancar y detener](#d1-arrancar-y-detener)
  - [D2. Create 3 (contenedor create3)](#d2-create-3-contenedor-create3)
  - [D3. JetRover y dashboard (contenedor jetrover)](#d3-jetrover-y-dashboard-contenedor-jetrover)
  - [D4. Verificación y diagnóstico](#d4-verificacion-y-diagnostico)
  - [D5. Foxglove](#d5-foxglove)
  - [D6. Las 5 reglas de oro](#d6-las-5-reglas-de-oro)
- [Anexo A – create3_relay.py](#anexo-a--create3_relaypy)
- [Anexo B – foxglove_bridge.yaml](#anexo-b--foxglove_bridgeyaml)
- [Anexo C – dashboard.sh](#anexo-c--dashboardsh)

---

# 1. Introducción

Esta guía reúne todo lo necesario para simular **dos robots a la vez** — el **iRobot Create 3** y el **Hiwonder JetRover** — controlarlos desde **Gazebo** y **RViz/MoveIt**, y verlos juntos, moviéndose en tiempo real, en **un solo dashboard de Foxglove**.

![Figura 1. Resultado final: el brazo del JetRover planificado en RViz (izquierda) y los dos robots en el mismo panel 3D de Foxglove (derecha).](img/final_rviz_fox.png)

*Figura 1. Resultado final: el brazo del JetRover planificado en RViz (izquierda) y los dos robots en el mismo panel 3D de Foxglove (derecha).*

## 1.1 Qué vas a conseguir

- Create 3 simulado en **ROS 2 Jazzy + Gazebo Harmonic** (contenedor `create3`).
- JetRover simulado en **ROS 2 Humble + Ignition Fortress + MoveIt 2** (contenedor `jetrover`).
- Un **relay** propio (`create3_relay.py`) que trae el Create 3 al "mundo" del JetRover sin que choquen tópicos, frames ni relojes.
- Un **dashboard de Foxglove** con ambos robots, sus mallas 3D, el LiDAR y un panel para conducir el Create 3.
- Un script (`dashboard.sh`) que levanta todo con un solo comando.

## 1.2 Cómo está organizada

| Parte | Contenido | Frecuencia |
|---|---|---|
| 2. Conceptos | Por qué está armado así: contenedores, dominios, relay, TF, Foxglove | Leer una vez |
| A. Instalación | Imágenes Docker, compilación, parches, archivos del dashboard | Una sola vez |
| B. Uso | Levantar todo, configurar Foxglove, mover los robots, apagar | Cada sesión |
| C. Problemas | Todos los errores reales que aparecieron y su solución | Cuando algo falla |
| D. Comandos esenciales | Chuleta para el día a día | Siempre a mano |
| Anexos | Código completo de los tres archivos | Referencia |

## 1.3 Dónde se ejecuta cada comando

Hay **tres mundos**: tu PC (host) y dos contenedores. El prompt de la terminal te dice en cuál estás:

| Prompt | Dónde estás |
|---|---|
| trabajo@trabajo-Inspiron-15-5510:~$ | HOST (Ubuntu 22.04) |
| root@trabajo-Inspiron-15-5510:~# | Dentro de un contenedor (create3 o jetrover) |

Los bloques de código llevan una etiqueta: **HOST** (borde azul) o **CONTENEDOR** (borde naranja; se indica cuál).

> **Atención.** Los dos contenedores usan `--net=host`, así que su prompt es igual. Guíate por **cómo abriste** la terminal (`docker exec -it create3 bash` o `docker exec -it jetrover bash`).
>
> Las rutas `/root/...` solo existen dentro del contenedor. En el host, `/root/ws` del JetRover es `~/docker/jetrover/ws`.

## 1.4 Estructura de carpetas en el host

```text
~/docker/
├── dashboard.sh                    # arranque automático (Anexo C)
├── create3_harmonic/
│   ├── Dockerfile                  # imagen create3-jazzy:v1
│   └── run.sh                      # crea el contenedor "create3" (dominio 78)
└── jetrover/
    ├── Dockerfile                  # imagen jetrover-humble:v2
    ├── run.sh                      # crea el contenedor "jetrover" (dominio 77)
    ├── gz_cache/                   # → /root/.ignition
    └── ws/                         # → /root/ws
        ├── src/ build/ install/    # workspace del JetRover
        ├── extra_pkgs/             # mallas del Create 3 para Foxglove
        │   └── irobot_create_description/
        └── scripts/
            ├── create3_relay.py    # relay 78 → 77 (Anexo A)
            └── foxglove_bridge.yaml# parámetros del bridge (Anexo B)
```

# 2. Conceptos

## 2.1 Los dos robots

|  | iRobot Create 3 | Hiwonder JetRover |
|---|---|---|
| Tipo | Base diferencial (2 ruedas + rueda loca) | Chasis Mecanum (4 ruedas omni) + brazo 5 DOF con pinza |
| ROS 2 | Jazzy | Humble (igual que la Jetson del robot real) |
| Simulador | Gazebo Harmonic (gz-sim 8) | Ignition Gazebo Fortress (6.x) |
| Código | iRobotEducation/create3_sim (rama jazzy) | luongdev-bot/ros2_ws |
| Movimiento | /cmd_vel  geometry_msgs/TwistStamped | /cmd_vel  geometry_msgs/Twist + MoveIt para el brazo |
| Sensores | bumper, cliff, IR, IMU, mouse óptico | LiDAR /scan, cámara RGB-D, IMU |
| Contenedor / dominio | create3 · 78 | jetrover · 77 |

![Figura 2. Create 3 en Gazebo Harmonic (mundo "depot") con los paneles Create3 HMI y Teleop.](img/c3_gazebo.png)

*Figura 2. Create 3 en Gazebo Harmonic (mundo "depot") con los paneles Create3 HMI y Teleop.*

![Figura 3. JetRover en Ignition Fortress con el brazo, la cámara y el LiDAR.](img/jr_gazebo.png)

*Figura 3. JetRover en Ignition Fortress con el brazo, la cámara y el LiDAR.*

## 2.2 Por qué dos contenedores Docker

- El host tiene **ROS 2 Humble + Gazebo Classic 11** (TurtleBot3/Nav2). Instalar Ignition o Harmonic encima obliga a apt a **desinstalar Gazebo Classic**. En Docker el host queda intacto.
- El Create 3 tiene su simulación mantenida para **Jazzy + Harmonic**; el JetRover, para **Humble + Fortress**. No pueden vivir en la misma instalación de ROS, así que cada uno tiene su imagen.
- **Imagen** = plantilla (se construye una vez). **Contenedor** = instancia en ejecución; con `--rm` se borra al salir, pero la imagen y los volúmenes (`ws/`, `gz_cache/`) quedan.

## 2.3 Arquitectura completa

![Figura 4. Los dos contenedores, el relay que los une y Foxglove en el host.](img/d_arch.png)

*Figura 4. Los dos contenedores, el relay que los une y Foxglove en el host.*

## 2.4 ROS_DOMAIN_ID: canales separados

Con `--net=host` todos los nodos ROS 2 del PC se pueden ver entre sí. `ROS_DOMAIN_ID` funciona como un **canal de radio**: solo se comunican los nodos con el mismo número.

![Figura 5. Dominios ROS en el PC. El relay es el único proceso que escucha en dos a la vez.](img/d_domains.png)

*Figura 5. Dominios ROS en el PC. El relay es el único proceso que escucha en dos a la vez.*

> **Concepto.** ¿Por qué no poner los dos robots en el mismo dominio? Porque usan **los mismos nombres**:
>
> • Tópicos: `/cmd_vel`, `/odom`, `/joint_states`, `/robot_description` → un comando movería a los dos.
>
> • Frames: `odom`, `base_link` → Foxglove mezclaría los árboles TF.
>
> • Reloj: dos Gazebos publicando `/clock` desordenan el tiempo de MoveIt.
>
> El argumento `namespace:=` del Create 3 separa los tópicos, pero **no** los nombres de frames ni el `/clock` (revisado en el código de create3_sim).

> **Nota.** Un nodo lee `ROS_DOMAIN_ID` **al arrancar**. Cambiar la variable no afecta a lo que ya está corriendo: hay que relanzar.

## 2.5 El relay (create3_relay.py)

Un único proceso Python con **dos contextos rclpy**: un nodo en el dominio 78 y otro en el 77.

![Figura 6. Qué entra y qué sale del relay.](img/d_relay.png)

*Figura 6. Qué entra y qué sale del relay.*

| Función | Detalle |
|---|---|
| Prefijo | `base_link` → `create3/base_link`, `odom` → `create3/odom`, etc. También renombra los `<link>` y `<joint>` del URDF para que el modelo coincida con su TF. |
| Frame común | Publica en `/tf_static` `world → odom` (JetRover en el origen) y `world → create3/odom` (Create 3 a 1,5 m en Y; configurable con `OFFSET_X/OFFSET_Y`). |
| Hora | Re-sella los mensajes del Create 3 con el reloj del dominio 77. |
| Comandos | Recibe `Twist` en `/create3/cmd_vel` (lo que publica el panel Teleop de Foxglove) y lo envía como `TwistStamped` al `/cmd_vel` del Create 3 (Jazzy). |
| /tf_foxglove | Copia liviana de todo el `/tf` del dominio 77: solo los frames que se movieron, máx. 10 Hz, más un "latido" cada 2 s. Ver 2.8. |
| /clock | No se copia: cada simulador conserva su propio reloj. |

> **Nota.** Humble y Jazzy son distribuciones distintas pero comparten el formato de los mensajes estándar (`tf2_msgs`, `nav_msgs`, `std_msgs`, `geometry_msgs`). Los avisos `sequence size exceeds remaining buffer` del relay son ruido de esa convivencia y se pueden ignorar.

## 2.6 Árbol TF combinado

![Figura 7. Un solo árbol con raíz "world" para los dos robots.](img/d_tf.png)

*Figura 7. Un solo árbol con raíz "world" para los dos robots.*

- `odom → base_footprint` del JetRover lo publica Gazebo; las articulaciones, `robot_state_publisher`.
- `create3/odom → create3/base_link` lo publica el **diffdrive_controller** del Create 3 (si no está activo, el Create 3 no aparece).
- En Foxglove: **Fixed frame = Display frame = `world`**.

## 2.7 Foxglove y las mallas 3D

Foxglove es una app del host (no un nodo ROS). Se conecta por WebSocket a `foxglove_bridge` (`ws://localhost:8765`) y le **pide** las mallas del URDF.

| Concepto | Qué implica |
|---|---|
| package://paquete/… | El bridge busca el paquete en `AMENT_PREFIX_PATH`. Por eso T6 hace `source` del workspace y agrega `extra_pkgs/irobot_create_description`. |
| file:///… | El bridge lo rechaza ("Asset URI not allowed"). La malla `base_link` del JetRover se cambió a `package://`. |
| Marcador ament_index | Archivo vacío `share/ament_index/resource_index/packages/<paquete>` que dice "este paquete existe". Sin él: "Package [...] does not exist". |
| Mesh up-axis | Las mallas STL no indican su eje vertical. Con el valor por defecto (Y-up) el JetRover se ve volcado. Debe ser **Z-up**. |
| Capa URDF por tópico | Foxglove trata como modelo el tópico `/robot_description`. Para `/create3/robot_description` se agrega una **Custom layer → URDF** con Source = Topic. |

![Figura 8. Por qué copiar la carpeta install del Create 3 no alcanza: --symlink-install deja enlaces a src/ y build/.](img/d_symlink.png)

*Figura 8. Por qué copiar la carpeta install del Create 3 no alcanza: --symlink-install deja enlaces a src/ y build/.*

## 2.8 La carga del TF y el límite de Foxglove

Foxglove guarda en memoria **todas** las transformaciones que recibe, con un límite de **645 277 por tópico**. El `/tf` del JetRover más el del Create 3 son cientos de mensajes por segundo: el límite se alcanza en minutos y desde ese momento el brazo se congela y aparecen frames "Missing transform".

![Figura 9. Solución: el bridge no envía /tf; Foxglove usa /tf_foxglove.](img/d_tfload.png)

*Figura 9. Solución: el bridge no envía /tf; Foxglove usa /tf_foxglove.*

## 2.9 El reloj de Foxglove (use_sim_time)

Foxglove dibuja cada frame en la pose que corresponde a la hora de su barra inferior. Con el bridge en `use_sim_time: true` la barra quedaba clavada en `1969-12-31 8:00:00.000` (t = 0); el Create 3 (sellado por el relay) se movía pero el JetRover (sellado por Gazebo) quedaba congelado.

![Figura 10. Con el reloj real, Foxglove siempre toma la pose más reciente.](img/d_time.png)

*Figura 10. Con el reloj real, Foxglove siempre toma la pose más reciente.*

> **Resultado esperado.** Configuración final: `use_sim_time: false` en `foxglove_bridge.yaml`. La barra muestra la fecha y hora de hoy y los dos robots se mueven.

## 2.10 ros2_control, spawners y RTF

- Las articulaciones simuladas se mueven con **controladores** de ros2_control, que se cargan y activan con un **spawner**.
- Create 3: `diffdrive_controller` (ruedas, `/odom`). JetRover: `joint_state_broadcaster`, `arm_controller`, `gripper_controller`.
- **RTF** (Real Time Factor, abajo a la derecha en Gazebo): cuánto avanza la simulación por segundo real. Con dos Gazebos en una laptop con gráfica Intel se ven valores de **6–30 %**. Todo es más lento y los spawners pueden agotar su espera de 5 s: por eso se usa `--switch-timeout 60`.
- Para bajar la carga, el Create 3 se lanza con `use_rviz:=false` (Foxglove reemplaza a su RViz).

# Parte A – Instalación (una sola vez)

Si ya tienes las imágenes y el workspace del JetRover de los tutoriales anteriores, revisa cada paso y aplica solo lo que te falte (sobre todo A4 a A7).

## A1. Comprobar Docker

**HOST**

```bash
docker context use default
docker --version
docker run --rm hello-world
```

> **Nota.** Si existe Docker Desktop, el contexto puede quedar en `desktop-linux` ("Cannot connect to the Docker daemon"). `docker context use default` usa el Docker Engine nativo.

## A2. Create 3: imagen y contenedor

### A2.1 Dockerfile

**HOST**

```bash
mkdir -p ~/docker/create3_harmonic
cd ~/docker/create3_harmonic
nano Dockerfile
```

```dockerfile
FROM osrf/ros:jazzy-desktop
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
      ros-jazzy-ros-gz \
      ros-jazzy-gz-ros2-control \
      ros-jazzy-ros2-controllers \
      ros-jazzy-irobot-create-msgs \
      python3-colcon-common-extensions \
      python3-vcstool git mesa-utils \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /root/ws
RUN git clone -b jazzy https://github.com/iRobotEducation/create3_sim.git src/create3_sim && \
    apt-get update && rosdep update && \
    rosdep install --from-paths src -i -y && \
    rm -rf /var/lib/apt/lists/*
RUN . /opt/ros/jazzy/setup.sh && colcon build --symlink-install

RUN echo "source /opt/ros/jazzy/setup.bash" >> /root/.bashrc && \
    echo "source /root/ws/install/setup.bash" >> /root/.bashrc
WORKDIR /root
CMD ["bash"]
```

> **Atención.** Este Dockerfile es una **versión de referencia** reconstruida a partir de lo que usa la imagen `create3-jazzy:v1` (Jazzy, rama `jazzy` de create3_sim, workspace en `/root/ws` compilado con `--symlink-install`). Si ya tienes tu propio Dockerfile en `~/docker/create3_harmonic`, consérvalo y compáralo con este.

### A2.2 Construir la imagen

**HOST**

```bash
cd ~/docker/create3_harmonic
docker build -t create3-jazzy:v1 .
docker images | grep create3
```

### A2.3 run.sh con el dominio 78 fijo

**HOST**

```bash
cd ~/docker/create3_harmonic
nano run.sh
chmod +x run.sh
```

```bash
#!/bin/bash
xhost +local:docker > /dev/null
docker run -e ROS_DOMAIN_ID=78 -it --rm \
  --name create3 \
  --net=host \
  --ipc=host \
  --device /dev/dri \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  create3-jazzy:v1
```

Si ya tenías un `run.sh` sin dominio, basta con agregarlo así (con respaldo):

**HOST**

```bash
cd ~/docker/create3_harmonic
cp run.sh run.sh.bak
sed -i 's/docker run /docker run -e ROS_DOMAIN_ID=78 /' run.sh
grep -n "docker run" run.sh        # docker run -e ROS_DOMAIN_ID=78 ...
```

### A2.4 Primera prueba del Create 3

**HOST**

```bash
cd ~/docker/create3_harmonic
./run.sh
```

**CONTENEDOR create3**

```bash
echo $ROS_DOMAIN_ID                 # 78
source /opt/ros/jazzy/setup.bash
source /root/ws/install/setup.bash
ros2 launch irobot_create_gz_bringup create3_gz.launch.py use_rviz:=false
```

En otra terminal, activa las ruedas:

**HOST**

```bash
docker exec -it create3 bash
```

**CONTENEDOR create3**

```bash
source /opt/ros/jazzy/setup.bash
source /root/ws/install/setup.bash
ros2 run controller_manager spawner diffdrive_controller -c /controller_manager --switch-timeout 60
```

> **Resultado esperado.** "Configured and activated diffdrive_controller" (o un aviso de que ya está activo). Prueba los botones del panel Teleop de Gazebo: el Create 3 debe moverse.

> **Concepto.** Argumentos útiles de `create3_gz.launch.py` (ver con `--show-args`): `use_rviz`, `namespace`, `world`, `x`, `y`, `yaw`, `spawn_dock`.

## A3. JetRover: imagen, código y compilación

### A3.1 Clonar el código

**HOST**

```bash
mkdir -p ~/docker/jetrover/gz_cache
cd ~/docker/jetrover
git clone https://github.com/luongdev-bot/ros2_ws.git ws
find ws/src -name package.xml | wc -l       # 15
```

### A3.2 Dockerfile y build

**HOST**

```bash
cd ~/docker/jetrover
printf "ws\ngz_cache\n" > .dockerignore
nano Dockerfile
```

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
      python3-vcstool git mesa-utils \
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

**HOST**

```bash
docker build -t jetrover-humble:v2 .
docker images | grep jetrover               # ~6.4 GB
```

### A3.3 Parches al código (en el host)

**1) NVIDIA → Mesa** en los launch de Gazebo (el autor usaba GPU NVIDIA):

**HOST**

```bash
cd ~/docker/jetrover/ws
sed -i "s/'__NV_PRIME_RENDER_OFFLOAD', '1'/'__NV_PRIME_RENDER_OFFLOAD', '0'/; s/'__GLX_VENDOR_LIBRARY_NAME', 'nvidia'/'__GLX_VENDOR_LIBRARY_NAME', 'mesa'/" src/simulations/jetrover_gazebo/launch/*.launch.py
grep -Rn "NV_PRIME\|GLX_VENDOR" src/simulations
```

**2) robot_description como string** en `sim_display.launch.py`:

**HOST**

```bash
F=src/simulations/jetrover_description/launch/sim_display.launch.py
sed -i "s/robot_description = Command(\['xacro ', urdf_path\])/robot_description = ParameterValue(Command(['xacro ', urdf_path]), value_type=str)/" $F
grep -q "import ParameterValue" $F || sed -i "1i from launch_ros.parameter_descriptions import ParameterValue" $F
python3 -m py_compile $F && echo "sintaxis OK"
```

**3) Malla base_link: file:// → package://** (para que Foxglove la pueda pedir):

**HOST**

```bash
F=src/simulations/jetrover_description/urdf/car_mecanum.gazebo.urdf.xacro
sed -i 's|file://$(find jetrover_description)/|package://jetrover_description/|' $F
grep -n "base_link.stl" $F                   # package://jetrover_description/...
```

### A3.4 run.sh (dominio 77)

**HOST**

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
  -e ROS_DOMAIN_ID=77 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/docker/jetrover/ws:/root/ws \
  -v $HOME/docker/jetrover/gz_cache:/root/.ignition \
  jetrover-humble:v2
```

> **Atención.** Las líneas que terminan en `\` son UNA sola orden: no las pegues sueltas en la terminal ("-v: orden no encontrada"). Por eso van en `run.sh`.

### A3.5 Compilar el workspace

**HOST**

```bash
cd ~/docker/jetrover
./run.sh
```

**CONTENEDOR jetrover**

```bash
echo $MACHINE_TYPE $LIDAR_TYPE $ROS_DOMAIN_ID   # JetRover_Mecanum A1 77
cd /root/ws
colcon build --symlink-install --packages-up-to jetrover_gazebo jetrover_moveit_config
source /root/ws/install/setup.bash
```

> **Resultado esperado.** "Summary: 7 packages finished". Como `ws/` está en el host, no se repite cada día.

## A4. Mallas del Create 3 para el bridge del JetRover

El bridge de Foxglove corre en el contenedor del JetRover, pero las mallas del Create 3 están en el otro contenedor. Hay que copiar el paquete `irobot_create_description` a `extra_pkgs/` **con archivos reales** (ver Figura 8). Con los dos contenedores corriendo:

**HOST**

```bash
# 1) Carpeta que colcon ignora (no se compila)
mkdir -p ~/docker/jetrover/ws/extra_pkgs
touch ~/docker/jetrover/ws/extra_pkgs/COLCON_IGNORE

# 2) Copiar el paquete instalado (estructura share/, ament_index/...)
docker cp -L create3:/root/ws/install/irobot_create_description ~/docker/jetrover/ws/extra_pkgs/

# 3) Reemplazar las mallas (enlaces rotos) por los archivos reales de src/
D=~/docker/jetrover/ws/extra_pkgs/irobot_create_description
rm -rf $D/share/irobot_create_description/meshes
docker cp create3:/root/ws/src/create3_sim/irobot_create_common/irobot_create_description/meshes \
  $D/share/irobot_create_description/

# 4) Reemplazar el marcador de ament_index (otro enlace roto) por un archivo vacío
rm $D/share/ament_index/resource_index/packages/irobot_create_description
touch $D/share/ament_index/resource_index/packages/irobot_create_description

# 5) Verificar
ls -la $D/share/irobot_create_description/meshes        # .dae con -rw- y tamaño en KB/MB
ls -la $D/share/ament_index/resource_index/packages/     # archivo de 0 bytes, sin flecha
```

> **Concepto.** `find . -xtype l` dentro de `$D` lista enlaces rotos restantes (urdf, scripts de setup). No hacen falta: Foxglove solo pide las mallas y el modelo lo publica el contenedor del Create 3.

Comprobación desde el contenedor del JetRover:

**CONTENEDOR jetrover**

```bash
source /root/ws/install/setup.bash
export AMENT_PREFIX_PATH=/root/ws/extra_pkgs/irobot_create_description:$AMENT_PREFIX_PATH
ros2 pkg prefix irobot_create_description     # /root/ws/extra_pkgs/irobot_create_description
```

## A5. Archivos del dashboard

Son tres archivos (código completo en los Anexos A, B y C). Si los descargaste, muévelos así:

**HOST**

```bash
mkdir -p ~/docker/jetrover/ws/scripts
mv -f ~/Descargas/create3_relay.py     ~/docker/jetrover/ws/scripts/
mv -f ~/Descargas/foxglove_bridge.yaml ~/docker/jetrover/ws/scripts/
mv -f ~/Descargas/dashboard.sh         ~/docker/
chmod +x ~/docker/dashboard.sh

grep -c tf_foxglove ~/docker/jetrover/ws/scripts/create3_relay.py   # > 0 (versión 2)
grep use_sim_time   ~/docker/jetrover/ws/scripts/foxglove_bridge.yaml  # use_sim_time: false
```

| Archivo | Dónde queda (host → contenedor) | Lo usa |
|---|---|---|
| create3_relay.py | ~/docker/jetrover/ws/scripts → /root/ws/scripts | Terminal del relay |
| foxglove_bridge.yaml | ~/docker/jetrover/ws/scripts → /root/ws/scripts | Terminal del bridge |
| dashboard.sh | ~/docker (solo host) | Tú, para levantar o detener todo |

### foxglove_bridge.yaml

```yaml
# Parametros del foxglove_bridge para el dashboard Create 3 + JetRover.
# Uso (contenedor jetrover):
#   ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml
/**:
  ros__parameters:
    port: 8765
    use_sim_time: false
    # Todo menos /tf (Foxglove usa /tf_foxglove, que publica create3_relay.py)
    topic_whitelist: ['^(?!/tf$).*']
```

| Parámetro | Qué hace |
|---|---|
| /**: | Aplica a cualquier nodo que cargue el archivo |
| port: 8765 | Puerto de conexión de Foxglove |
| use_sim_time: false | Reloj real (ver 2.9) |
| topic_whitelist: ['^(?!/tf$).*'] | Pasa todos los tópicos menos /tf (expresión regular con "lookahead" negativo). /tf_static y /tf_foxglove sí pasan |

## A6. Instalar Foxglove

**HOST**

```bash
# Si ya tienes el .deb descargado desde foxglove.dev:
sudo apt install ~/foxglove-studio.deb
```

Abre la app e inicia sesión. El aviso rojo "Free plan… over the 3 user limit" es de la cuenta de la organización y no afecta la conexión local.

# Parte B – Uso: levantar todo y trabajar

## B1. Opción automática: dashboard.sh

**HOST**

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}"   # no deben existir create3 ni jetrover
~/docker/dashboard.sh
```

Debe ejecutarse desde una **terminal de GNOME**: abre una pestaña por tarea, en orden, y cada una espera a que la anterior esté lista.

![Figura 11. Las 8 pestañas que abre dashboard.sh.](img/d_terminals.png)

*Figura 11. Las 8 pestañas que abre dashboard.sh.*

| Pestaña | Qué hace | Espera a… |
|---|---|---|
| C3-base | ./run.sh del Create 3 (mantiene vivo el contenedor) | — |
| C3-sim | create3_gz.launch.py use_rviz:=false | Que exista el contenedor |
| C3-ruedas | spawner diffdrive_controller --switch-timeout 60 | Que responda el controller_manager |
| (30 s) | Pausa para repartir la carga de los dos Gazebos | — |
| JR-base | ./run.sh del JetRover | — |
| JR-sim | gazebo_moveit.launch.py | Que exista el contenedor |
| JR-relay | create3_relay.py | Que exista /clock |
| JR-lidar | TF estático del LiDAR | Que exista /clock |
| JR-fox | foxglove_bridge --params-file …yaml | Que exista /tf_foxglove |

> **Resultado esperado.** Cuando **JR-fox** diga "Server listening on port 8765", conecta Foxglove (B3).

> **Atención.** En **C3-base** y **JR-base** nunca escribas `exit`: borra el contenedor.
>
> En las demás, **Ctrl+C** detiene la tarea y te deja dentro del contenedor para relanzar a mano.
>
> Detener todo: `~/docker/dashboard.sh stop`.

## B2. Opción manual: terminal por terminal

Lo mismo que hace el script, por si quieres controlar cada paso.

### T1 – Simulación del Create 3

**HOST**

```bash
cd ~/docker/create3_harmonic && ./run.sh
```

**CONTENEDOR create3**

```bash
source /opt/ros/jazzy/setup.bash
source /root/ws/install/setup.bash
ros2 launch irobot_create_gz_bringup create3_gz.launch.py use_rviz:=false
```

### T2 – Ruedas del Create 3

**HOST**

```bash
docker exec -it create3 bash
```

**CONTENEDOR create3**

```bash
source /opt/ros/jazzy/setup.bash
source /root/ws/install/setup.bash
ros2 run controller_manager spawner diffdrive_controller -c /controller_manager --switch-timeout 60
```

### T3 – Simulación del JetRover

**HOST**

```bash
cd ~/docker/jetrover && ./run.sh
```

**CONTENEDOR jetrover**

```bash
source /root/ws/install/setup.bash
ros2 launch jetrover_gazebo gazebo_moveit.launch.py
```

- Cierra el panel **"Visualize lidar"** de Gazebo (ralentiza el arranque).
- Espera las tres líneas `Configured and activated` (joint_state_broadcaster, arm_controller, gripper_controller).
- Verifica que el JetRover esté **de pie**; si volcó, relanza T3.

### T4 – Relay

**HOST**

```bash
docker exec -it jetrover bash
```

**CONTENEDOR jetrover**

```bash
source /root/ws/install/setup.bash
python3 /root/ws/scripts/create3_relay.py
```

> **Resultado esperado.** `Relay listo: dominio 78 -> 77, prefijo "create3/"`
>
> `robot_description del Create 3 republicado con prefijo`

> **Nota.** Arráncalo **después** de la simulación del JetRover. Si reinicias la simulación del JetRover, reinicia también el relay.

### T5 – TF del LiDAR

**HOST**

```bash
docker exec -it jetrover bash
```

**CONTENEDOR jetrover**

```bash
source /root/ws/install/setup.bash
ros2 run tf2_ros static_transform_publisher \
  --frame-id lidar_frame --child-frame-id jetrover/base_footprint/lidar
```

### T6 – Bridge de Foxglove

**HOST**

```bash
docker exec -it jetrover bash
```

**CONTENEDOR jetrover**

```bash
source /root/ws/install/setup.bash
export AMENT_PREFIX_PATH=/root/ws/extra_pkgs/irobot_create_description:$AMENT_PREFIX_PATH
ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml
```

| Línea | Por qué |
|---|---|
| source …/setup.bash | Sin esto el bridge no encuentra jetrover_description y el JetRover es invisible |
| export AMENT_PREFIX_PATH=… | Para que encuentre irobot_create_description (mallas del Create 3) |
| --params-file …yaml | Puerto, reloj real y exclusión de /tf |

## B3. Configurar Foxglove

### Conectar

1. **Open connection → Foxglove WebSocket →** `ws://localhost:8765` **→ Open**.
2. Elige el layout **"Create3 y JetRover Dashboard"** (o crea uno: menú de Layouts, arriba a la derecha).
3. La barra inferior debe mostrar la **fecha y hora de hoy**.

### Panel 3D

| Sección | Ajuste | Valor |
|---|---|---|
| Frame | Fixed frame / Display frame | world / world |
| Frame | Follow mode | Off |
| Scene | Mesh up-axis | Z-up (imprescindible) |
| Scene | Ignore COLLADA &lt;up_axis&gt; | Off |
| Topics | /robot_description (JetRover) | Visible · Control mode = Transforms |
| Topics | /scan | Visible · Point size = 4 |
| Topics | Tópicos de imagen (/depth_cam/*, /filtered_*, /model_depth…) | Ocultos (saturan el buffer de 250 MB) |
| Custom layers | ⋮ → Add URDF · Source = Topic · Topic = /create3/robot_description | Modelo del Create 3 |
| /robot_description y URDF | Show axis | Off (On solo para diagnosticar) |

> **Nota.** En la pestaña **Topics** de la izquierda debe aparecer `/tf_foxglove` y **no** `/tf`. Es correcto: Foxglove usa cualquier tópico de tipo TFMessage.

### Panel Teleop para el Create 3

1. Botón **+** de paneles (arriba a la derecha) → **Teleop**.
2. En sus ajustes: **Topic = `/create3/cmd_vel`**.
3. Mantén pulsadas las flechas: el Create 3 se mueve en Gazebo y en Foxglove.

### Guardar el layout

Menú de Layouts → **Save** sobre "Create3 y JetRover Dashboard".

![Figura 12. Dashboard funcionando: hora real en la barra, los dos robots y un panel Raw Messages.](img/final_clock.png)

*Figura 12. Dashboard funcionando: hora real en la barra, los dos robots y un panel Raw Messages.*

## B4. Mover los robots

### Brazo del JetRover (RViz / MoveIt)

1. Panel **MotionPlanning → Planning**. **Planning Group = `arm`**.
2. **Start State = `<current>`** (con otro valor, Execute falla).
3. **Goal State**: `<random valid>`, `home`, o arrastra la esfera/flechas del extremo del brazo.
4. Pulsa **Plan & Execute** (mejor que Plan y luego Execute por separado).
5. Mira Gazebo y Foxglove. Con RTF bajo el movimiento tarda 5–15 s.

![Figura 13. Pestaña Planning: grupo arm, Start State &lt;current&gt;, Goal State &lt;random valid&gt;.](img/rviz_planning.png)

*Figura 13. Pestaña Planning: grupo arm, Start State &lt;current&gt;, Goal State &lt;random valid&gt;.*

> **Atención.** Si sale **Failed** tras Plan → Execute: el brazo se movió entre los dos clics y el plan quedó viejo. Usa **Plan & Execute**.
>
> Para el brazo vertical: pestaña **Joints**, joint1…joint5 = 0, y Plan & Execute.

### Chasis del JetRover

**CONTENEDOR jetrover**

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"   # adelante
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {y: 0.2}}"   # lateral (Mecanum)
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{angular: {z: 0.5}}"  # giro
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}"                   # PARAR
```

### Create 3

- Botones del panel **Teleop** de su Gazebo.
- Panel **Teleop** de Foxglove (`/create3/cmd_vel`).
- Desde el contenedor del JetRover, a través del relay:

**CONTENEDOR jetrover**

```bash
timeout 3 ros2 topic pub -r 10 /create3/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}}"
```

> **Nota.** Directamente en el dominio 78, el Create 3 (Jazzy) espera **TwistStamped**: `ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {linear: {x: 0.1}}}"`. Con `Twist` no se mueve.

## B5. Apagar

**HOST**

```text
~/docker/dashboard.sh stop
```

O a mano: Ctrl+C en cada terminal y, al final, Ctrl+C + `exit` en las terminales principales (T1 y T3). Comprueba con `docker ps -a` que no queda `create3` ni `jetrover`.

> **Nota.** Los "Segmentation fault" al cerrar Gazebo o move_group son fallos en los destructores al apagar: inofensivos.

# Parte C – Problemas y soluciones

## C1. Foxglove

| Síntoma / mensaje | Causa | Solución |
|---|---|---|
| JetRover invisible (solo LiDAR); "Package [jetrover_description] does not exist" | Bridge sin source del workspace | source /root/ws/install/setup.bash y relanzar el bridge |
| base_link: "Asset URI not allowed" | Malla con file:// | Parche A3.3-3 (package://) y relanzar la simulación |
| JetRover volcado, piezas flotando | Mesh up-axis = Y-up | Scene → Mesh up-axis = Z-up |
| Create 3: solo ruedas y rueda loca; "Failed to load mesh … body_visual.dae" | Mallas copiadas como enlaces rotos | A4 paso 3 (copiar meshes desde src/) |
| "Package [irobot_create_description] does not exist" en el log del bridge | Marcador ament_index roto o falta el export | A4 paso 4 y export AMENT_PREFIX_PATH |
| Fixed frame en rojo | Layout copiado con base_footprint | Fixed/Display frame = world |
| "Missing transform from frame &lt;create3/base_link&gt; to frame &lt;world&gt;" | diffdrive_controller inactivo, o TF saturado | C2 (controlador) o reconectar Foxglove |
| "At most 645277 transforms … on topic /tf" | Foxglove guarda todo el /tf | Bridge con foxglove_bridge.yaml (sin /tf) + relay v2 |
| Barra en 1969-12-31 8:00:00.000 y brazo congelado | use_sim_time: true | use_sim_time: false en el yaml |
| "296 Infinity invalid values" en /scan | Rayos del LiDAR que no tocan nada | Normal; ignorar |
| "Frame buffer limit (250 MB) reached" | Imágenes suscritas en el 3D | Ocultar tópicos de imagen y reconectar |
| /create3/robot_description no aparece en Topics | Foxglove solo reconoce /robot_description | Custom layers → Add URDF (Source = Topic) |
| Raw Messages: "Waiting to diff…" | Panel en modo comparación | Clic en el primer ícono (diff) para apagarlo |

![Figura 14. Create 3 sin cuerpo ni parachoques: las mallas .dae eran enlaces rotos.](img/c3_wheels.png)

*Figura 14. Create 3 sin cuerpo ni parachoques: las mallas .dae eran enlaces rotos.*

![Figura 15. Límite de TF alcanzado: todos los frames create3/ en rojo y el brazo congelado.](img/err_tflimit.png)

*Figura 15. Límite de TF alcanzado: todos los frames create3/ en rojo y el brazo congelado.*

![Figura 16. Mesh up-axis = Y-up: el JetRover se ve volcado aunque su TF es correcto.](img/fox_broken.png)

*Figura 16. Mesh up-axis = Y-up: el JetRover se ve volcado aunque su TF es correcto.*

## C2. Simulación y controladores

| Síntoma / mensaje | Causa | Solución |
|---|---|---|
| Create 3 no se mueve; "Failed to activate controller: diffdrive_controller" | RTF bajo: la activación supera los 5 s | spawner … --switch-timeout 60 |
| "Switch controller timed out after 5 seconds" al lanzar el Create 3 | Igual que arriba (spawner del propio launch) | Activarlo a mano en T2 |
| Create 3 no responde a Twist | En Jazzy /cmd_vel es TwistStamped | Usar TwistStamped o /create3/cmd_vel vía relay |
| "ros2: invalid choice: 'control'" | ros2controlcli no está en la imagen del Create 3 | ros2 service call /controller_manager/list_controllers controller_manager_msgs/srv/ListControllers |
| MoveIt: Execute → Failed | Start State distinto de &lt;current&gt; o plan viejo | Start State = &lt;current&gt; y Plan & Execute |
| JetRover volcado en Gazebo | Movimiento brusco con física lenta | Relanzar T3 (y luego el relay) |
| Todo muy lento, RTF 6–30 % | Dos Gazebos + MoveIt + RViz en la laptop | use_rviz:=false en el Create 3; cerrar "Visualize lidar" |
| Spawners del JetRover colgados | Arranque lento | Esperar 1–2 min; relanzar T3 |

## C3. Docker, dominios y CLI de ROS

| Síntoma / mensaje | Causa | Solución |
|---|---|---|
| "No such container: jetrover" tras escribir exit | exit en la terminal principal (--rm) | Volver a ./run.sh; no se pierde nada de ws/ |
| "bash: docker: command not found" | docker ejecutado dentro del contenedor | docker solo en el HOST |
| "Permiso denegado" con rutas /root/… | Ruta del contenedor usada en el host | En el host: ~/docker/jetrover/ws/… |
| "/opt/ros/jazzy/setup.bash: No existe" | Comando de Jazzy en el host (Humble) | Entrar al contenedor create3 |
| Árbol TF partido, datos de otro robot | Dominio compartido (Zenoh en 42) | Dominios 77 y 78 |
| xmlrpc … Fault: "!rclpy.ok()" | Daemon de ros2 trabado | ros2 daemon stop && ros2 daemon start |
| "failed to initialize wait set" | CLI de ros2 sobrecargado | Repetir; usar --no-daemon; preferir Foxglove para verificar |
| ros2 topic echo --once no termina | Nadie publica (o descubrimiento lento) | timeout 10 ros2 topic echo … |
| hz \| tail no imprime nada | Python bufferiza la salida en pipes | No usar pipes con hz; o PYTHONUNBUFFERED=1 |
| "sequence size exceeds remaining buffer" | Mensajes Jazzy leídos desde Humble | Ruido inofensivo |

# Parte D – Comandos esenciales del día a día

## D1. Arrancar y detener

| Acción | Dónde | Comando |
|---|---|---|
| Levantar todo | HOST | ~/docker/dashboard.sh |
| Detener todo | HOST | ~/docker/dashboard.sh stop |
| Ver contenedores | HOST | docker ps -a --format "table {{.Names}}\t{{.Status}}" |
| Contenedor Create 3 | HOST | cd ~/docker/create3_harmonic && ./run.sh |
| Contenedor JetRover | HOST | cd ~/docker/jetrover && ./run.sh |
| Otra terminal | HOST | docker exec -it create3 bash   \|   docker exec -it jetrover bash |
| Detener un contenedor colgado | HOST | docker stop jetrover   \|   docker stop create3 |

## D2. Create 3 (contenedor create3)

| Acción | Comando |
|---|---|
| Cargar ROS | source /opt/ros/jazzy/setup.bash && source /root/ws/install/setup.bash |
| Simulación | ros2 launch irobot_create_gz_bringup create3_gz.launch.py use_rviz:=false |
| Activar ruedas | ros2 run controller_manager spawner diffdrive_controller -c /controller_manager --switch-timeout 60 |
| Estado controladores | ros2 service call /controller_manager/list_controllers controller_manager_msgs/srv/ListControllers \| grep -oE "name='[a-z_]+'\|state='[a-z]+'" |
| Odometría | timeout 15 ros2 topic hz /odom |
| Mover | ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {linear: {x: 0.1}}}" |
| Dominio | echo $ROS_DOMAIN_ID   → 78 |

## D3. JetRover y dashboard (contenedor jetrover)

| Acción | Comando |
|---|---|
| Cargar ROS | source /root/ws/install/setup.bash |
| Simulación | ros2 launch jetrover_gazebo gazebo_moveit.launch.py |
| Relay | python3 /root/ws/scripts/create3_relay.py |
| TF del LiDAR | ros2 run tf2_ros static_transform_publisher --frame-id lidar_frame --child-frame-id jetrover/base_footprint/lidar |
| Mallas Create 3 | export AMENT_PREFIX_PATH=/root/ws/extra_pkgs/irobot_create_description:$AMENT_PREFIX_PATH |
| Bridge | ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml |
| Mover chasis | ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}" |
| Parar chasis | ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}" |
| Mover Create 3 (relay) | timeout 3 ros2 topic pub -r 10 /create3/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}}" |
| Dominio | echo $ROS_DOMAIN_ID   → 77 |

## D4. Verificación y diagnóstico

| Pregunta | Comando (contenedor jetrover) |
|---|---|
| ¿Llega el Create 3? | ros2 topic list \| grep create3 |
| ¿Odometría con prefijo? | timeout 10 ros2 topic echo /create3/odom --once \| head -8 |
| ¿Create 3 colgado de world? | ros2 run tf2_ros tf2_echo world create3/base_link |
| ¿JetRover colgado de world? | ros2 run tf2_ros tf2_echo world base_footprint |
| ¿TF liviano activo? | ros2 topic hz /tf_foxglove |
| ¿Reloj de Gazebo? | timeout 10 ros2 topic hz /clock |
| ¿Quién publica un tópico? | ros2 topic info /clock -v \| grep -E "Node name\|Reliability\|Durability" |
| ¿Brazo listo? | ros2 action list \| grep follow_joint_trajectory |
| ¿Paquete visible? | ros2 pkg prefix irobot_create_description |
| CLI trabado | ros2 daemon stop && ros2 daemon start |
| Log del launch | LOG=$(ls -td /root/.ros/log/*/ \| head -1)launch.log; grep -E "ERROR\|activated" $LOG \| tail -20 |

## D5. Foxglove

| Acción | Dónde / cómo |
|---|---|
| Conectar | Open connection → Foxglove WebSocket → ws://localhost:8765 |
| Frames | Fixed frame = Display frame = world |
| Robot bien dibujado | Scene → Mesh up-axis = Z-up |
| Modelo del Create 3 | Custom layers → Add URDF → Topic /create3/robot_description |
| Conducir el Create 3 | Panel Teleop → Topic /create3/cmd_vel |
| Ver un error | Pasar el mouse sobre el ícono rojo · pestaña Problems |
| Limpiar estado | Desconectar y reconectar |
| Guardar | Layouts → Save |

## D6. Las 5 reglas de oro

1. **Docker y git en el HOST; ROS en el CONTENEDOR.**
2. **Nunca `exit`** en la terminal principal de un contenedor (la de `./run.sh`).
3. **Siempre `source`** antes de `ros2 launch`, `ros2 run` y del bridge.
4. Orden: **Create 3 → ruedas → JetRover → relay → bridge → Foxglove**. Si reinicias el JetRover, reinicia el relay.
5. Si algo se ve mal en Foxglove: **pasa el mouse sobre el ícono rojo** y lee el mensaje antes de tocar nada.

# Anexo A – create3_relay.py

Ubicación: `~/docker/jetrover/ws/scripts/create3_relay.py` (host) = `/root/ws/scripts/create3_relay.py` (contenedor jetrover). Los parámetros editables están en el bloque "Configuración".

**CONTENEDOR jetrover · python3**

```python
#!/usr/bin/env python3
"""
create3_relay.py  -  Trae el Create 3 (dominio 78) al mundo del JetRover (dominio 77).

  Dominio 78 (Create 3, Jazzy)           Dominio 77 (JetRover, Humble)
  /tf, /tf_static         ──prefijo──►   /tf, /tf_static   (create3/odom, create3/base_link, ...)
  /robot_description      ──prefijo──►   /create3/robot_description
  /odom                   ──prefijo──►   /create3/odom
  /cmd_vel (TwistStamped) ◄──convierte── /create3/cmd_vel (Twist, p.ej. Teleop de Foxglove)
  /clock                  ✗ no se copia (cada simulador conserva su reloj)

Ademas publica en el dominio 77 un frame comun 'world':
  world -> odom           (JetRover, en el origen)
  world -> create3/odom   (Create 3, desplazado OFFSET_X / OFFSET_Y)

Y un /tf "liviano" para Foxglove:
  /tf (todo, cientos de msgs/s) ──► /tf_foxglove (solo frames que se movieron,
                                    max FOX_RATE Hz + latido cada FOX_HEARTBEAT s)
  Foxglove guarda TODO el /tf que recibe y se satura (limite 645277 transforms).
  El bridge se lanza sin /tf (topic_whitelist) y Foxglove usa /tf_foxglove.

Uso (dentro del contenedor del JetRover):
  python3 /root/ws/scripts/create3_relay.py
"""
import copy
import math
import threading
import time
import xml.etree.ElementTree as ET

import rclpy
from rclpy.context import Context
from rclpy.executors import SingleThreadedExecutor, ExternalShutdownException
from rclpy.clock import Clock, ClockType
from rclpy.node import Node
from rclpy.parameter import Parameter
from rclpy.qos import QoSProfile, DurabilityPolicy, ReliabilityPolicy, HistoryPolicy

from geometry_msgs.msg import Twist, TwistStamped, TransformStamped
from nav_msgs.msg import Odometry
from std_msgs.msg import String
from tf2_msgs.msg import TFMessage

# ---------------- Configuracion ----------------
CREATE3_DOMAIN = 78
JETROVER_DOMAIN = 77
PREFIX = 'create3/'
WORLD = 'world'
JETROVER_ROOT = 'odom'      # frame raiz del arbol TF del JetRover
OFFSET_X = 0.0              # posicion del Create 3 respecto del JetRover (m)
OFFSET_Y = 1.5
FOX_RATE = 10.0             # Hz maximos de /tf_foxglove
FOX_HEARTBEAT = 2.0         # s: reenviar un frame quieto cada tanto
FOX_MIN_MOVE = 0.001        # m / rad: cambio minimo para reenviar
# ------------------------------------------------

QOS_LATCHED = QoSProfile(depth=1, history=HistoryPolicy.KEEP_LAST,
                         reliability=ReliabilityPolicy.RELIABLE,
                         durability=DurabilityPolicy.TRANSIENT_LOCAL)
QOS_STATIC_SUB = QoSProfile(depth=100, history=HistoryPolicy.KEEP_LAST,
                            reliability=ReliabilityPolicy.RELIABLE,
                            durability=DurabilityPolicy.TRANSIENT_LOCAL)


def pfx(frame: str) -> str:
    """'base_link' -> 'create3/base_link' (sin duplicar el prefijo)."""
    frame = frame.lstrip('/')
    return frame if frame.startswith(PREFIX) else PREFIX + frame


def prefix_urdf(urdf: str) -> str:
    """Renombra links y referencias de joints para que coincidan con los frames TF."""
    root = ET.fromstring(urdf)
    for link in root.iter('link'):
        if 'name' in link.attrib:
            link.set('name', pfx(link.get('name')))
    for joint in root.iter('joint'):
        for tag in ('parent', 'child'):
            el = joint.find(tag)
            if el is not None and 'link' in el.attrib:
                el.set('link', pfx(el.get('link')))
    return ET.tostring(root, encoding='unicode')

class JetRoverSide(Node):
    """Nodo en el dominio 77: publica lo que vera Foxglove."""

    def __init__(self, context):
        super().__init__('create3_relay_out', context=context,
                         parameter_overrides=[Parameter('use_sim_time', value=True)])
        self.tf_pub = self.create_publisher(TFMessage, '/tf', 100)
        self.tf_static_pub = self.create_publisher(TFMessage, '/tf_static', QOS_LATCHED)
        self.desc_pub = self.create_publisher(String, '/create3/robot_description', QOS_LATCHED)
        self.odom_pub = self.create_publisher(Odometry, '/create3/odom', 10)
        self.cmd_callback = None   # se asigna desde main()
        self.create_subscription(Twist, '/create3/cmd_vel', self._on_cmd, 10)
        # --- /tf liviano para Foxglove ---
        self.fox_pub = self.create_publisher(TFMessage, '/tf_foxglove', 100)
        self.fox_latest = {}       # child -> ultimo TransformStamped recibido
        self.fox_sent = {}         # child -> (TransformStamped enviado, hora de pared)
        self.fox_lock = threading.Lock()
        self.create_subscription(TFMessage, '/tf', self._on_any_tf, 100)
        self.create_timer(1.0 / FOX_RATE, self._fox_tick,
                          clock=Clock(clock_type=ClockType.STEADY_TIME))

    def _on_any_tf(self, msg):
        with self.fox_lock:
            for t in msg.transforms:
                self.fox_latest[t.child_frame_id] = t

    @staticmethod
    def _moved(a, b):
        ta, tb = a.transform.translation, b.transform.translation
        ra, rb = a.transform.rotation, b.transform.rotation
        dpos = math.dist((ta.x, ta.y, ta.z), (tb.x, tb.y, tb.z))
        dot = abs(ra.x * rb.x + ra.y * rb.y + ra.z * rb.z + ra.w * rb.w)
        dang = 2.0 * math.acos(min(1.0, dot))
        return (dpos > FOX_MIN_MOVE or dang > FOX_MIN_MOVE
                or a.header.frame_id != b.header.frame_id)

    def _fox_tick(self):
        now = time.monotonic()
        out = TFMessage()
        with self.fox_lock:
            for child, t in self.fox_latest.items():
                prev = self.fox_sent.get(child)
                if prev is None or now - prev[1] > FOX_HEARTBEAT or self._moved(t, prev[0]):
                    out.transforms.append(t)
                    self.fox_sent[child] = (t, now)
        if out.transforms:
            self.fox_pub.publish(out)

    def _on_cmd(self, msg):
        if self.cmd_callback:
            self.cmd_callback(msg)

    def now(self):
        return self.get_clock().now().to_msg()


class Create3Side(Node):
    """Nodo en el dominio 78: escucha al Create 3 y le manda comandos."""

    def __init__(self, context, out: JetRoverSide):
        super().__init__('create3_relay_in', context=context)
        self.out = out
        self.static = {}           # child_frame -> TransformStamped (acumulado)
        self.lock = threading.Lock()
        self.cmd_pub = self.create_publisher(TwistStamped, '/cmd_vel', 10)
        self.create_subscription(TFMessage, '/tf', self._on_tf, 100)
        self.create_subscription(TFMessage, '/tf_static', self._on_tf_static, QOS_STATIC_SUB)
        self.create_subscription(String, '/robot_description', self._on_desc, QOS_LATCHED)
        self.create_subscription(Odometry, '/odom', self._on_odom, 10)
        self._publish_static()     # publica 'world' aunque el Create 3 aun no este
        self.get_logger().info(
            f'Relay listo: dominio {CREATE3_DOMAIN} -> {JETROVER_DOMAIN}, prefijo "{PREFIX}"')

    # --- TF dinamico: prefijo + hora del JetRover ---
    def _on_tf(self, msg):
        stamp = self.out.now()
        out = TFMessage()
        for t in msg.transforms:
            t2 = copy.deepcopy(t)
            t2.header.stamp = stamp
            t2.header.frame_id = pfx(t.header.frame_id)
            t2.child_frame_id = pfx(t.child_frame_id)
            out.transforms.append(t2)
        self.out.tf_pub.publish(out)

    # --- TF estatico: se acumula todo y se republica completo ---
    def _on_tf_static(self, msg):
        with self.lock:
            for t in msg.transforms:
                t2 = copy.deepcopy(t)
                t2.header.frame_id = pfx(t.header.frame_id)
                t2.child_frame_id = pfx(t.child_frame_id)
                self.static[t2.child_frame_id] = t2
        self._publish_static()

    def _world_tf(self, child, x, y):
        t = TransformStamped()
        t.header.frame_id = WORLD
        t.child_frame_id = child
        t.transform.translation.x = float(x)
        t.transform.translation.y = float(y)
        t.transform.rotation.w = 1.0
        return t

    def _publish_static(self):
        stamp = self.out.now()
        with self.lock:
            transforms = [self._world_tf(JETROVER_ROOT, 0.0, 0.0),
                          self._world_tf(pfx('odom'), OFFSET_X, OFFSET_Y)]
            transforms += list(self.static.values())
        for t in transforms:
            t.header.stamp = stamp
        self.out.tf_static_pub.publish(TFMessage(transforms=transforms))

    # --- Modelo: links renombrados ---
    def _on_desc(self, msg):
        try:
            self.out.desc_pub.publish(String(data=prefix_urdf(msg.data)))
            self.get_logger().info('robot_description del Create 3 republicado con prefijo')
        except ET.ParseError as e:
            self.get_logger().error(f'No se pudo leer el URDF: {e}')

    # --- Odometria ---
    def _on_odom(self, msg):
        m = copy.deepcopy(msg)
        m.header.stamp = self.out.now()
        m.header.frame_id = pfx(msg.header.frame_id)
        m.child_frame_id = pfx(msg.child_frame_id)
        self.out.odom_pub.publish(m)

    # --- Comandos: Twist (dominio 77) -> TwistStamped (dominio 78) ---
    def send_cmd(self, twist: Twist):
        ts = TwistStamped()
        ts.header.frame_id = 'base_link'
        # stamp en 0: el diff_drive_controller del Create 3 lo reemplaza por su hora actual
        ts.twist = twist
        self.cmd_pub.publish(ts)


def main():
    ctx_c3, ctx_jr = Context(), Context()
    rclpy.init(context=ctx_c3, domain_id=CREATE3_DOMAIN)
    rclpy.init(context=ctx_jr, domain_id=JETROVER_DOMAIN)

    out = JetRoverSide(ctx_jr)
    inp = Create3Side(ctx_c3, out)
    out.cmd_callback = inp.send_cmd

    ex_jr = SingleThreadedExecutor(context=ctx_jr)
    ex_c3 = SingleThreadedExecutor(context=ctx_c3)
    ex_jr.add_node(out)
    ex_c3.add_node(inp)
    threading.Thread(target=ex_jr.spin, daemon=True).start()
    try:
        ex_c3.spin()
    except (KeyboardInterrupt, ExternalShutdownException):
        pass  # Ctrl+C
    finally:
        inp.destroy_node()
        out.destroy_node()
        rclpy.try_shutdown(context=ctx_c3)
        rclpy.try_shutdown(context=ctx_jr)


if __name__ == '__main__':
    main()
```

# Anexo B – foxglove_bridge.yaml

```yaml
# Parametros del foxglove_bridge para el dashboard Create 3 + JetRover.
# Uso (contenedor jetrover):
#   ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml
/**:
  ros__parameters:
    port: 8765
    use_sim_time: false
    # Todo menos /tf (Foxglove usa /tf_foxglove, que publica create3_relay.py)
    topic_whitelist: ['^(?!/tf$).*']
```

# Anexo C – dashboard.sh

Ubicación: `~/docker/dashboard.sh` (host). Requiere `gnome-terminal`.

**HOST · bash**

```bash
#!/usr/bin/env bash
# =============================================================================
#  dashboard.sh  -  Levanta (o detiene) el dashboard Foxglove Create 3 + JetRover
#
#  Uso (en el HOST, desde una terminal de GNOME):
#     ~/docker/dashboard.sh          # levanta todo (abre una pestaña por tarea)
#     ~/docker/dashboard.sh stop     # detiene ambos contenedores
#
#  Pestañas que abre:
#     C3-base   contenedor create3   (./run.sh)          <- NO escribir exit
#     C3-sim    simulacion Create 3  (dominio 78)
#     C3-ruedas spawner diffdrive_controller
#     JR-base   contenedor jetrover  (./run.sh)          <- NO escribir exit
#     JR-sim    simulacion JetRover  (Gazebo + MoveIt + RViz, dominio 77)
#     JR-relay  create3_relay.py     (78 -> 77, /tf_foxglove)
#     JR-lidar  TF estatico del LiDAR
#     JR-fox    foxglove_bridge      (ws://localhost:8765)
# =============================================================================

C3_DIR="$HOME/docker/create3_harmonic"
JR_DIR="$HOME/docker/jetrover"
C3=create3
JR=jetrover
SELF="$(readlink -f "$0")"

C3_SRC='export ROS_DOMAIN_ID=78; source /opt/ros/jazzy/setup.bash; source /root/ws/install/setup.bash'
JR_SRC='source /root/ws/install/setup.bash'

# ---------- utilidades ----------
title()  { echo -ne "\033]0;$1\007"; echo -e "\n=== $1 ===\n"; }

wait_container() {            # espera a que exista el contenedor $1
  echo "Esperando el contenedor '$1'..."
  until docker ps --format '{{.Names}}' | grep -qx "$1"; do sleep 1; done
  echo "Contenedor '$1' listo."
}

# Ejecuta $2 dentro del contenedor $1. Al terminar (o con Ctrl+C) queda un
# bash dentro del contenedor para seguir trabajando.
in_container() {
  docker exec -it "$1" bash -c "trap : INT; $2; echo; echo '[terminado - sigues dentro del contenedor $1]'; exec bash"
}

open_tab() {                  # abre una pestaña nueva que ejecuta: dashboard.sh job <nombre>
  gnome-terminal --tab -- bash -c "'$SELF' job $1; exec bash"
  sleep 1
}

# ---------- tareas (una por pestaña) ----------
job() {
  case "$1" in
    c3_base)
      title "C3-base (contenedor create3 - NO escribir exit)"
      cd "$C3_DIR" && ./run.sh ;;

    c3_sim)
      title "C3-sim (Create 3, dominio 78)"
      wait_container $C3
      in_container $C3 "$C3_SRC; ros2 launch irobot_create_gz_bringup create3_gz.launch.py use_rviz:=false" ;;

    c3_ruedas)
      title "C3-ruedas (diffdrive_controller)"
      wait_container $C3
      in_container $C3 "$C3_SRC
        echo 'Esperando al controller_manager...'
        until ros2 service list --no-daemon 2>/dev/null | grep -q /controller_manager/list_controllers; do sleep 3; done
        sleep 5
        ros2 run controller_manager spawner diffdrive_controller -c /controller_manager --switch-timeout 60" ;;

    jr_base)
      title "JR-base (contenedor jetrover - NO escribir exit)"
      cd "$JR_DIR" && ./run.sh ;;

    jr_sim)
      title "JR-sim (JetRover: Gazebo + MoveIt + RViz, dominio 77)"
      wait_container $JR
      in_container $JR "$JR_SRC; ros2 launch jetrover_gazebo gazebo_moveit.launch.py" ;;

    jr_relay)
      title "JR-relay (create3_relay.py)"
      wait_container $JR
      in_container $JR "$JR_SRC
        echo 'Esperando /clock del JetRover...'
        until ros2 topic list --no-daemon 2>/dev/null | grep -qx /clock; do sleep 3; done
        python3 /root/ws/scripts/create3_relay.py" ;;

    jr_lidar)
      title "JR-lidar (TF estatico del LiDAR)"
      wait_container $JR
      in_container $JR "$JR_SRC
        until ros2 topic list --no-daemon 2>/dev/null | grep -qx /clock; do sleep 3; done
        ros2 run tf2_ros static_transform_publisher --frame-id lidar_frame --child-frame-id jetrover/base_footprint/lidar" ;;

    jr_fox)
      title "JR-fox (foxglove_bridge, ws://localhost:8765)"
      wait_container $JR
      in_container $JR "$JR_SRC
        export AMENT_PREFIX_PATH=/root/ws/extra_pkgs/irobot_create_description:\$AMENT_PREFIX_PATH
        echo 'Esperando /tf_foxglove (relay)...'
        until ros2 topic list --no-daemon 2>/dev/null | grep -qx /tf_foxglove; do sleep 3; done
        ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml" ;;

    *) echo "Tarea desconocida: $1"; exit 1 ;;
  esac
}

# ---------- modos ----------
case "${1:-start}" in
  job)
    job "$2" ;;

  stop)
    echo "Deteniendo contenedores $JR y $C3..."
    docker stop $JR $C3 2>/dev/null
    docker ps -a --format 'table {{.Names}}\t{{.Status}}' ;;

  start)
    if docker ps -a --format '{{.Names}}' | grep -qxE "$C3|$JR"; then
      echo "Ya existe un contenedor '$C3' o '$JR':"
      docker ps -a --format 'table {{.Names}}\t{{.Status}}'
      echo "Ejecuta primero:  $SELF stop"
      exit 1
    fi
    command -v gnome-terminal >/dev/null || { echo "No se encontro gnome-terminal"; exit 1; }

    echo "Abriendo pestañas del Create 3..."
    open_tab c3_base; open_tab c3_sim; open_tab c3_ruedas

    echo "Esperando 30 s antes del JetRover (para repartir la carga)..."
    sleep 30
    echo "Abriendo pestañas del JetRover..."
    open_tab jr_base; open_tab jr_sim; open_tab jr_relay; open_tab jr_lidar; open_tab jr_fox

    echo
    echo "Listo. Cuando JR-fox diga 'Server listening on port 8765', conecta Foxglove a ws://localhost:8765"
    echo "Para detener todo:  $SELF stop" ;;

  *)
    echo "Uso: $SELF [start|stop]"; exit 1 ;;
esac
```
