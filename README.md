# Robot unitree go2 y SLAM

## Repositorio del robot go2 usado
En este apartado se explica los pasos realizados para que, a partir del repositorio de anujjain-dev, se pueda integrar correctamente la función de mapeo de SLAM toolbox. La versión de Ros usada en este repositorio es HUMBLE.
A continuación se comparte el link del repositorio de [anujjain](https://github.com/anujjain-dev/unitree-go2-ros2).
Recuerde también de descargar SLAM toolbox.

## Cambios a realizar para emplear SLAM

### 1 Cambiar el sensor
En el punto 2.6 del archivo README del repositorio original se explica como cambiar el sensor LIDAR 3D por uno tipo laser. Este cambio debe de hacerse para tener mensajes tipo /scan necesario para el mapeo.

### 2 Cambios en el archivo slam.launch.py
En el archivo `slam.launch.py` que se encuentra en la ruta `src/unitree-go2-ros2/robots/configs/go2_config/launch/slam.launch.py` se tiene, originalmente, la siguiente línea de código:

```bash
slam_launch_path = PathJoinSubstitution(
        [FindPackageShare('champ_navigation'), 'launch', 'slam.launch.py']
    )
```
Esto debe ser cambiado por lo siguiente:

```bash
slam_launch_path = PathJoinSubstitution(
        [FindPackageShare('slam_toolbox'), 'launch', 'online_async_launch.py']
    )
```
Con este cambio la función de mapeo se encuentra habilitada. Para probarlo se lanza la simulación en gazebo y el rviz con:

```bash
ros2 launch go2_config gazebo_velodyne.launch.py rviz:=true
```
En rviz agregue el plugin de Map y agregue como tópico /map. Si desea observar en rviz el funcionamiento del laser puede cambiar el tópico de LaserScan por /scan 
En otra terminal lance SLAM:

```bash
ros2 launch go2_config slam.launch.py sim:=true
```

Finalmente, en una terminal aparte active la herramienta de teleop para controlar el robot y observar la creación del mapa.

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

### 3 Cambios en el archivo gait.yaml
En el archivo `gait.yaml` que se encuentra en la ruta `src/unitree-go2-ros2/robots/configs/go2_config/config/gait/gait.yaml` cambiar el parámetro `odom_saler: 0.9` a  `odom_saler: 2.2`. Este valor puede ser modificado dependiendo del funcionamiento de su simulación en gazebo, con valores superiores a 2 y menores a 2.6 se tiene un buen resultado al momento de realizar el mapeo.
Luego de realizar estos cambios ya debería ser posible observar la creación de un mapa bastante preciso, pero si no, el siguiente cambio puede ayudar.

### 4 Cambios en el archivo slam.yaml (opcional)
Los siguientes cambios ayudan a que el mapa se genere o actualice a una mayor velocidad. En el archivo `slam.yaml` en la ruta `src/unitree-go2-ros2/robots/configs/go2_config/launch/slam.launch.py` se cambia el parametro `map_update_interval: 5.0` a `map_update_interval: 1.0`, además, tambien se puede agregar al final del archivo:
```bash
min_pass_through: 1
occupancy_threshold: 0.1
```
Con estos cambios realizados se puede realizar un mapeo bastante preciso.


## Agregar el mundo de Small_house

Para probar estos cambios se empleo el archivo .world obtenido del repositorio de [mlherd/Dataset-of-Gazebo-Worls-Models-and-Maps](https://github.com/mlherd/Dataset-of-Gazebo-Worlds-Models-and-Maps/tree/master), más especificamente, se usó el archivo [small_house](https://github.com/mlherd/Dataset-of-Gazebo-Worlds-Models-and-Maps/tree/master/worlds/small_house).
Por la forma en como está configurado el repositorio del robot go2 se deben de realizar algunos cambios en los archivos que se obtiene luego de descargar el archivo `small_house.zip`.

### Archivo small_house.world
En este archivo se debe de comentar o borrar las siguientes líneas de código:

```bash
    <include>
      <pose>-3.5 -4.5 0.0 0.0 0.0 1.58</pose>
      <uri>model://turtlebot3_waffle_pi</uri>
    </include>
```

Este cambio se hace debido a que, en el momento de lanzar la simulación de gazebo, incluye al robot `turtlebot3` y eso ocasiona un error.

### Archivos model.sdf
Los archivos dentro de la carpeta `models` se encargan de armar el escenario en donde se realizará la simulación de gazebo. Dentro de esta carpeta hay más carpetas de cada una de las partes del escenario, por ejemplo `aws_robomaker_residential_AirconditionerA_01`, dentro, por lo general, hay 4 archivos. El archivo `model.sdf` hay 2 líneas de código parecidas a esta:

```bash
<uri>file://models/aws_robomaker_residential_AirconditionerA_01/meshes/aws_AirconditionerA_01_collision.DAE</uri>
```

Estas líneas deben de ser cambiagas por lo siguiente:
```bash
<uri>model://aws_robomaker_residential_AirconditionerA_01/meshes/aws_AirconditionerA_01_collision.DAE</uri>
```

Este cambio se debe de hacer en todos los archivos `model.sdf` de todas las carpetas. Se incluye una carpeta dentro del `src` en donde el cambio fue aplicado a todos los modelos.

### Agregar los archivos a gazebo y el launch
Luego de hacer los cambios en el archivo `small_house.world` se debe de agregar el mundo en el archivo launch del repositorio, esto se lo hace en `src/unitree-go2-ros2/robots/configs/go2_config/launch/gazebo_velodyne.launch.py` (archivo usado para lanzar el robot y el sensor, si se desea tambien se puede hacer lo mismo en el archivo `gazebo.launch.py`). Se debe de cambiar la siguiente línea de código:

```bash
default_world_path = os.path.join(config_pkg_share, "worlds/default.world")
```

por:

```bash
default_world_path = os.path.join(config_pkg_share, "worlds/small_house.world")
```

En sí, cualquier escenario que se desee usar en gazebo debe de ser agregado en esta línea de código. Además, se debe incluir el archivo `small_house.world` dentro de la ruta `src/unitree-go2-ros2/robots/configs/go2_config/worlds`.
Luego, se debe de agregar las carpetas de los modelos dentro de gazebo. Cada una de las carpetad models, por ejemplo `aws_robomaker_residential_AirconditionerA_01`, debe ser agregado en la carpta models dentro de gazebo, la ruta de esa carpeta, en mi caso, es  `~/.gazebo/models`, con ello, también se debe de incluir en esta ruta la carpeta `photos`.
Con estos cambios ya es posible realizar la simulación del robot en este escenario.

## Aspectos a tener en cuenta
Estos son algunos de los inconvenientes que se presentan al momento de probar la simulación:
* En algunas ocasiones, al momento de lanzar la simulación, gazebo se cierra inesperadamente, por lo que se debe de terminar el proceso y volver a lanzarlo.
* En ocasiones, al momento de lanzar gazebo, se presenta el robot "acostado" en el suelo, la forma de corregir esto es cerrando el proceso y volver a lanzar la simulación. Si el error persiste, la forma de corregirlo (al menos la que yo probé) es reiniciar la computadora.
* La simulación en gazebo puede ser algo pesada, si se presentan problemas de rendimiento, se pude desactivar las sombras en el apartado de `Scene` en la opción `shadows`.
* Cuando se inicia la simulación es posible ver al robot deslizarse por el suelo sin motivo aparante, es probable que en algún momento se haya cambiado un parametro que haga que surja este problema, la forma de solucionarlo (por lo que he probado) es cambiar, dentro de la opción de `Physics`, los parametros `real time update rate` y `max step size` a 200 y 0,005, respectivamente. Con ello el robot dejará de deslizarse, sin embargo, esto ocasiona problemas con algunas de las estructuras del escenario, tal como movimientos repetidos o la desaparación de estos objetos de la escena (no he encontrado solución a ese problema pero no altera el funcionamiento de la simulación).

