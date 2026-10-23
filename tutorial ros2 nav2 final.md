**SLAM y Navegación Autónoma con TurtleBot3**

De cero a un robot navegando solo, en simulación con Gazebo. Este documento reúne los conceptos de robótica vistos y documenta paso a paso el trabajo práctico realizado: instalar el entorno, mapear un mundo simulado con SLAM, y lograr que un robot navegue de forma autónoma entre varios puntos.

# Parte 1 — Conceptos fundamentales

## 1.1 Percepción: PointCloud y LaserScan

Un robot necesita "ver" su entorno para poder moverse sin chocar. Los dos formatos más comunes en ROS 2 para representar esa percepción son la nube de puntos y el LaserScan.

### PointCloud (nube de puntos)

Una nube de puntos (sensor_msgs/msg/PointCloud2) es una colección de puntos en el espacio, cada uno con coordenadas (x, y, z) y opcionalmente color o intensidad. Se genera con sensores como LIDAR 3D o cámaras de profundidad (RealSense, Kinect). Es literalmente una "foto" hecha de miles de puntos flotando en el espacio.

### LaserScan

Es la versión 2D y más simple: un LIDAR 2D (como el RPLidar A1) gira y mide distancias en un plano horizontal. En vez de puntos (x,y,z), el mensaje sensor_msgs/msg/LaserScan guarda un array de distancias, una por cada ángulo del barrido, junto con:

- angle_min / angle_max: rango angular del barrido

<!-- -->

- angle_increment: separación angular entre mediciones

- ranges\[\]: array de distancias medidas

- range_min / range_max: límites válidos del sensor

|               |                        |                                  |
|---------------|------------------------|----------------------------------|
|               | **LaserScan**          | **PointCloud2**                  |
| Dimensión     | 2D (un plano)          | 2D o 3D                          |
| Sensor típico | LIDAR 2D (RPLidar A1)  | LIDAR 3D, cámaras de profundidad |
| Uso típico    | Navegación 2D, SLAM 2D | Mapeo 3D, detección de objetos   |
| Peso/tamaño   | Liviano                | Más pesado                       |

### El puente entre ambos: pointcloud_to_laserscan

Es un nodo que convierte una nube de puntos en un LaserScan, "aplanando" la nube a un plano horizontal (input: /cloud, output: /scan). Esto es necesario porque Nav2 y slam_toolbox están optimizados para trabajar con LaserScan, mucho más liviano que una nube 3D completa.

## 1.2 URDF: la descripción física del robot

URDF (Unified Robot Description Format) es un archivo XML que describe de qué está hecho el robot físicamente:

- Links: partes rígidas (chasis, rueda, sensor)

- Joints: articulaciones que conectan links (fija, rotacional, prismática)

Cada link incluye geometría visual (para RViz2), geometría de colisión (para física) y propiedades inerciales. Lo más importante: el URDF le dice al sistema dónde está montado cada sensor respecto al robot. El nodo robot_state_publisher lee el URDF y publica automáticamente los frames de TF correspondientes.

## 1.3 TF: el árbol de transformaciones

TF mantiene la relación espacial entre todos los frames del robot en todo momento. La cadena típica es:

```
map → odom → base_link → laser_frame
```

- map: el mapa global fijo

- odom: posición estimada por odometría — acumula error (drift) con el tiempo

- base_link: el centro del robot

- laser_frame: dónde está montado el sensor (viene del URDF)

Nav2 necesita esta cadena completa para responder "el láser detectó un punto a 2 m — ¿dónde cae eso en el mapa global?".

## 1.4 SLAM y slam_toolbox

### ¿Qué es SLAM?

SLAM (Simultaneous Localization and Mapping) resuelve un problema de huevo-gallina: el robot no tiene mapa todavía, y tampoco sabe dónde está — pero necesita ambas cosas para navegar. SLAM construye el mapa y localiza al robot al mismo tiempo, comparando lo que el LIDAR ve ahora contra lo que ya vio antes.

