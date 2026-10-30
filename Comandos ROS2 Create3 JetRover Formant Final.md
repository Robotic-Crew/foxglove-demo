# Comandos del día a día

*ROS 2 · Create 3 (simulado y real) · JetRover · Foxglove · Formant*

![Ecosistema de trabajo](img/d_eco.png)

Guía de referencia: concepto, requisitos, comando y resultado esperado · Diego Pedrazas · Octubre 2026

## Índice

- [1. Cómo usar esta guía](#1-como-usar-esta-guia)
  - [1.1 Dónde se ejecuta cada comando](#11-donde-se-ejecuta-cada-comando)
  - [1.2 El ecosistema completo](#12-el-ecosistema-completo)
  - [1.3 Tabla de dominios y RMW](#13-tabla-de-dominios-y-rmw)
- [2. Conceptos de ROS 2 en una página](#2-conceptos-de-ros-2-en-una-pagina)
  - [2.1 Compatibilidad de QoS](#21-compatibilidad-de-qos)
- [3. ROS 2: comandos generales](#3-ros-2-comandos-generales)
  - [3.1 Entorno](#31-entorno)
  - [3.2 Descubrir qué hay corriendo](#32-descubrir-que-hay-corriendo)
  - [3.3 Leer y publicar datos](#33-leer-y-publicar-datos)
  - [3.4 Servicios, acciones y parámetros](#34-servicios-acciones-y-parametros)
  - [3.5 TF](#35-tf)
  - [3.6 Compilar, lanzar, grabar](#36-compilar-lanzar-grabar)
  - [3.7 Cuando el CLI de ros2 falla](#37-cuando-el-cli-de-ros2-falla)
- [4. Docker](#4-docker)
- [5. Create 3 simulado (contenedor create3, dominio 78)](#5-create-3-simulado-contenedor-create3-dominio-78)
- [6. Create 3 real (/pada_1 en rcrew-2)](#6-create-3-real-pada_1-en-rcrew-2)
  - [6.1 Chequeos previos (en orden)](#61-chequeos-previos-en-orden)
  - [6.2 Levantar el robot](#62-levantar-el-robot)
  - [6.3 Laptop: servicios, verificación y RViz](#63-laptop-servicios-verificacion-y-rviz)
  - [6.4 Operar el robot](#64-operar-el-robot)
- [7. JetRover simulado (contenedor jetrover, dominio 77)](#7-jetrover-simulado-contenedor-jetrover-dominio-77)
- [8. Dashboard Foxglove (Create 3 + JetRover)](#8-dashboard-foxglove-create-3--jetrover)
  - [8.1 Ajustes de Foxglove](#81-ajustes-de-foxglove)
- [9. Formant](#9-formant)
  - [9.1 Servicios del agente](#91-servicios-del-agente)
  - [9.2 Tópicos que consume Formant (robot real)](#92-topicos-que-consume-formant-robot-real)
  - [9.3 Configuración en la web (sin comandos)](#93-configuracion-en-la-web-sin-comandos)
- [10. Diagnóstico: el orden correcto](#10-diagnostico-el-orden-correcto)
- [11. Rutinas del día a día](#11-rutinas-del-dia-a-dia)
  - [11.1 Simulación Create 3 + JetRover + Foxglove](#111-simulacion-create-3--jetrover--foxglove)
  - [11.2 Robot real Create 3 + RViz + Formant](#112-robot-real-create-3--rviz--formant)
  - [11.3 Las reglas de oro](#113-las-reglas-de-oro)

---

# 1. Cómo usar esta guía

Chuleta de los comandos que se usan en el día a día con **ROS 2**, el **iRobot Create 3** (simulado y real), el **Hiwonder JetRover** (simulado) y **Formant**. Cada comando aparece como una ficha:

| Parte de la ficha | Qué contiene |
|---|---|
| Título | Qué se quiere lograr |
| Descripción | Qué hace el comando y por qué se usa |
| Requisitos | Qué tiene que estar corriendo o configurado antes |
| Bloque de código | El comando, con una etiqueta que dice DÓNDE ejecutarlo |
| Resultado esperado | Lo que deberías ver si todo está bien (recuadro verde) |
| Atención | Errores típicos o trampas (recuadro naranja) |

## 1.1 Dónde se ejecuta cada comando

| Etiqueta | Dónde | Cómo llegar |
|---|---|---|
| HOST | Tu laptop Ubuntu 22.04 (ROS 2 Humble) | Terminal normal: trabajo@…$ |
| CONTENEDOR create3 | Simulación del Create 3 (Jazzy + Harmonic) | docker exec -it create3 bash |
| CONTENEDOR jetrover | Simulación del JetRover (Humble + Fortress) | docker exec -it jetrover bash |
| ROBOT rcrew-2 | Raspberry Pi del Create 3 real | ssh rcrew@rcrew-2 |
| FORMANT (web) | app.formant.io / teleop.formant.io | Navegador |

## 1.2 El ecosistema completo

![Figura 1. Laptop, contenedores, robot real y Formant. Cada bloque vive en su propio dominio ROS.](img/d_eco.png)

*Figura 1. Laptop, contenedores, robot real y Formant. Cada bloque vive en su propio dominio ROS.*

## 1.3 Tabla de dominios y RMW

| Qué | ROS_DOMAIN_ID | RMW | Namespace |
|---|---|---|---|
| formant-agent local (laptop) | 0 | por defecto | /turtle1 (pruebas) |
| Robot real vía Zenoh (laptop) | 42 | por defecto (lado laptop) | /pada_1 + raíz (nav2) |
| Robot real en rcrew-2 y formant-agent del robot | 0 | rmw_cyclonedds_cpp | /pada_1 + raíz |
| Simulación JetRover | 77 | por defecto | raíz |
| Simulación Create 3 | 78 | por defecto | raíz (create3/ vía relay) |

> **Atención.** La causa número uno de "no llega nada" es estar en el **dominio equivocado**. Antes de diagnosticar, ejecuta siempre `echo $ROS_DOMAIN_ID`.
>
> La segunda causa (robot real): el **RMW equivocado**. El Create 3 habla CycloneDDS; con FastDDS los tópicos aparecen en la lista pero no llega ningún dato.

# 2. Conceptos de ROS 2 en una página

| Concepto | Qué es | Analogía / ejemplo |
|---|---|---|
| Nodo | Un proceso que hace una tarea | robot_state_publisher, move_group, foxglove_bridge |
| Tópico | Canal de mensajes publicador → suscriptores, continuo | /cmd_vel, /odom, /scan |
| Servicio | Pregunta–respuesta, una vez | /controller_manager/list_controllers |
| Acción | Tarea larga con progreso (feedback) y cancelable | /pada_1/dock, follow_joint_trajectory |
| Parámetro | Configuración de un nodo | use_sim_time, port |
| Interfaz | El "tipo" de un mensaje, servicio o acción | geometry_msgs/msg/Twist |
| TF | Árbol de sistemas de coordenadas | map → odom → base_link |
| QoS | Políticas de entrega: Reliability y Durability | TRANSIENT_LOCAL para /map y /tf_static |
| ROS_DOMAIN_ID | "Canal de radio" DDS: solo se ven nodos del mismo número | 77, 78, 42, 0 |
| RMW | Implementación de DDS que usa ROS | rmw_fastrtps_cpp (Humble) / rmw_cyclonedds_cpp |
| Workspace | Carpeta con src/ build/ install/ | /root/ws |
| source | Cargar ROS o un workspace en la terminal actual | source /opt/ros/humble/setup.bash |
| Daemon ros2 | Proceso en segundo plano que cachea el grafo para el CLI | Se traba seguido: ros2 daemon stop |
| Lifecycle node | Nodo con estados (unconfigured → inactive → active) | map_server, amcl (nav2) |
| Controlador | Plugin de ros2_control que mueve articulaciones | diffdrive_controller, arm_controller |

## 2.1 Compatibilidad de QoS

| Publicador | Suscriptor | ¿Conectan? |
|---|---|---|
| RELIABLE | BEST_EFFORT | Sí |
| BEST_EFFORT | RELIABLE | No (sin avisar) |
| TRANSIENT_LOCAL | VOLATILE | Sí, pero no recibe lo ya publicado |
| VOLATILE | TRANSIENT_LOCAL | No (sin avisar) |

# 3. ROS 2: comandos generales

## 3.1 Entorno

### Cargar ROS y el workspace

`source` ejecuta un script que define variables (PATH, AMENT_PREFIX_PATH…) en la terminal actual. Hay que hacerlo en **cada terminal nueva**.

**Requisitos:** ROS instalado · workspace compilado (para el segundo)

**HOST / CONTENEDOR**

```bash
source /opt/ros/humble/setup.bash        # Jazzy: /opt/ros/jazzy/setup.bash
source /root/ws/install/setup.bash        # workspace (contenedores)
```

> **Resultado esperado.** No imprime nada. Si el archivo no existe: "No existe el archivo o el directorio" (estás en el lugar equivocado o falta compilar).

### Ver la configuración ROS de la terminal

Muestra distro, dominio, RMW y si está limitado a localhost.

**Requisitos:** source hecho

**HOST / CONTENEDOR**

```bash
echo $ROS_DOMAIN_ID
printenv | grep -E "ROS_|RMW|CYCLONE"
```

> **Resultado esperado.** `ROS_DISTRO=humble` (o jazzy), `ROS_DOMAIN_ID=77` en jetrover, `78` en create3. Si `ROS_DOMAIN_ID` sale vacío, el dominio es 0.

### Cambiar de dominio en esta terminal

Afecta solo a los comandos que lances **después** en esta terminal.

**HOST / CONTENEDOR**

```bash
export ROS_DOMAIN_ID=78
```

> **Resultado esperado.** Sin salida. Verifica con `echo $ROS_DOMAIN_ID`.

### Chequeo general

Revisa red, RMW, paquetes y reporta problemas del entorno.

**Requisitos:** source hecho

**HOST / CONTENEDOR**

```bash
ros2 doctor --report | head -40
```

> **Resultado esperado.** Secciones NETWORK, PLATFORM, RMW MIDDLEWARE con los valores actuales.

## 3.2 Descubrir qué hay corriendo

### Listar nodos

Todos los nodos visibles en el dominio actual.

**Requisitos:** source · dominio correcto

**CONTENEDOR**

```bash
ros2 node list
ros2 node info /robot_state_publisher      # tópicos, servicios y acciones de un nodo
```

> **Resultado esperado.** Una lista como `/move_group`, `/robot_state_publisher`, `/foxglove_bridge`…

### Listar tópicos con su tipo

`-t` agrega el tipo de mensaje. Combinado con grep filtra rápido.

**Requisitos:** source · dominio correcto

**CONTENEDOR**

```bash
ros2 topic list -t
ros2 topic list | grep create3
ros2 topic list --no-daemon                # si el daemon está trabado
```

> **Resultado esperado.** `/cmd_vel [geometry_msgs/msg/Twist]`, `/odom [nav_msgs/msg/Odometry]`… Vacío = nadie publica en ese dominio.

### Quién publica y con qué QoS

`-v` lista publicadores y suscriptores con Reliability y Durability. Clave para diagnosticar QoS y fuentes duplicadas.

**Requisitos:** source

**CONTENEDOR**

```bash
ros2 topic info /tf_static -v | grep -E "Node name|Endpoint type|Reliability|Durability"
ros2 topic info /joint_states               # solo Publisher count / Subscription count
```

> **Resultado esperado.** Bloques por endpoint: `Node name: robot_state_publisher`, `Endpoint type: PUBLISHER`, `Durability: TRANSIENT_LOCAL`.

### Ver la estructura de un tipo

Muestra los campos de un mensaje, servicio o acción, para saber qué escribir en un pub o call.

**Requisitos:** source

**HOST / CONTENEDOR**

```bash
ros2 interface show geometry_msgs/msg/Twist
ros2 interface show irobot_create_msgs/action/Dock
```

> **Resultado esperado.** `Vector3 linear` / `float64 x`… y para acciones las tres partes separadas por `---` (goal, result, feedback).

## 3.3 Leer y publicar datos

### Ver un mensaje

`--once` muestra uno y termina. `timeout` evita quedarse esperando si nadie publica.

**Requisitos:** source · tópico con publicador

**CONTENEDOR**

```bash
timeout 10 ros2 topic echo /odom --once
timeout 10 ros2 topic echo /odom --once --field pose.pose.position
ros2 topic echo /robot_description --once --full-length | head -5
```

> **Resultado esperado.** Un mensaje en YAML: `header: stamp: …`, `pose: pose: position: x: …`. Si no sale nada en 10 s, nadie publica (o dominio/RMW equivocado).

> **Atención.** Sin `--full-length` los strings largos (como el URDF) se cortan con "…", y un `grep` posterior no encuentra nada.

### Medir frecuencia y ancho de banda

`hz` = mensajes por segundo; `bw` = bytes por segundo. Ctrl+C para salir.

**Requisitos:** source

**CONTENEDOR**

```bash
ros2 topic hz /scan
ros2 topic bw /depth_cam/image
timeout 15 ros2 topic hz /odom              # se corta solo
```

> **Resultado esperado.** `average rate: 10.000` con min/max/std dev. "does not appear to be published yet" = no hay publicador.

> **Atención.** No pases `hz` por un pipe (`| tail`): Python bufferiza la salida y, al cortarse con timeout, no imprime nada.

### Publicar un mensaje

`-r 10` publica 10 veces por segundo hasta Ctrl+C; `--once` publica uno. El texto entre comillas es YAML con los campos del tipo.

**Requisitos:** source · conocer el tipo (interface show)

**CONTENEDOR**

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}"            # parar
timeout 3 ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{angular: {z: 0.5}}"
```

> **Resultado esperado.** `publisher: beginning loop` y `publishing #1: …`. Si aparece "Waiting for at least 1 matching subscription(s)…", nadie escucha ese tópico con ese tipo.

## 3.4 Servicios, acciones y parámetros

### Servicios

Listar y llamar servicios (pregunta–respuesta).

**Requisitos:** source

**CONTENEDOR**

```bash
ros2 service list | grep controller_manager
ros2 service call /controller_manager/list_controllers controller_manager_msgs/srv/ListControllers
```

> **Resultado esperado.** `requester: making request…` y luego `response: …` con los datos.

### Acciones

Tareas largas con progreso. `--feedback` muestra el avance.

**Requisitos:** source · servidor de acción activo

**HOST (dominio 42) / ROBOT**

```bash
ros2 action list
ros2 action info /pada_1/dock
ros2 action send_goal /pada_1/undock irobot_create_msgs/action/Undock "{}" --feedback
```

> **Resultado esperado.** `Goal accepted with ID: …`, líneas de `Feedback:` y al final `Goal finished with status: SUCCEEDED`.

> **Atención.** Cortar el cliente con Ctrl+C **no cancela** el goal en el robot.

### Parámetros

Leer o cambiar la configuración de un nodo en vivo.

**Requisitos:** source · nodo corriendo

**CONTENEDOR**

```bash
ros2 param list /foxglove_bridge
ros2 param get /foxglove_bridge use_sim_time
ros2 param set /move_group use_sim_time true
```

> **Resultado esperado.** `Boolean value is: False` / `Set parameter successful`.

## 3.5 TF

### Transformación entre dos frames

Imprime cada segundo dónde está un frame respecto de otro. La prueba definitiva de que la cadena TF está completa.

**Requisitos:** source · TF publicándose

**CONTENEDOR jetrover**

```bash
ros2 run tf2_ros tf2_echo odom base_footprint
ros2 run tf2_ros tf2_echo world create3/base_link
```

> **Resultado esperado.** `- Translation: [x, y, z]` y `- Rotation: in Quaternion …` repetidos. "Invalid frame ID" = falta algún eslabón de la cadena.

### Dibujar el árbol TF

Genera un PDF con todos los frames y quién los publica.

**Requisitos:** source · TF publicándose

**HOST / CONTENEDOR**

```bash
ros2 run tf2_tools view_frames            # crea frames_<fecha>.pdf en la carpeta actual
```

> **Resultado esperado.** "Generating graph in frames.pdf file…". Un árbol con varias raíces sueltas indica un TF partido.

### Publicar un TF estático

Une dos frames con una transformación fija (aquí: el frame del LiDAR que Gazebo nombra distinto).

**Requisitos:** source

**CONTENEDOR jetrover**

```bash
ros2 run tf2_ros static_transform_publisher \
  --frame-id lidar_frame --child-frame-id jetrover/base_footprint/lidar
```

> **Resultado esperado.** "Spinning until stopped - publishing transform …". Se deja corriendo.

## 3.6 Compilar, lanzar, grabar

### Compilar un workspace

`--symlink-install` enlaza Python/launch/YAML: editarlos no requiere recompilar. `--packages-select` compila solo un paquete.

**Requisitos:** estar en la raíz del workspace · source de ROS

**CONTENEDOR jetrover**

```bash
cd /root/ws
colcon build --symlink-install --packages-up-to jetrover_gazebo jetrover_moveit_config
colcon build --symlink-install --packages-select mi_paquete
source install/setup.bash
```

> **Resultado esperado.** "Summary: N packages finished [Xs]". Con errores: "Failed &lt;&lt;&lt; paquete" y el detalle arriba.

### Instalar dependencias de un workspace

Lee los package.xml e instala con apt lo que falte.

**Requisitos:** rosdep update hecho una vez

**CONTENEDOR**

```bash
rosdep install --from-paths src -i -y
```

> **Resultado esperado.** "#All required rosdeps installed successfully".

### Ver los argumentos de un launch

Muestra qué parámetros acepta sin lanzar nada.

**Requisitos:** source

**CONTENEDOR create3**

```bash
ros2 launch irobot_create_gz_bringup create3_gz.launch.py --show-args
```

> **Resultado esperado.** Lista de `'namespace'`, `'use_rviz'`, `'world'`… con su descripción y valor por defecto.

### Grabar y reproducir datos (rosbag)

Graba tópicos a disco para analizarlos o reproducirlos después (también se abren en Foxglove).

**Requisitos:** source

**CONTENEDOR**

```bash
ros2 bag record -o prueba /odom /scan /tf /tf_static
ros2 bag info prueba
ros2 bag play prueba
```

> **Resultado esperado.** "Recording..." / tabla con duración, mensajes y tópicos / reproducción en tiempo real.

### Herramientas gráficas

Grafo de nodos y tópicos, y visualizador 3D.

**Requisitos:** source · DISPLAY disponible

**HOST / CONTENEDOR**

```bash
ros2 run rqt_graph rqt_graph
rviz2 -d archivo.rviz
```

> **Resultado esperado.** Se abre la ventana.

## 3.7 Cuando el CLI de ros2 falla

### Reiniciar el daemon

El daemon cachea el grafo; cuando se traba devuelve listas vacías o tracebacks de xmlrpc ("!rclpy.ok()").

**HOST / CONTENEDOR**

```bash
ros2 daemon stop
ros2 daemon start
ros2 topic list --no-daemon                # alternativa: no usar el daemon
```

> **Resultado esperado.** "The daemon has been stopped" / "The daemon has been started".

| Síntoma del CLI | Qué hacer |
|---|---|
| xmlrpc.client.Fault … !rclpy.ok() | ros2 daemon stop && ros2 daemon start |
| failed to initialize wait set | PC sobrecargada: repetir, usar --no-daemon |
| echo --once no termina | Nadie publica: anteponer timeout 10 |
| hz \| tail vacío | No usar pipes con hz |
| pgrep no encuentra un proceso | Linux corta los nombres a 15 caracteres: pgrep -x component_conta |

# 4. Docker

### Ver contenedores

`-a` incluye los detenidos. El formato tabla muestra nombre y estado.

**Requisitos:** Docker Engine

**HOST**

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}"
docker images | grep -E "create3|jetrover"
```

> **Resultado esperado.** `jetrover  Up 2 hours`, `create3  Up 2 hours`. Si no aparecen, no están corriendo.

### Crear un contenedor (run.sh)

Ejecuta `docker run` con todas las opciones (red, GPU, X11, dominio, volúmenes). Esta terminal queda como **principal** del contenedor.

**Requisitos:** imagen construida · estar en la carpeta del proyecto

**HOST**

```bash
cd ~/docker/create3_harmonic && ./run.sh
cd ~/docker/jetrover && ./run.sh
```

> **Resultado esperado.** El prompt cambia a `root@…#`.

> **Atención.** En esta terminal **no escribas `exit`**: con `--rm` el contenedor se borra.

### Abrir otra terminal en un contenedor

Entra a un contenedor que ya corre. Aquí `exit` solo cierra esta terminal.

**Requisitos:** contenedor corriendo

**HOST**

```bash
docker exec -it jetrover bash
docker exec -it create3 bash
```

> **Resultado esperado.** Prompt `root@…#`. "No such container" = no está corriendo.

### Copiar archivos desde un contenedor

`-L` sigue los enlaces simbólicos del primer nivel.

**Requisitos:** contenedor corriendo

**HOST**

```bash
docker cp -L create3:/root/ws/install/irobot_create_description ~/docker/jetrover/ws/extra_pkgs/
```

> **Resultado esperado.** "Successfully copied 56.8kB to …".

> **Atención.** Con `--symlink-install` los archivos internos siguen siendo enlaces a src/ y build/: copia también esas carpetas reales.

### Detener, construir, limpiar

Comandos de mantenimiento.

**HOST**

```bash
docker stop jetrover create3                 # detiene (y borra, por --rm)
docker build -t jetrover-humble:v2 .           # reconstruir imagen
docker rmi jetrover-humble:v1                  # borrar imagen vieja
docker context use default                     # si dice "desktop-linux"
```

> **Resultado esperado.** Devuelve el nombre del contenedor detenido / "Successfully tagged …".

# 5. Create 3 simulado (contenedor create3, dominio 78)

**Requisitos comunes:** contenedor `create3` corriendo, `source /opt/ros/jazzy/setup.bash && source /root/ws/install/setup.bash`, `ROS_DOMAIN_ID=78`.

### Lanzar la simulación

Abre Gazebo Harmonic con el Create 3 en el dock. `use_rviz:=false` ahorra carga (Foxglove lo reemplaza).

**Requisitos:** requisitos comunes

**CONTENEDOR create3**

```bash
ros2 launch irobot_create_gz_bringup create3_gz.launch.py use_rviz:=false
```

> **Resultado esperado.** Ventana de Gazebo con el mundo "depot", el robot y los paneles Create3 HMI y Teleop.

### Activar las ruedas

Carga y activa el `diffdrive_controller`. Con la PC cargada (RTF bajo) hace falta más tiempo que los 5 s por defecto.

**Requisitos:** simulación corriendo

**CONTENEDOR create3**

```bash
ros2 run controller_manager spawner diffdrive_controller -c /controller_manager --switch-timeout 60
```

> **Resultado esperado.** "Configured and activated diffdrive_controller". "Controller already loaded" + "activated" también es correcto.

> **Atención.** "Failed to activate controller" = se agotó el tiempo: repetir con `--switch-timeout 60`.

### Ver el estado de los controladores

En esta imagen no está `ros2 control`; se llama al servicio directamente.

**Requisitos:** simulación corriendo

**CONTENEDOR create3**

```bash
ros2 service call /controller_manager/list_controllers controller_manager_msgs/srv/ListControllers | grep -oE "name='[a-z_]+'|state='[a-z]+'"
```

> **Resultado esperado.** `name='diffdrive_controller'` `state='active'` y `name='joint_state_broadcaster'` `state='active'`.

### Mover el Create 3

En Jazzy el `/cmd_vel` del Create 3 es **TwistStamped** (con Twist no se mueve).

**Requisitos:** ruedas activas

**CONTENEDOR create3**

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {linear: {x: 0.1}}}"
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/TwistStamped "{twist: {angular: {z: 0.5}}}"
```

> **Resultado esperado.** El robot avanza / gira en Gazebo. Ctrl+C para detener.

### Odometría y batería

Confirma que el controlador publica.

**Requisitos:** ruedas activas

**CONTENEDOR create3**

```bash
timeout 15 ros2 topic hz /odom
timeout 10 ros2 topic echo /battery_state --once --field percentage
```

> **Resultado esperado.** `average rate:` &gt; 0 (con RTF bajo puede ser 1–3 Hz) / un número entre 0 y 1.

### Dock y undock en simulación

Acciones del Create 3 (el robot debe estar frente al dock).

**Requisitos:** ruedas activas

**CONTENEDOR create3**

```bash
ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}" --feedback
ros2 action send_goal /dock irobot_create_msgs/action/Dock "{}" --feedback
```

> **Resultado esperado.** "Goal finished with status: SUCCEEDED". El robot retrocede y gira 180° (undock) o se acopla (dock).

# 6. Create 3 real (/pada_1 en rcrew-2)

El robot real corre ROS 2 Humble con **CycloneDDS** y namespace `/pada_1`. La Raspberry **rcrew-2** lleva nav2, el lidar y el puente Zenoh; la laptop recibe los datos por Tailscale en el **dominio 42**.

> **Atención.** Todo comando ROS ejecutado **en rcrew-2** necesita:
>
> `export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`
>
> `export CYCLONEDDS_URI=file:///home/rcrew/create3/cyclonedds.xml`
>
> Sin eso los tópicos aparecen en la lista pero no llega ningún dato.

## 6.1 Chequeos previos (en orden)

### Red y lidar

Si un paso falla, no tiene sentido seguir.

**Requisitos:** Tailscale activo en la laptop

**HOST**

```bash
ssh rcrew@rcrew-2 uptime                      # 1. ¿la laptop llega a rcrew-2?
ssh rcrew@rcrew-2 ping -c2 192.168.1.116      # 2. ¿rcrew-2 llega al Create 3?
ssh rcrew@rcrew-2 ls -la /dev/ttyUSB0         # 3. ¿lidar enchufado?
```

> **Resultado esperado.** 1: "up X days, load average…". 2: "2 received, 0% packet loss". 3: el dispositivo `/dev/ttyUSB0`.
>
> Si falla el 2, el Create 3 está dormido o fuera de la WiFi: hay que despertarlo físicamente o ponerlo en el dock. Si falla el 1, puede que el robot se haya quedado sin batería (la Pi se alimenta de él).

## 6.2 Levantar el robot

### Puente Zenoh

Conecta el DDS del robot con la laptop por un único puerto TCP (7447) a través de Tailscale.

**Requisitos:** chequeos OK

**HOST**

```bash
ssh rcrew@rcrew-2 'nohup setsid ~/.local/bin/start-zenoh-bridge.sh > /tmp/zenoh-bridge.log 2>&1 </dev/null &'
sleep 10
ssh rcrew@rcrew-2 'ss -ltn | grep 7447'
```

> **Resultado esperado.** Una línea `LISTEN … :7447`.

### nav2 y lidar

Lanza localización (AMCL), mapa y navegación. Tarda unos 60 s.

**Requisitos:** puente Zenoh

**HOST**

```bash
ssh rcrew@rcrew-2 'nohup setsid ~/run_nav_casa.sh > /tmp/nav2.log 2>&1 </dev/null &'
ssh rcrew@rcrew-2 'ps -eo comm | grep -c component_conta'    # contenedores nav2
ssh rcrew@rcrew-2 'grep -c "Loaded node" /tmp/nav2.log'      # nodos cargados
```

> **Resultado esperado.** `1` (un solo contenedor) y `9` (nodos cargados).

> **Atención.** Si hay **2** contenedores, ninguno carga: matar todos (`pkill -x component_conta`) y lanzar uno.

### ¿El lidar gira de verdad?

El nodo puede decir "health status: OK" con el motor parado. Si `rchar` crece, está leyendo datos.

**Requisitos:** nav2 lanzado

**HOST**

```bash
ssh rcrew@rcrew-2 'P=$(pgrep -x rplidar_node); grep rchar /proc/$P/io; sleep 5; grep rchar /proc/$P/io'
```

> **Resultado esperado.** El segundo `rchar` es decenas de KB mayor que el primero. Clavado en ~62 KB = solo handshake (problema de alimentación).

### Nodos de nav2 activos

Son lifecycle nodes: existir no alcanza, tienen que estar en `active`.

**Requisitos:** nav2 lanzado

**HOST**

```bash
ssh rcrew@rcrew-2 'source /opt/ros/humble/setup.bash
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
  export CYCLONEDDS_URI=file:///home/rcrew/create3/cyclonedds.xml
  for N in map_server amcl controller_server planner_server bt_navigator; do
    printf "%-19s " $N
    ros2 service call /$N/get_state lifecycle_msgs/srv/GetState | grep -oE "label=.[a-z_]+"
  done'
```

> **Resultado esperado.** Cinco líneas con `label='active`.

## 6.3 Laptop: servicios, verificación y RViz

### Servicios rcrew de la laptop

systemd de usuario: cliente Zenoh (dominio 42), relay de /tf_static, URDF del Create 3, mapa local y watchdog de batería.

**Requisitos:** puente Zenoh en el robot

**HOST**

```bash
systemctl --user status 'rcrew-*'
systemctl --user restart rcrew-zenoh-client rcrew-tf-relay rcrew-map-local
systemctl --user start rcrew-watchdog.timer
```

> **Resultado esperado.** Cada servicio en `active (running)`.

### ¿Llegan los datos del robot?

Prueba cada tópico importante con un mensaje.

**Requisitos:** servicios rcrew activos

**HOST**

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=42
ros2 daemon stop
for T in /map /scan /tf /pada_1/odom /local_costmap/costmap; do
  printf "%-28s " "$T"
  timeout 15 ros2 topic echo $T --once --no-daemon >/dev/null 2>&1 && echo OK || echo "SIN DATOS"
done
```

> **Resultado esperado.** Cinco líneas terminadas en `OK`.

> **Atención.** "Llegan datos" no es lo mismo que "los datos cambian": el Create 3 puede publicar odometría congelada. Si el robot se mueve y la pose no cambia, hace falta un power-cycle.

### Cadena TF del robot real

Confirma que AMCL localiza (`map → odom`) y que el resto del árbol está completo.

**Requisitos:** dominio 42 · nav2 activo

**HOST**

```bash
ros2 run tf2_ros tf2_echo map base_link --ros-args -r /tf_static:=/tf_static_tl
```

> **Resultado esperado.** Una traslación. "Invalid frame ID map" = AMCL no localiza (casi siempre falta /scan) o falta la pose inicial (2D Pose Estimate en RViz).

### Abrir RViz del robot

Abre RViz con la configuración de pada_1 (Fixed Frame = map).

**Requisitos:** dominio 42 · datos llegando

**HOST**

```bash
~/.local/bin/rviz-pada1.sh
# o solo RViz:
ROS_DOMAIN_ID=42 ros2 run rviz2 rviz2 -d ~/.rviz2/pada1.rviz --ros-args -r /tf_static:=/tf_static_tl
# si RViz quedó congelado (instancia vieja):
for P in $(pgrep -x rviz2); do kill $P; done
```

> **Resultado esperado.** Mapa, costmap local, lidar y el robot; arriba "Global Status: Ok".

## 6.4 Operar el robot

### Despertar el robot

El Create 3 entra en bajo consumo; publicar velocidad cero lo despierta.

**Requisitos:** dominio 42 (laptop) o RMW Cyclone (robot)

**HOST (dominio 42)**

```bash
ros2 topic pub --once /pada_1/cmd_vel geometry_msgs/msg/Twist "{}"
```

> **Resultado esperado.** "publishing #1" y la odometría vuelve a llegar.

### Desacoplar (undock)

Acoplado, el Create 3 **ignora** `cmd_vel` por diseño.

**Requisitos:** robot en el dock

**HOST (dominio 42)**

```bash
ros2 action send_goal /pada_1/undock irobot_create_msgs/action/Undock "{}" --feedback
```

> **Resultado esperado.** "Goal finished with status: SUCCEEDED". El robot retrocede y gira 180°.

### Volver al dock

La acción **no navega**: gira buscando la señal infrarroja (1–2 m, con línea de vista).

**Requisitos:** robot cerca y de frente al dock · dock enchufado

**HOST (dominio 42)**

```bash
ros2 topic echo /pada_1/dock_status --once                 # ver dock_visible
ros2 action send_goal /pada_1/dock irobot_create_msgs/action/Dock "{}" --feedback
```

> **Resultado esperado.** `dock_visible: true` y luego "SUCCEEDED".

> **Atención.** Si el dock no está cerca, gira, no lo encuentra y aborta gastando batería. Dale tiempo: cortar el cliente no cancela el goal.

### Batería

`percentage` va de 0 a 1. `current` **positivo = cargando** (más confiable que dock_status).

**Requisitos:** datos llegando

**HOST (dominio 42)**

```bash
ros2 topic echo /pada_1/battery_state --once | grep -E "percentage|current"
```

> **Resultado esperado.** `percentage: 0.80` y `current: 1.2` (positivo en el dock, negativo fuera).

> **Atención.** La batería del Create 3 alimenta también a rcrew-2 y al lidar: si se agota se pierde todo acceso remoto.

### Terminar la sesión

Apaga lo que más consume y pausa el watchdog.

**Requisitos:** robot en el dock

**HOST**

```bash
ssh rcrew@rcrew-2 'pkill -x rplidar_node; pkill -x component_conta'
systemctl --user stop rcrew-watchdog.timer
```

> **Resultado esperado.** Sin salida. `ps` en rcrew-2 ya no muestra rplidar_node.

# 7. JetRover simulado (contenedor jetrover, dominio 77)

**Requisitos comunes:** contenedor `jetrover` corriendo y `source /root/ws/install/setup.bash`.

### Lanzar Gazebo + MoveIt + RViz

Simulación completa: física, sensores, controladores, planificador y RViz con MotionPlanning.

**Requisitos:** requisitos comunes

**CONTENEDOR jetrover**

```bash
ros2 launch jetrover_gazebo gazebo_moveit.launch.py
```

> **Resultado esperado.** Gazebo y RViz abiertos y tres líneas "Configured and activated" (joint_state_broadcaster, arm_controller, gripper_controller). Cierra el panel "Visualize lidar".

### Solo el modelo (sin física)

Para revisar el URDF con sliders de articulaciones.

**Requisitos:** requisitos comunes

**CONTENEDOR jetrover**

```bash
ros2 launch jetrover_description sim_display.launch.py
```

> **Resultado esperado.** RViz con el robot y la ventana joint_state_publisher_gui.

### ¿Está listo el brazo?

Comprueba controladores y articulaciones.

**Requisitos:** simulación corriendo

**CONTENEDOR jetrover**

```bash
ros2 action list | grep follow_joint_trajectory
timeout 10 ros2 topic echo /joint_states --once | head -8
```

> **Resultado esperado.** `/arm_controller/follow_joint_trajectory` y un mensaje con `name: [joint1, …]` y `position: […]`.

### Mover el chasis Mecanum

`linear.y` mueve de lado (solo Mecanum). Siempre terminar con el mensaje vacío.

**Requisitos:** simulación corriendo

**CONTENEDOR jetrover**

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"    # adelante
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {y: 0.2}}"    # lateral
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{angular: {z: 0.5}}"   # giro
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{}"                    # PARAR
```

> **Resultado esperado.** El robot se mueve en Gazebo; `/odom` cambia.

### Mover el brazo (RViz, sin comando)

Panel MotionPlanning → Planning.

**Requisitos:** simulación corriendo

**RViz**

```bash
Planning Group = arm
Start State    = <current>
Goal State     = <random valid> | home | arrastrar el marcador
Botón          = Plan & Execute
```

> **Resultado esperado.** El brazo se mueve en RViz y Gazebo. "Failed" = Start State distinto de &lt;current&gt; o plan viejo.

### Ver el log del launch

Busca errores y activaciones sin hacer scroll en T1.

**CONTENEDOR jetrover**

```bash
LOG=$(ls -td /root/.ros/log/*/ | head -1)launch.log
grep -E "spawner|ERROR|activated" $LOG | tail -20
```

> **Resultado esperado.** Líneas de spawner y "activated".

# 8. Dashboard Foxglove (Create 3 + JetRover)

![Figura 2. Arquitectura del dashboard: relay 78→77 y foxglove_bridge.](img/d_arch.png)

*Figura 2. Arquitectura del dashboard: relay 78→77 y foxglove_bridge.*

### Levantar o detener todo

Abre 8 pestañas en orden (simulaciones, ruedas, relay, LiDAR, bridge).

**Requisitos:** terminal de GNOME · sin contenedores previos

**HOST**

```text
~/docker/dashboard.sh
~/docker/dashboard.sh stop
```

> **Resultado esperado.** En la pestaña JR-fox: "Server listening on port 8765".

### Relay Create 3 → JetRover

Trae el Create 3 al dominio 77 con prefijo `create3/`, frame `world`, conversión Twist→TwistStamped y `/tf_foxglove`.

**Requisitos:** simulaciones corriendo (primero el JetRover)

**CONTENEDOR jetrover**

```bash
python3 /root/ws/scripts/create3_relay.py
```

> **Resultado esperado.** `Relay listo: dominio 78 -> 77, prefijo "create3/"`
>
> `robot_description del Create 3 republicado con prefijo`
>
> Los avisos "sequence size exceeds remaining buffer" son ruido Humble↔Jazzy.

### Bridge de Foxglove

Publica los tópicos por WebSocket (sin /tf) y sirve las mallas.

**Requisitos:** source del workspace · relay corriendo

**CONTENEDOR jetrover**

```bash
export AMENT_PREFIX_PATH=/root/ws/extra_pkgs/irobot_create_description:$AMENT_PREFIX_PATH
ros2 run foxglove_bridge foxglove_bridge --ros-args --params-file /root/ws/scripts/foxglove_bridge.yaml
```

> **Resultado esperado.** "Server listening on port 8765".

### Verificar el dashboard desde ROS

Tópicos y TF que necesita Foxglove.

**Requisitos:** relay corriendo

**CONTENEDOR jetrover**

```bash
ros2 topic list | grep -E "create3|tf_foxglove"
ros2 topic hz /tf_foxglove
ros2 run tf2_ros tf2_echo world create3/base_link
ros2 pkg prefix irobot_create_description
```

> **Resultado esperado.** /create3/cmd_vel, /create3/odom, /create3/robot_description, /tf_foxglove · ≤ 10 Hz · traslación con y ≈ 1.5 · /root/ws/extra_pkgs/irobot_create_description.

### Mover el Create 3 a través del relay

Publica Twist en el dominio 77; el relay lo entrega como TwistStamped en el 78.

**Requisitos:** relay corriendo · ruedas activas

**CONTENEDOR jetrover**

```bash
timeout 3 ros2 topic pub -r 10 /create3/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}}"
```

> **Resultado esperado.** El Create 3 avanza ~30 cm en Gazebo y en Foxglove.

## 8.1 Ajustes de Foxglove

| Ajuste | Valor |
|---|---|
| Conexión | Open connection → Foxglove WebSocket → ws://localhost:8765 |
| Frame | Fixed frame = Display frame = world |
| Scene → Mesh up-axis | Z-up |
| Custom layers → Add URDF | Source = Topic · /create3/robot_description |
| Panel Teleop | Topic = /create3/cmd_vel |
| Barra inferior | Fecha y hora de hoy (use_sim_time: false) |

# 9. Formant

Formant tiene un **agente** (`formant-agent`) que se suscribe a tópicos ROS y los envía a su nube; desde la web (app.formant.io / teleop.formant.io) se ven los streams y se teleopera. Hay agentes en dos lugares: en la **laptop** (pruebas con turtlesim, dominio 0) y en **rcrew-2** (robot real, CycloneDDS).

## 9.1 Servicios del agente

### Estado y reinicio del agente

El agente y el sidecar son servicios systemd del sistema.

**Requisitos:** sudo

**HOST / ROBOT**

```bash
systemctl status formant-agent formant-sidecar
sudo systemctl restart formant-agent
```

> **Resultado esperado.** `Active: active (running)`.

### Ver el log del agente

`-f` sigue el log en vivo; `--since` limita el tiempo.

**Requisitos:** sudo o grupo systemd-journal

**HOST / ROBOT**

```bash
journalctl -u formant-agent -f
journalctl -u formant-agent --since "-10min" | tail -50
```

> **Resultado esperado.** Líneas de arranque, suscripciones y envío de datos sin errores repetidos.

### ¿A qué tópicos está suscrito el agente?

Confirma que el agente recibe lo que configuraste en los streams.

**Requisitos:** agente corriendo

**HOST**

```bash
ssh rcrew@rcrew-2 'journalctl -u formant-agent --since "-3min" | grep -oE "Created subscription for ?[^ ]*" | sort -u'
```

> **Resultado esperado.** /map_formant, /cloud, /odom_relay, /pada_1/battery_state y /pada_1/cmd_vel, sin líneas "Deferring subscription".

### Ver la configuración DDS del agente

El agente del robot debe usar CycloneDDS y leer un cyclonedds.xml accesible (corre como usuario formant).

**Requisitos:** acceso a rcrew-2

**ROBOT rcrew-2**

```bash
cat /etc/systemd/system/formant-agent.service.d/10-ros-dds.conf
ls -la /etc/formant/cyclonedds.xml
```

> **Resultado esperado.** `Environment=ROS_DOMAIN_ID=0`, `ROS_LOCALHOST_ONLY=0`, `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`, `CYCLONEDDS_URI=file:///etc/formant/cyclonedds.xml`
>
> y el archivo con permisos `-rw-r--r--`.

### Aplicar un cambio de configuración del agente

Después de editar el drop-in de systemd hay que recargar y reiniciar.

**Requisitos:** sudo

**ROBOT rcrew-2**

```bash
sudo systemctl daemon-reload
sudo systemctl restart formant-agent
```

> **Resultado esperado.** Sin salida; luego revisar las suscripciones (ficha anterior).

### Detener Formant antes de usar Gazebo en el host

El agente local publica en el dominio 0 (su propio /tf) y contamina simulaciones del host que usen ese dominio (TurtleBot3).

**Requisitos:** sudo

**HOST**

```bash
sudo systemctl stop formant-agent formant-sidecar
# al terminar:
sudo systemctl start formant-agent formant-sidecar
```

> **Resultado esperado.** `systemctl status` muestra `inactive (dead)`.

> **Nota.** "401 Unauthorized" repetido en los logs de **formant-sidecar** no bloquea la telemetría ni el joystick (lo maneja formant-agent). Es un problema aparte del sidecar.

## 9.2 Tópicos que consume Formant (robot real)

| Tópico | Tipo | Para qué | Por qué este y no otro |
|---|---|---|---|
| /map_formant | nav_msgs/OccupancyGrid | Mapa de fondo | /map tiene timestamp viejo y Formant lo descarta |
| /odom_relay | nav_msgs/Odometry | Posición del robot | Republicado localmente; va en la raíz, no en /pada_1 |
| /cloud | sensor_msgs/PointCloud2 | Lidar | — |
| /pada_1/cmd_vel | geometry_msgs/Twist | Joystick | — |
| /pada_1/battery_state | sensor_msgs/BatteryState | Batería (campo percentage) | — |

### Comprobar los tópicos de Formant

Antes de culpar a Formant, confirmar que los tópicos existen y publican.

**Requisitos:** dominio 42 (laptop)

**HOST (dominio 42)**

```bash
for T in /map_formant /odom_relay /cloud /pada_1/battery_state; do
  printf "%-24s " $T; timeout 15 ros2 topic echo $T --once --no-daemon >/dev/null 2>&1 && echo OK || echo "SIN DATOS"
done
```

> **Resultado esperado.** Cuatro líneas OK.

## 9.3 Configuración en la web (sin comandos)

| Tarea | Ruta en Formant | Valores clave |
|---|---|---|
| Stream de localización | app.formant.io → Settings → Devices → create3 → Telemetry → + Add Stream → ROS Localization | Base reference frame = map · Odometry /odom_relay · Map /map_formant · Point cloud /cloud · Path y Goal vacíos · 1 Hz |
| Stream de batería | Telemetry → + Add Stream → ROS topic | /pada_1/battery_state, path percentage |
| Mostrar el mapa en teleop | Devices → create3 → Teleoperation → LOCALIZATION → + → create3.localization; Primary view 1 = create3.localization | Solo aparece después de recibir datos |
| Joystick | Teleoperation → JOYSTICK → + → Add ROS Topic → /pada_1/cmd_vel | Es un control, no una vista: Sidebar view = None |
| Manejar | Shift + T (modo teleop) → desacoplar el robot → UNLOCK → joystick | Acoplado, el robot ignora cmd_vel |

> **Atención.** No pongas `/plan` ni `/goal_pose` en el stream de localización: no publican en reposo y el agente queda en "Deferring subscription", bloqueando todo el stream.
>
> El joystick solo funciona en vistas de **teleop.formant.io**; los dashboards de app.formant.io muestran datos pero no controlan.

# 10. Diagnóstico: el orden correcto

Muchas causas distintas dan el mismo síntoma ("no se ve" / "no se mueve"). Ir de afuera hacia adentro y no saltar pasos:

| # | Pregunta | Comando |
|---|---|---|
| 1 | ¿Estoy en el lugar correcto? (host / contenedor / robot) | mirar el prompt · docker ps |
| 2 | ¿Dominio y RMW correctos? | echo $ROS_DOMAIN_ID · printenv \| grep RMW |
| 3 | ¿Hice source? | ros2 pkg prefix &lt;paquete&gt; |
| 4 | ¿Existe el tópico y quién lo publica? | ros2 topic info &lt;tópico&gt; -v |
| 5 | ¿Llegan mensajes? | timeout 10 ros2 topic echo &lt;tópico&gt; --once |
| 6 | ¿Los datos CAMBIAN? | comparar dos echo seguidos de /odom |
| 7 | ¿La cadena TF está completa? | ros2 run tf2_ros tf2_echo &lt;padre&gt; &lt;hijo&gt; |
| 8 | ¿Los controladores están activos? | list_controllers (servicio) · ros2 action list |
| 9 | ¿El visualizador es una instancia fresca? | reconectar Foxglove · matar RViz viejo |

> **Concepto.** Valida siempre el instrumento con un control positivo antes de concluir que algo no publica: un CLI trabado y un sistema roto se ven exactamente igual.

# 11. Rutinas del día a día

## 11.1 Simulación Create 3 + JetRover + Foxglove

**HOST**

```text
~/docker/dashboard.sh                    # todo en pestañas
# Foxglove → ws://localhost:8765 → layout "Create3 y JetRover Dashboard"
~/docker/dashboard.sh stop               # al terminar
```

## 11.2 Robot real Create 3 + RViz + Formant

**HOST**

```bash
# Empezar
ssh rcrew@rcrew-2 uptime && ssh rcrew@rcrew-2 ping -c2 192.168.1.116
ssh rcrew@rcrew-2 'nohup setsid ~/.local/bin/start-zenoh-bridge.sh > /tmp/zenoh-bridge.log 2>&1 </dev/null &'
ssh rcrew@rcrew-2 'nohup setsid ~/run_nav_casa.sh > /tmp/nav2.log 2>&1 </dev/null &'
systemctl --user restart rcrew-zenoh-client rcrew-tf-relay rcrew-map-local
systemctl --user start rcrew-watchdog.timer
~/.local/bin/rviz-pada1.sh
ROS_DOMAIN_ID=42 ros2 action send_goal /pada_1/undock irobot_create_msgs/action/Undock "{}"
# Formant: teleop.formant.io → Shift+T → UNLOCK → joystick

# Terminar
ROS_DOMAIN_ID=42 ros2 action send_goal /pada_1/dock irobot_create_msgs/action/Dock "{}" --feedback
ROS_DOMAIN_ID=42 ros2 topic echo /pada_1/battery_state --once | grep current    # positivo
ssh rcrew@rcrew-2 'pkill -x rplidar_node; pkill -x component_conta'
systemctl --user stop rcrew-watchdog.timer
```

## 11.3 Las reglas de oro

1. **Mira el prompt** antes de ejecutar: docker y git en el HOST; ros2 en el contenedor o el robot.
2. **`echo $ROS_DOMAIN_ID`** antes de diagnosticar. Robot real: además, **CycloneDDS**.
3. **`source`** en cada terminal nueva.
4. **`timeout`** delante de cada `echo --once` o `hz`; **`--no-daemon`** si el CLI se comporta raro.
5. **Nunca `exit`** en la terminal principal de un contenedor.
6. Robot real: **vigila la batería** — si se agota, se pierde todo acceso remoto.
7. En Foxglove y Formant: **lee el mensaje del ícono rojo** antes de cambiar nada.
