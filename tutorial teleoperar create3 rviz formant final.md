# Teleoperar el iRobot Create 3 con RViz y Formant

Tutorial paso a paso: conceptos, comandos y configuración, desde cero hasta manejar el robot con el joystick viendo el mapa.

**Robot:** iRobot Create 3, namespace `/pada_1` · **A bordo:** rcrew-2 (Raspberry Pi, ROS 2 Humble) · **Estación:** laptop (ROS 2 Humble) · **Fecha:** 10 de septiembre de 2026

## 1. A dónde vamos a llegar

El objetivo son estas dos pantallas funcionando en simultáneo.

**RViz**

*(Captura: mapa de la casa, costmap local con las capas de inflación, retornos del lidar en rojo, el robot como disco oscuro con la flecha magenta de AMCL. Arriba a la izquierda, **Global Status: Ok** y el display Map en **Status: Ok** con 347 x 191.)*

**Formant**

*(Captura: el mismo mapa a pantalla completa, el robot como punto celeste, y el **joystick como control superpuesto abajo a la derecha**. Abajo, la barra de estado con CPU, memoria, ping, conexión P2P y la batería.)*

> **Lo más importante de todo el tutorial.** RViz y Formant son **caminos independientes**. Formant tiene un agente en el robot que habla con su nube; RViz corre en la laptop y recibe por un puente Zenoh. Lo único que comparten es el origen: los topics del robot.
>
> Por eso, si el robot no se mueve en RViz, la pregunta correcta nunca es "por qué Formant no le avisa a RViz", sino "¿se está moviendo el robot, y su odometría lo refleja?".

## 2. Conceptos que hay que entender antes

### 2.1 El RMW: por qué todo depende de CycloneDDS

ROS 2 no tiene un master central: los nodos se descubren solos por DDS. Y ROS 2 habla DDS a través de una capa intermedia, el **RMW**. Humble trae FastDDS de fábrica, pero **el Create 3 usa CycloneDDS**.