### slam_toolbox

Es el paquete de ROS 2 más usado para SLAM 2D. Está basado en optimización de grafos de pose, y soporta loop closure: si el robot vuelve a un lugar ya visitado, corrige retroactivamente todo el mapa.

|                      |                                                              |
|----------------------|--------------------------------------------------------------|
| **Modo**             | **Qué hace**                                                 |
| Mapping (async/sync) | Construye el mapa desde cero mientras el robot se mueve      |
| Localization         | Usa un mapa ya guardado y solo estima la posición del robot  |
| Lifelong             | Sigue actualizando el mapa incluso después del mapeo inicial |

## 1.5 Nav2: el stack de navegación

Nav2 calcula cómo se mueve el robot de un punto A a un punto B evitando obstáculos. Sus componentes principales:

- BT Navigator: orquesta la lógica de alto nivel con un árbol de comportamiento

- Costmaps (global y local): mapas de "costo" que representan qué tan ocupada está cada celda

- Planner Server: calcula la ruta global (A\*, Dijkstra, NavFn)

- Controller Server: calcula la velocidad en tiempo real (/cmd_vel) para seguir la ruta

- Behavior/Recovery Server: recupera al robot si se atasca

- AMCL: localización dentro de un mapa conocido

- Waypoint Follower: navega a una lista de poses en secuencia

## 1.6 Relocalización y AMCL

Cuando el robot arranca, su odometría empieza en (0,0), pero eso no significa que esté en el origen del mapa. AMCL (Adaptive Monte Carlo Localization) estima la posición real del robot dentro de un mapa conocido usando un filtro de partículas.

Relocalización es el proceso de decirle a AMCL "estás acá" para que sus partículas arranquen cerca de la posición real. Esto se hace publicando en el tópico /initialpose, típicamente con la herramienta "2D Pose Estimate" de RViz2: clic + arrastre sobre el mapa, en la posición y orientación del robot.

Una vez publicada la pose inicial, mover el robot un poco ayuda a que el filtro de partículas converja — es decir, que se concentren en la posición real, reduciendo la incertidumbre (covarianza).

## 1.7 Simuladores: Gazebo vs Webots

|                       |                            |                             |
|-----------------------|----------------------------|-----------------------------|
|                       | **Gazebo**                 | **Webots**                  |
| Integración con ROS 2 | Nativa, estándar de facto  | Requiere puente webots_ros2 |
| Comunidad ROS         | Enorme                     | Más chica                   |
| Curva de aprendizaje  | Más compleja de configurar | Interfaz más amigable       |
| Rendimiento           | Puede ser pesado           | Generalmente más liviano    |

Para aprender ROS 2 desde cero siguiendo tutoriales estándar, Gazebo es la opción con más soporte y documentación disponible.

## 1.8 Hardware de cómputo embebido: Jetson Orin NX vs Jetson Thor

### Jetson Orin NX 16GB

Placa de cómputo embebido de Nvidia para IA en el "edge": GPU con núcleos CUDA + Tensor Cores, 16 GB de memoria compartida, corre Ubuntu con JetPack y tiene soporte completo de ROS 2. Ideal para percepción, Nav2 e inferencia liviana. Consumo bajo (10-25 W).

### Jetson Thor (VLA / Nvidia GR00T)

Generación mucho más potente, pensada para robots humanoides y modelos VLA (Vision-Language-Action): modelos que ingieren video de cámaras junto con comandos en lenguaje natural y predicen directamente comandos de motor. GR00T es el modelo VLA de Nvidia para esto.

|            |                        |                                        |
|------------|------------------------|----------------------------------------|
|            | **Orin NX 16GB**       | **Jetson Thor (T5000)**                |
| Uso típico | Robots móviles, drones | Humanoides, manipulación compleja, VLA |
| Memoria    | 16 GB                  | 128 GB LPDDR5X                         |
| Potencia   | ~10-25 W               | ~40-130 W                              |
| Ideal para | Percepción, Nav2       | Modelos VLA grandes, LLMs a bordo      |

## 1.9 Herramientas de conectividad y trabajo remoto

### Tailscale

VPN mesh basada en WireGuard: conecta todos los dispositivos (PC, laptop, Jetson, robot) como si estuvieran en la misma red local, sin abrir puertos ni configurar NAT. Cada dispositivo tiene una IP fija dentro de la "tailnet".

### tmux

Multiplexor de terminal: permite tener múltiples paneles dentro de una sola sesión SSH, y esa sesión sigue viva aunque se corte la conexión. Esencial para trabajar con nodos de ROS 2 corriendo sin pantalla propia (headless).

- tmux new -s nombre — crear sesión

- Ctrl+b d — desconectarse sin matar la sesión

- tmux attach -t nombre — reconectar

### Termius

Cliente SSH gráfico multiplataforma: guarda las conexiones SSH organizadas, sincroniza entre dispositivos, y tiene SFTP integrado.

Cómo se combinan: Tailscale da la red para llegar al dispositivo, Termius es la app para conectarse por SSH, y tmux mantiene las sesiones de trabajo vivas dentro de esa conexión.

# Parte 2 — Tutorial práctico paso a paso

Todo lo que sigue documenta el trabajo realizado en simulación, usando Ubuntu 22.04 + ROS 2 Humble + Gazebo 11 + TurtleBot3 (modelo burger).

## 2.1 Verificación del entorno base

```bash
printenv ROS_DISTRO      # debe devolver: humble
colcon version-check
```

## 2.2 Instalación de Gazebo y TurtleBot3

```bash
sudo apt update
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-turtlebot3 \
  ros-humble-turtlebot3-msgs ros-humble-turtlebot3-simulations -y
```

Se configuró el modelo de robot a usar (burger) como variable de entorno persistente:

```bash
echo "export TURTLEBOT3_MODEL=burger" >> ~/.bashrc
source ~/.bashrc
```

## 2.3 Primera simulación en Gazebo

```bash
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

Este comando abre Gazebo con un mundo de prueba (paredes hexagonales verdes, cilindros blancos como obstáculos) y el robot burger ya spawneado en el centro.

![](media/image1.png)

*Gazebo levantado por primera vez: mundo hexagonal y línea del rayo del LIDAR detectando una pared.*

### Detección y resolución de interferencia con otro proyecto

Al correr ros2 topic list, aparecieron tópicos inesperados de un proyecto anterior (bridge de Formant) mezclados con la sesión de Gazebo. Se resolvió pausando los servicios systemd correspondientes:

```bash
sudo systemctl stop formant-agent.service
sudo systemctl stop formant-sidecar.service
```

## 2.4 Mover el robot con teleop

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

Con las teclas W/A/S/D/X se controla el robot manualmente.

![](media/image2.png)

*El robot moviéndose: el abanico azul del LIDAR simulado detecta las paredes alrededor.*

## 2.5 Mapeo con slam_toolbox

```bash
sudo apt install ros-humble-slam-toolbox -y
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
```

Para visualizar el mapa mientras se construye, se abrió RViz2 y se agregaron manualmente los displays de Map y LaserScan:

```bash
rviz2 -d $(ros2 pkg prefix slam_toolbox)/share/slam_toolbox/rviz/slam_toolbox_default.rviz
```

![](media/image3.png)

*Primeros puntos del mapa apareciendo en RViz2: zona gris = espacio libre, bloques negros = obstáculos, puntos rojos = LaserScan en vivo.*

Recorriendo el mundo completo con el teleop, el mapa se fue completando:

![](media/image4.png)

*Mapa final: contorno hexagonal cerrado y los 9 obstáculos internos bien delimitados.*

## 2.6 Guardar el mapa

```bash
mkdir -p ~/ros2_maps
ros2 run nav2_map_server map_saver_cli -f ~/ros2_maps/turtlebot3_world
```

Esto genera turtlebot3_world.pgm (imagen del mapa) y turtlebot3_world.yaml (metadatos: resolución, origen, umbrales de ocupación).

Nota importante: el mapa debe guardarse en una carpeta propia (por ejemplo ~/ros2_maps/), no dentro del repositorio de otro proyecto — se detectó que un repo llamado "babar" era en realidad un proyecto de Node.js/TypeScript sin relación con ROS 2.

## 2.7 Cargar el mapa con Nav2

```bash
ros2 launch nav2_bringup bringup_launch.py use_sim_time:=true \
  map:=/home/trabajo/ros2_maps/turtlebot3_world.yaml