> **El error más caro de este proyecto.** Con el RMW equivocado el descubrimiento funciona —`ros2 topic list` muestra los 24 topics con sus tipos— pero **no llega ni un solo dato**. Parece que el robot está dormido.
>
> Todo comando que toque al Create 3 necesita:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=file:///home/rcrew/create3/cyclonedds.xml
```

> Y lo mismo vale para el **agente de Formant**: mientras usó FastDDS nunca recibió la batería ni la odometría del robot. Ver la sección 5.1.

### 2.2 QoS: el que falla en silencio

DDS no conecta publisher y subscriber si sus políticas son incompatibles, y **no avisa**.

| Publisher | Subscriber | ¿Conectan? |
| --- | --- | --- |
| RELIABLE | BEST_EFFORT | Sí |
| BEST_EFFORT | RELIABLE | **No** |
| TRANSIENT_LOCAL | VOLATILE | Sí, pero sin recibir lo ya publicado |
| VOLATILE | TRANSIENT_LOCAL | **No** |

`TRANSIENT_LOCAL` es para lo que se publica **una sola vez**: el mapa, los transforms estáticos, el URDF. En RViz esto se ve como un display en `Status: Warn` con `0 x 0` y ningún error.

### 2.3 TF y el Fixed Frame

```
map --> odom --> base_link --> laser, imu, bumpers, ruedas
```

**map** es el mapa fijo, **odom** la odometría (suave pero deriva) y **base_link** el robot. AMCL corrige la deriva publicando `map -> odom`.

> Si falta un eslabón, **RViz no dibuja nada** aunque los datos lleguen perfectos. El diagnóstico es una línea:

```bash
ros2 run tf2_ros tf2_echo map base_link
```

> Si dice `Invalid frame ID "map"`, AMCL no está localizando — casi siempre porque falta `/scan`, o sea el lidar.

### 2.4 Dos namespaces

| Namespace | Quién publica | Ejemplos |
| --- | --- | --- |
| `/pada_1/*` | El Create 3 | odom, imu, cmd_vel, battery_state |
| raíz `/*` | nav2 y el lidar, en rcrew-2 | map, scan, plan, amcl_pose, odom_relay |

Un nodo `tf_relay` copia `/pada_1/tf` a `/tf` para unir los dos árboles.

### 2.5 Por qué hace falta un puente para RViz

La laptop está en `192.168.0.x` y el robot en `192.168.1.x`: LANs distintas, unidas solo por Tailscale. El descubrimiento DDS usa multicast, que no cruza una VPN. Y además DDS anuncia sus IPs *dentro* del mensaje, y las del robot no son enrutables desde afuera.

**zenoh-bridge-ros2dds** habla DDS nativo de cada lado y conecta ambos por un único puerto TCP.

## 3. Parte 1 — Levantar el robot

### 3.1 Chequeos previos

En orden. Si uno falla, no tiene sentido seguir.

```bash
# 1. La laptop llega a rcrew-2?
ssh rcrew@rcrew-2 uptime

# 2. rcrew-2 llega al Create 3?
ssh rcrew@rcrew-2 ping -c2 192.168.1.116

# 3. El lidar esta enchufado?
ssh rcrew@rcrew-2 ls -la /dev/ttyUSB0
```

> Si falla el paso 2, el Create 3 está dormido o fuera de la WiFi. **No hay comando que lo arregle**: hay que despertarlo físicamente o ponerlo en el dock.
>
> Si falla el paso 1, o rcrew-2 está apagada, o el Create 3 se quedó sin batería — **la Pi se alimenta del robot**, así que cuando el robot muere se apagan los dos.

### 3.2 Puente Zenoh

```bash
ssh rcrew@rcrew-2 'nohup setsid ~/.local/bin/start-zenoh-bridge.sh \
    > /tmp/zenoh-bridge.log 2>&1 </dev/null &'

# verificar a los 10 segundos
ssh rcrew@rcrew-2 'ss -ltn | grep 7447'
```

### 3.3 nav2 y el lidar

```bash
ssh rcrew@rcrew-2 'nohup setsid ~/run_nav_casa.sh > /tmp/nav2.log 2>&1 </dev/null &'
```

Tarda unos 60 segundos. Tres verificaciones imprescindibles:

```bash
# 1. UN solo contenedor (si hay dos, ninguno carga nada)
ssh rcrew@rcrew-2 'ps -eo comm | grep -c component_conta'      # debe dar 1

# 2. Los 9 nodos cargados
ssh rcrew@rcrew-2 'grep -c "Loaded node" /tmp/nav2.log'         # debe dar 9

# 3. El lidar gira DE VERDAD
ssh rcrew@rcrew-2 'P=$(pgrep -x rplidar_node); grep rchar /proc/$P/io; sleep 5; grep rchar /proc/$P/io'
```

> **Dos fallas silenciosas que cuestan horas.**
>
> **Contenedores duplicados:** si quedó un nav2 viejo corriendo, los dos se llaman `/nav2_container`, el `load_composable_nodes` no sabe a cuál mandarle la carga y **queda colgado sin escribir ningún error**. El proceso existe y no hace nada.
>
> **El lidar que no gira:** el nodo arranca, lee el número de serie, dice `health status: OK` y anuncia el modo de escaneo — todo eso aunque el motor esté parado. La única forma de saberlo es mirar `rchar`: girando lee decenas de KB/s; si queda clavado en ~62 KB solo hizo el handshake y es un problema de alimentación.

### 3.4 Confirmar que nav2 está activo

```bash
ssh rcrew@rcrew-2 'source /opt/ros/humble/setup.bash
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
  export CYCLONEDDS_URI=file:///home/rcrew/create3/cyclonedds.xml
  for N in map_server amcl controller_server planner_server bt_navigator; do
    printf "%-19s " $N
    ros2 service call /$N/get_state lifecycle_msgs/srv/GetState | grep -oE "label=.[a-z_]+"
  done'
```

Los cinco tienen que decir `active`. Son **lifecycle nodes**: que el proceso exista no alcanza, tienen que estar activados.

## 4. Parte 2 — RViz

### 4.1 Las cuatro piezas de la laptop

Corren como servicios systemd de usuario y arrancan solos al iniciar sesión. Cada una existe por una limitación concreta:

| Servicio | Por qué existe |
| --- | --- |
| `rcrew-zenoh-client` | Trae los topics del robot por Tailscale, en **dominio 42** para aislarse del formant-agent local, que corre en dominio 0 y publica su propio `/tf`. |
| `rcrew-tf-relay` | El puente reemite todo como VOLATILE, pero tf2 exige TRANSIENT_LOCAL en `/tf_static`. Sin esto los ~25 frames de sensores nunca llegan. Republica en `/tf_static_tl`. |
| `rcrew-create3-description` | El robot **no publica** `/robot_description`. Sin URDF, RViz solo lo dibuja con ejes TF: 25 frames con nombres superpuestos, ilegible. |
| `rcrew-map-local` | `/map` es TRANSIENT_LOCAL y se publica una vez. Sobre la VPN compite con los costmaps y a veces no llega. Como es estático, se sirve local. |

```bash
systemctl --user status 'rcrew-*'
systemctl --user restart rcrew-zenoh-client rcrew-tf-relay rcrew-map-local
```

### 4.2 Verificar que llegan los datos

```bash
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=42
ros2 daemon stop

for T in /map /scan /tf /pada_1/odom /local_costmap/costmap; do
  printf "%-28s " "$T"
  timeout 15 ros2 topic echo $T --once --no-daemon >/dev/null 2>&1 && echo OK || echo "SIN DATOS"
done
```

> Usar siempre `--no-daemon`. El daemon de ros2cli se corrompe seguido y devuelve listas vacías o tracebacks de xmlrpc, haciendo creer que no hay datos cuando sí los hay.

### 4.3 La cadena TF

```bash
ros2 run tf2_ros tf2_echo map base_link --ros-args -r /tf_static:=/tf_static_tl
```

Tiene que devolver una traslación. Si tarda, esperar: AMCL necesita unos scans para converger y empezar a publicar `map -> odom`.

### 4.4 La configuración de RViz: la tabla de QoS

Archivo `~/.rviz2/pada1.rviz`, **Fixed Frame: map**. Esto es lo más importante del archivo:

| Display | Topic | Durability | Reliability | Activo |
| --- | --- | --- | --- | --- |
| Map | `/map` | **Transient Local** | Reliable | Sí |
| Global Costmap | `/global_costmap/costmap` | Volatile | Reliable | **No** |
| Local Costmap | `/local_costmap/costmap` | Volatile | Reliable | Sí |
| LaserScan | `/scan` | Volatile | Best Effort | Sí |
| AMCL Pose | `/amcl_pose` | Volatile | Reliable | Sí |
| Create3 (RobotModel) | `/robot_description` | **Transient Local** | Reliable | Sí |
| TF | — | — | — | **No** |

> **Regla de oro.** Lo que llega **por el puente** va en Volatile, porque el puente reemite así. Lo que se publica **localmente** (`/map`, `/robot_description`) va en Transient Local. Mezclarlos deja el display en Warn con `0 x 0` y sin ningún error.

**Global Costmap va apagado**: es el payload más grande y con los tres activos uno se queda sin cargar. **TF va apagado**: con 25 frames tapa el robot.

### 4.5 Abrir RViz

```bash
~/.local/bin/rviz-pada1.sh
```

O solo RViz, sin tocar el resto:

```bash
export ROS_DOMAIN_ID=42
ros2 run rviz2 rviz2 -d ~/.rviz2/pada1.rviz --ros-args -r /tf_static:=/tf_static_tl
```

El único remapeo necesario apunta al topic reemitido por el relay.

> **Si RViz muestra datos viejos o congelados**, casi siempre es una instancia vieja suscrita a un puente que ya no existe. No se recupera sola, y `pkill -x rviz2` no alcanza: el proceso padre `ros2 run rviz2` corre como `python3` y sobrevive.

```bash
for P in $(pgrep -x rviz2); do kill $P; done
```

Con esto tenés que llegar a la primera imagen de la sección 1.

## 5. Parte 3 — Formant

### 5.1 Primero: el agente tiene que usar CycloneDDS

> **Sin este paso, nada de lo que sigue funciona de forma confiable.** El agente viene configurado con FastDDS. Con ese RMW **nunca recibió** `/pada_1/odom` ni `/pada_1/battery_state`, y `/odom_relay` le llegaba de forma intermitente. Sin odometría, el stream de localización no emite datos y el mapa no aparece.

Hay un detalle no obvio: el agente corre como usuario `formant` y `/home/rcrew` tiene permisos 750, así que **no puede leer el `cyclonedds.xml` del robot**. Hay que dejarle una copia accesible.

```bash
sudo mkdir -p /etc/formant
sudo cp /home/rcrew/create3/cyclonedds.xml /etc/formant/cyclonedds.xml
sudo chmod 644 /etc/formant/cyclonedds.xml

sudo cp /etc/systemd/system/formant-agent.service.d/10-ros-dds.conf \
        /etc/systemd/system/formant-agent.service.d/10-ros-dds.conf.bak

sudo tee /etc/systemd/system/formant-agent.service.d/10-ros-dds.conf >/dev/null <<'EOF'
[Service]
Environment=ROS_DOMAIN_ID=0
Environment=ROS_LOCALHOST_ONLY=0
Environment=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
Environment=CYCLONEDDS_URI=file:///etc/formant/cyclonedds.xml
EOF

sudo systemctl daemon-reload
sudo systemctl restart formant-agent
```

Verificar que ahora sí recibe:

```bash
ssh rcrew@rcrew-2 'journalctl -u formant-agent --since "-3min" \
  | grep -oE "Created subscription for ?[^ ]*" | sort -u'
```

Tienen que aparecer `/map_formant`, `/cloud`, `/odom_relay`, `/pada_1/battery_state` y `/pada_1/cmd_vel`, sin líneas de `Deferring`.

### 5.2 Los topics que consume Formant

| Topic | Tipo | Para qué |
| --- | --- | --- |
| `/map_formant` | nav_msgs/OccupancyGrid | Mapa de fondo |
| `/odom_relay` | nav_msgs/Odometry | Posición del robot |
| `/cloud` | sensor_msgs/PointCloud2 | Lidar |
| `/pada_1/cmd_vel` | geometry_msgs/Twist | Joystick |
| `/pada_1/battery_state` | sensor_msgs/BatteryState | Batería |

> **Dos detalles que costaron mucho tiempo.**
>
> Usar `/map_formant` y **no** `/map`: un nodo `map_republisher` reemite el mapa con timestamp fresco cada segundo, y Formant descarta el original por antiguo.
>
> Usar `/odom_relay` y **no** `/pada_1/odom`: el relay lo republica desde un nodo local. Y ojo, va en la **raíz**, no bajo `/pada_1/`.
>
> Y **no** poner `/plan` ni `/goal_pose` en el stream de localización: ninguno de los dos publica en reposo, y el agente se queda esperando en un bucle de `Deferring subscription` que bloquea todo el stream.

### 5.3 Crear el stream de localización

```
app.formant.io  -->  Settings  -->  Devices  -->  create3
                -->  pestaña Telemetry  -->  + Add Stream  -->  ROS Localization
```

| Campo | Valor |
| --- | --- |
| Telemetry stream name | `create3.localization` |
| ROS version | ROS 2 |
| **Base reference frame** | **map** |
| Odometry topic | `/odom_relay` |
| Map topic | `/map_formant` |
| Point cloud topics | `/cloud` |
| Path topic | *vacío* |
| Goal topic | *vacío* |
| Rate limit (Hz) | 1 |
| On demand | desactivado |

> **Base reference frame es un frame de TF, no un topic.** Poniendo `map`, Formant transforma la odometría —que está en el frame `odom`— al frame del mapa usando el árbol TF. Si se deja en `odom`, el robot aparece corrido y con deriva creciente.

### 5.4 Habilitar el stream para teleoperación

Crear el stream no lo hace visible. Hay que registrarlo también en la pestaña de teleoperación:

```
Settings --> Devices --> create3 --> pestaña Teleoperation
      --> bajar hasta la seccion LOCALIZATION --> +  --> elegir create3.localization
```

Y asignarlo a la vista, en los desplegables de arriba de esa misma pestaña:

```
Primary view 1  -->  create3.localization
```

> El stream solo aparece en el desplegable **después de que Formant haya recibido datos**. Si no figura, el agente todavía no emitió ningún datapoint: revisar la sección 5.1.

### 5.5 El joystick

Este es el paso que faltaba al final. En la **misma pestaña Teleoperation**:

```
bajar hasta la seccion JOYSTICK  -->  +  -->  "Add ROS Topic"  -->  /pada_1/cmd_vel
```

> **El joystick NO es una vista.** Es un **control** que se superpone abajo a la derecha, junto al botón UNLOCK.
>
> Si por error se pone `cmd_vel` en **Sidebar view**, aparece como un panel vacío arriba a la derecha: está intentando *mostrar* los datos del topic en vez de darte un control para *publicarlos*. Los Sidebar view tienen que quedar en `None`.

Con esto se llega a la segunda imagen de la sección 1.

### 5.6 Manejar

1. Entrar en modo teleoperación (**Shift + T** o desde el menú del dispositivo)
2. **Desacoplar el robot**: acoplado, el Create 3 ignora `cmd_vel` por diseño
3. Destrabar el botón **UNLOCK** abajo a la derecha
4. Mover el joystick

```bash
ssh rcrew@rcrew-2 'source /opt/ros/humble/setup.bash
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
  export CYCLONEDDS_URI=file:///home/rcrew/create3/cyclonedds.xml
  ros2 action send_goal /pada_1/undock irobot_create_msgs/action/Undock "{}"'
```

## 6. Diagnóstico: el orden correcto

Cuatro causas distintas dan el mismo síntoma. Este orden va de lo más externo hacia adentro; si un paso falla, no tiene sentido seguir.

| # | Pregunta | Comando |
| --- | --- | --- |
| 1 | ¿El Create 3 está en la red? | `ping -c2 192.168.1.116` |
| 2 | ¿Se mueve físicamente? | mirarlo |
| 3 | ¿La odometría **cambia**? | contar poses distintas, no si "llegan datos" |
| 4 | ¿Llega a la laptop? | `ros2 topic echo /tf --once --no-daemon` |
| 5 | ¿La cadena TF está completa? | `tf2_echo map base_link` |
| 6 | ¿RViz es una instancia fresca? | `ps -eo lstart,comm | grep rviz2` |

> **Distinguir "llegan datos" de "los datos cambian".** El Create 3 puede publicar odometría a 20 Hz con valores **congelados** mientras se mueve físicamente. Pasó el 2/9/2026: 441 mensajes, una sola pose distinta. `echo --once` devuelve OK igual y te manda a depurar RViz, que no tiene nada que ver. Se arregla con un **power-cycle del robot**.

### 6.1 Trampas de instrumentación

| Herramienta | Falla | Solución |
| --- | --- | --- |
| `ros2 topic hz` | Vacío aunque lleguen 300 msgs en 15 s | Contar con un suscriptor propio |
| `ros2 topic list` | Lista vacía o traceback xmlrpc | `ros2 daemon stop` |
| `echo > archivo` | Bufferea: el archivo parece vacío | `stdbuf -oL` |
| `pgrep -x nombre_largo` | Los nombres se truncan a **15 caracteres** | `pgrep -x component_conta` |

**Validar siempre el instrumento con un control positivo** antes de concluir que algo no publica. Un monitor roto y un sistema roto se ven exactamente igual.

## 7. Problemas frecuentes

| Síntoma | Causa | Solución |
| --- | --- | --- |
| Se ven los topics pero no llega nada | FastDDS contra el Create 3 | `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp` |
| El puente descubre 0 entidades | Falta `ROS_DISTRO=humble`; asume 'iron' | Exportarla en el script |
| Display en Warn con 0 x 0 | Durability incompatible | Ver la regla de oro (4.4) |
| Se corta todo el flujo de golpe | Ruta RELIABLE congestionada; zenoh cierra el transporte | `reliable_routes_blocking: false` |
| nav2 arranca pero no carga nodos | Dos contenedores con el mismo nombre | Matar todos y lanzar uno solo |
| El stream no aparece en el desplegable | Formant no recibió ningún datapoint | Revisar 5.1 y 5.2 |
| Panel vacío arriba a la derecha | `cmd_vel` puesto como Sidebar view | Moverlo a la sección JOYSTICK |
| El joystick no mueve el robot | Robot acoplado, o UNLOCK trabado | Desacoplar y destrabar |
| Todo muerto tras un reinicio | Procesos con `nohup`, sin systemd | Relanzar 3.2 y 3.3 |

## 8. La batería: el punto más frágil

> **La batería del Create 3 alimenta todo el sistema:** el robot, el lidar y la Raspberry rcrew-2. Cuando se agota no se apaga solo el robot, **se pierde todo acceso remoto** hasta que alguien lo toque físicamente.
>
> El consumo de base es alto: nav2 más el lidar girando son varios vatios continuos aunque el robot esté quieto. **Fuera del dock la autonomía es de pocas horas.**
>
> Y hay un riesgo peor: si la aplicación ROS del Create 3 falla mientras está desacoplado, se queda sin Dock, sin `cmd_vel` y sin odometría — **atrapado, sin salida remota**. Pasó el 9/9/2026.

```bash
ros2 topic echo /pada_1/battery_state --once | grep -E "percentage|current"
```

`percentage` va de 0 a 1. **`current` positivo = cargando.** Es más confiable que `dock_status`, que dice `true` apenas hace contacto.

Hay un watchdog en la laptop que cada 90 segundos verifica que lleguen datos y avisa por notificación de escritorio al **30%** y al **20%**.

### 8.1 Volver al dock

```bash
ros2 action send_goal /pada_1/dock irobot_create_msgs/action/Dock "{}" --feedback
```

> **La acción Dock NO navega.** Solo gira sobre sí misma buscando la señal infrarroja, con alcance de uno o dos metros y línea de vista. Si el dock no está cerca, gira, no lo encuentra y aborta gastando batería.
>
> Verificar `dock_visible` antes de lanzarla. Y darle **tiempo suficiente**: con un timeout de 90 segundos se corta a mitad de maniobra, y matar el cliente **no cancela el goal**.
>
> **No hay ninguna pose del dock guardada en el mapa**, así que nav2 tampoco puede llevarlo. Vale la pena grabarla la próxima vez que esté acoplado y bien localizado.

## 9. Rutina diaria

### Empezar

```bash
# chequeos
ssh rcrew@rcrew-2 uptime
ssh rcrew@rcrew-2 ping -c2 192.168.1.116

# robot
ssh rcrew@rcrew-2 'nohup setsid ~/.local/bin/start-zenoh-bridge.sh > /tmp/zenoh-bridge.log 2>&1 </dev/null &'
ssh rcrew@rcrew-2 'nohup setsid ~/run_nav_casa.sh > /tmp/nav2.log 2>&1 </dev/null &'

# laptop
systemctl --user restart rcrew-zenoh-client rcrew-tf-relay rcrew-map-local
systemctl --user start rcrew-watchdog.timer

# rviz
~/.local/bin/rviz-pada1.sh

# desacoplar para teleoperar
ros2 action send_goal /pada_1/undock irobot_create_msgs/action/Undock "{}"
```

### Terminar

```bash
# al dock (verificar dock_visible primero)
ros2 action send_goal /pada_1/dock irobot_create_msgs/action/Dock "{}" --feedback

# confirmar carga: current POSITIVO
ros2 topic echo /pada_1/battery_state --once | grep current

# bajar nav2 y el lidar, que son el grueso del consumo
ssh rcrew@rcrew-2 'pkill -x rplidar_node; pkill -x component_conta'

# pausar el watchdog (sin /tf daria falsos positivos)
systemctl --user stop rcrew-watchdog.timer
```

## 10. Pendientes conocidos

| Pendiente | Impacto |
| --- | --- |
| Servicios systemd en el robot sin instalar | Cada reinicio de rcrew-2 o caída del lidar exige relanzar a mano |
| No hay pose del dock en el mapa | nav2 no puede mandar el robot a cargar de forma autónoma |
| AMCL con incertidumbre alta | ±0,7 m y ±47° tras mover el robot a mano; converge teleoperando un rato |
| Configuración de escena 3D en teleop.formant.io | No se pudo crear una nueva con capa Occupancy Map; se resolvió usando la vista clásica de app.formant.io |

> **Estado al 10 de septiembre de 2026:** sistema verificado de punta a punta. El joystick de Formant mueve el robot, el movimiento se ve en el mapa de Formant y en RViz simultáneamente.