```

Al arrancar, es normal ver mensajes en bucle indicando que AMCL todavía no tiene pose inicial. Esto se resuelve en el paso siguiente — no hay que interrumpir el proceso mientras aparecen estos mensajes:

```bash
[amcl]: AMCL cannot publish a pose or update the transform. Please set the initial pose...
[global_costmap]: Timed out waiting for transform from base_link to map...
```

## 2.8 Relocalización con 2D Pose Estimate

```bash
rviz2 -d $(ros2 pkg prefix nav2_bringup)/share/nav2_bringup/rviz/nav2_default_view.rviz
```

![](media/image5.png)

*Antes de publicar /initialpose: el LaserScan (magenta) no coincide con las paredes — "efecto fantasma" típico.*

Con la herramienta "2D Pose Estimate" se hizo clic + arrastre en la posición aproximada del robot, y luego se movió un poco con el teleop para que AMCL terminara de converger:

![](media/image6.png)

*AMCL convergido en RViz2: el LaserScan coincide con las paredes y la nube de partículas está compacta.*

![](media/image7.png)

*Vista de Gazebo correspondiente al mismo instante.*

Se puede verificar numéricamente la confianza de AMCL leyendo su covarianza:

```bash
ros2 topic echo /amcl_pose --once
```

Valores bajos de varianza en x, y, yaw indican una localización confiable.

## 2.9 Navegación a un solo destino (Nav2 Goal)

Con la localización confirmada, se probó el primer objetivo de navegación autónoma con la herramienta "Nav2 Goal" (clic + arrastre para posición y orientación del destino):

![](media/image8.png)

*Feedback: reached — el robot navegó solo hasta el punto marcado, sin intervención del teleop.*

## 2.10 Navegación por waypoints

Para visitar varios puntos en secuencia, se activó el modo "Waypoint / Nav Through Poses Mode", se marcaron 3 puntos con "Nav2 Goal" (que en este modo se acumulan en una cola), y se disparó la secuencia con "Start Waypoint Following":

![](media/image9.png)

*Tres waypoints acumulados antes de iniciar la secuencia; el robot los recorrió de forma completamente autónoma.*

### Pipeline completo logrado

```bash
URDF (geometría del robot)
   -> TF (árbol de transformaciones)
   -> LIDAR simulado (/scan)
   -> slam_toolbox (mapeo con SLAM)
   -> mapa guardado (.pgm + .yaml)
   -> Nav2: map_server + AMCL (localización)
   -> relocalización (/initialpose)
   -> Planner + Controller (ruta y velocidad)
   -> Waypoint Follower (navegación a múltiples puntos)
```

# Parte 3 — Comandos útiles del día a día

## Entorno y verificación

```bash
printenv ROS_DISTRO                    # confirmar distro de ROS 2 activa
ros2 node list                         # nodos activos
ros2 topic list                        # tópicos activos
ros2 topic echo /nombre_topico --once  # ver un mensaje puntual
ros2 topic echo /amcl_pose --once      # pose y covarianza de AMCL
systemctl list-units | grep -i formant # servicios que puedan interferir
```

## Gazebo + TurtleBot3

```bash
gazebo --version                                        # verificar instalación
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py # lanzar mundo + robot
ros2 run turtlebot3_teleop teleop_keyboard               # mover el robot
```

## SLAM (mapeo)

```bash
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
ros2 run nav2_map_server map_saver_cli -f ~/ros2_maps/nombre_mapa
```

## Nav2 (navegación)

```bash
ros2 launch nav2_bringup bringup_launch.py use_sim_time:=true \
  map:=/ruta/al/mapa.yaml
```

## RViz2

```bash
# Vista genérica de SLAM
rviz2 -d $(ros2 pkg prefix slam_toolbox)/share/slam_toolbox/rviz/slam_toolbox_default.rviz

# Vista específica de Nav2
rviz2 -d $(ros2 pkg prefix nav2_bringup)/share/nav2_bringup/rviz/nav2_default_view.rviz
```

Para guardar una configuración propia y no repetir los displays cada vez: File → Save Config As... → guardar en ~/ros2_maps/mi_config.rviz

## Publicar una pose inicial por terminal (alternativa al clic en RViz2)

```bash
ros2 topic pub -1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {frame_id: 'map'},
  pose: {
    pose: {
      position: {x: 0.0, y: 0.0, z: 0.0},
      orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
    }
  }
}"
```

Usar siempre punto decimal (0.5), nunca coma (0,5) — YAML interpreta la coma como separador de campo y falla con un error de parseo.

## Gestión de paquetes

```bash
sudo apt update
sudo apt install ros-humble-<paquete> -y
colcon version-check          # revisar versiones de colcon instaladas
```

## tmux (sesiones persistentes por SSH)

```bash
tmux new -s trabajo      # crear sesión nueva
Ctrl+b d                 # desconectar sin matar la sesión
tmux attach -t trabajo   # reconectar a una sesión existente
tmux ls                  # listar sesiones activas
```

# Parte 4 — Problemas comunes y cómo resolverlos

|                                                              |                                                              |                                                                    |
|--------------------------------------------------------------|--------------------------------------------------------------|--------------------------------------------------------------------|
| **Síntoma**                                                  | **Causa probable**                                           | **Solución**                                                       |
| Tópicos raros (/turtle1/cmd_vel, /formant/command) mezclados | Servicios systemd de otro proyecto en el mismo dominio ROS 2 | sudo systemctl stop formant-agent.service formant-sidecar.service  |
| "Please set the initial pose" en bucle                       | Localización no inicializada (normal al arrancar Nav2)       | Publicar /initialpose con "2D Pose Estimate" en RViz2              |
| "Timed out waiting for transform base_link to map"           | Consecuencia de no tener /initialpose publicado              | Se resuelve solo al dar la pose inicial                            |
| RViz2 no muestra el mapa/scan                                | Faltan los displays Map y LaserScan                          | Add -\> By topic -\> /map -\> Map; repetir con /scan -\> LaserScan |
| Robot y LaserScan desfasados (efecto fantasma)               | Pose inicial imprecisa o recién publicada                    | Mover el robot con el teleop para que AMCL converja                |
| "Feedback: aborted" en waypoints                             | Waypoint en zona no explorada o muy cerca de un obstáculo    | Elegir puntos en zonas blancas amplias, lejos de postes y paredes  |
| Botón "Reset" deja todo en rojo/inactivo                     | Reset reinicia el lifecycle manager completo                 | Presionar "Startup" y volver a dar /initialpose                    |
| Error "Failed to populate field: getattr()"                  | Uso de coma decimal en vez de punto                          | Reemplazar comas por puntos en los valores numéricos               |
| El mapa guardado no aparece donde se esperaba                | Carpeta de destino mezclada con otro repo                    | Usar una carpeta dedicada como ~/ros2_maps/                        |

*Documento generado como resumen del trabajo de aprendizaje de ROS 2, SLAM y Nav2 en simulación con TurtleBot3 y Gazebo.*
