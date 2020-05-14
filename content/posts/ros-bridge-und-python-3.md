---
title: "Ros Bridge Und Python 3"
date: 2019-10-25T18:07:36+02:00
tags: ["ros"]
draft:  false
---
Nachdem ROS (melodic) und CARLA installiert werden, soll rosbridge für CARLA eingerichtet werden. Über `carla_ros_bridge` kann CARLA Simulator mit ROS Nodes kommunizieren.

Zuerst wird [Quellcode](https://github.com/carla-simulator/ros-bridge) heruntergeladen und in `catkin` Workspace kompiliert. Dann wird das Problem bei Ausführung in Python 3 Umgebung diskutiert.

# Setup
Vorbereitung: Variablen definieren, Ordner erstellen, Quellcode herunterladen. Hier bezeichnet man den Pfad zu `catkin` Workspace mit der Variable `CATKIN_WORKSPACE`, Quellcode werden im Ordner `$ROS_PYTHON3_CODE` verlegt. Um die Benutzerumgebung von der Systemumgebung zu trennen, verwendet man hier eine neue Python Umgebung: `$PYTHON3_VIRTUALENV`.

Hier wird angenommen, dass ein `catkin` Workspace schon vorher eingerichtet wird. Falls nicht, siehe [Create a ROS Workspace](https://wiki.ros.org/ROS/Tutorials/InstallingandConfiguringROSEnvironment)

```console
// activate python virtualenv
$ export PYTHON3_VIRTUALENV=$HOME/dev/PS_AUT/venv
$ source $PYTHON3_VIRTUALENV/bin/activate

// setup catkin workspace 
(venv) $ export CATKIN_WORKSPACE=$HOME/dev/PS_AUT/catkin_ws
(venv) $ source $CATKIN_WORKSPACE/devel/setup.bash
(venv) $ export ROS_PYTHON_VERSION=3
(venv) $ export ROS_PYTHON3_CODE=$CATKIN_WORKSPACE/../ros-python3-packages
```

Fasst man die obige Befehle in ein Skript zusammen, spart man viel Arbeit.
```console
// save commands to a script
$ cat <<'EOF' >> ros_env_setup.bash
#!/bin/bash
export PYTHON3_VIRTUALENV=$HOME/dev/PS_AUT/venv
source $PYTHON3_VIRTUALENV/bin/activate
export CATKIN_WORKSPACE=$HOME/dev/PS_AUT/catkin_ws
source $CATKIN_WORKSPACE/devel/setup.bash
export ROS_PYTHON_VERSION=3
export ROS_PYTHON3_CODE=$CATKIN_WORKSPACE/../ros-python3-packages
EOF
// next time only need to run this line for setup
$ . ros_env_setup.bash
```
## Install
Nun werden *dependencies* installiert. Danach kompiliert man das Package `carla_ros_bridge`.

```console
// download source code
(venv) $ mkdir -p $ROS_PYTHON3_CODE
(venv) $ git clone https://github.com/carla-simulator/ros-bridge.git $ROS_PYTHON3_CODE/ros-bridge
(venv) $ ln -s $ROS_PYTHON3_CODE/ros-bridge $CATKIN_WORKSPACE/src/

// install dependencies
(venv) $ cd $CATKIN_WORKSPACE
(venv) $ rosdep update
(venv) $ rosdep install --from-paths src --ignore-src -r

// compile carla_ros_bridge
(venv) $ catkin_make
```

## Test
Zum Testen ruft man den CARLA Simulator sowie ROS Server auf, dann startet `carla_ros_bridge`. Man öffnet drei Konsole und führt zuerst Befehle von §Setup aus, bevor man die folgende Befehle eintyppt.

### CARLA

```console
// run CARLA
(venv) $ export CARLA_PATH=$HOME/dev/PS_AUT/CARLA
(venv) $ $CARLA_PATH/CarlaUE4.sh -windowed -ResX=320 -ResY=240
```

### ROS Server

```console
// run ros
(venv) $ roscore
```

### carla_ros_bridge

```console
// run carla_ros_bridge
(venv) $ export PYTHONPATH=$PYTHONPATH:$CARLA_PATH/PythonAPI/carla/dist/carla-0.9.6-py3.5-linux-x86_64.egg
(venv) $ roslaunch carla_ros_bridge carla_ros_bridge.launch
```

Alternativ kann man das Package `carla` installieren statt `PYTHONPATH` jedes mal zu definieren.
```console
// install carla_ros_bridge
(venv) $ python -m easy_install $CARLA_PATH/PythonAPI/carla/dist/carla-0.9.6-py3.5-linux-x86_64.egg
Processing carla-0.9.6-py3.5-linux-x86_64.egg
creating /home/hjhee/dev/PS_AUT/venv/lib/python3.6/site-packages/carla-0.9.6-py3.5-linux-x86_64.egg
Extracting carla-0.9.6-py3.5-linux-x86_64.egg to /home/hjhee/dev/PS_AUT/venv/lib/python3.6/site-packages
Adding carla 0.9.6 to easy-install.pth file

Installed /home/hjhee/dev/PS_AUT/venv/lib/python3.6/site-packages/carla-0.9.6-py3.5-linux-x86_64.egg
Processing dependencies for carla==0.9.6
Searching for carla==0.9.6
Reading https://pypi.org/simple/carla/
No local packages or working download links found for carla==0.9.6
error: Could not find suitable distribution for Requirement.parse('carla==0.9.6')
```

Die Fehlermeldung scheint kein Problem zu sein bei Ausführung von `carla_ros_bridge`.

## Python 3
Hier wird verschiedene Probleme bei Ausführung von `carla_ros_bridge` in eine Python 3 Umgebung diskutiert. Es wird nur gezeigt, wie man die Probleme umgehen kann und keine Vollständigkeit garantiert.

### Tipps
Wenn Fehler bei Kompilieren auftritt, überprüfen, ob die Werte von Variablen (z.B. `ROS_PYTHON_VERSION`) sinnvoll sind. Bevor man neue Kompilierung startet, führt man den Befehl `rm -rf $CATKIN_WORKSPACE/build/*` durch, um die alte Konfiguration zu löschen.

### PyInit__tf2
```python
Traceback (most recent call last):
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/bridge.py", line 25, in <module>
    from carla_ros_bridge.actor import Actor
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/actor.py", line 19, in <module>
    import carla_ros_bridge.transforms as trans
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/transforms.py", line 16, in <module>
    import tf
  File "/opt/ros/melodic/lib/python2.7/dist-packages/tf/__init__.py", line 30, in <module>
    from tf2_ros import TransformException as Exception, ConnectivityException, LookupException, ExtrapolationException
  File "/opt/ros/melodic/lib/python2.7/dist-packages/tf2_ros/__init__.py", line 38, in <module>
    from tf2_py import *
  File "/opt/ros/melodic/lib/python2.7/dist-packages/tf2_py/__init__.py", line 38, in <module>
    from ._tf2 import *
ImportError: dynamic module does not define module export function (PyInit__tf2)
```

Diese Fehlermeldung tritt auf, wenn man den Befehl vom vorherigen Abschnitt ausführt. Wenn man die Fehlermeldung siehe, fällt auf, dass bei den Befehl `import tf` die Module `tf`, `tf2_ros`, `tf2_py` aus der Systemumgebung `/opt/ros/melodic/lib/python2.7/dist-packages` abgerufen werden. [randoms](https://github.com/ros/geometry2/issues/259#issuecomment-353061593) zeigte eine [Lösung](https://github.com/ros/geometry2/issues/259#issuecomment-353268956). Die Module `tf`, `tf2_ros`, `tf2_py` braucht man in `catkin` Workspace mit Python 3 neu kompilieren.

```console
(venv) $ git clone https://github.com/ros/geometry2.git $ROS_PYTHON3_CODE/geometry2
(venv) $ ln -s $ROS_PYTHON3_CODE/geometry2/{tf2,tf2_ros,tf2_py} $CATKIN_WORKSPACE/src/
(venv) $ cd $CATKIN_WORKSPACE
(venv) $ catkin_make
```

### 'dict' object has no attribute 'has_key'
```python
Exception in thread Thread-5:
Traceback (most recent call last):
  File "/usr/lib/python3.6/threading.py", line 916, in _bootstrap_inner
    self.run()
  File "/usr/lib/python3.6/threading.py", line 864, in run
    self._target(*self._args, **self._kwargs)
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/bridge.py", line 264, in _update_actors_thread
    self._update_actors(current_actors)
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/bridge.py", line 280, in _update_actors
    self._create_actor(carla_actor)
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/bridge.py", line 360, in _create_actor
    carla_actor, parent, self.comm, self._ego_vehicle_control_applied_callback)
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/ego_vehicle.py", line 51, in __init__
    prefix=carla_actor.attributes.get('role_name'))
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/vehicle.py", line 42, in __init__
    if carla_actor.attributes.has_key('object_type'):
AttributeError: 'dict' object has no attribute 'has_key'
```
Dieses Problem tritt auf, wenn man den Befehl ausführt.
```console
(venv) $ roslaunch carla_ros_bridge carla_ros_bridge_with_example_ego_vehicle.launch
```

Man sieht, dass es um den [Unterschied](https://stackoverflow.com/a/33727186) zwischen Python 2 und Python 3 handelt. Nach meiner Untersuchtung von dem Package `ros-bridge/carla_ros_bridge/src/carla_ros_bridge` ist sie die einzige Zeile, die die Attribute `has_key` noch benutzt. In anderen Dateien werden sie mit dem neuen Operator `in` ersetzt, wie z. B. [1](https://github.com/carla-simulator/ros-bridge/blob/1833ef3691202478159a221a21725f22c3ad34e6/carla_ros_bridge/src/carla_ros_bridge/communication.py#L86), [2](https://github.com/carla-simulator/ros-bridge/blob/1833ef3691202478159a221a21725f22c3ad34e6/carla_ros_bridge/src/carla_ros_bridge/bridge.py#L344).

Man ersetzt die Zeile 42 in `$CATKIN_WORKSPACE/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/vehicle.py`  mit `if 'object_type' in carla_actor.attributes:` und kompiliert:
```console
(venv) $ cd $CATKIN_WORKSPACE
(venv) $ sed -i -r "s/^(.*)\s(.*)\.has_key\((.*)\):/\1 \3 in \2:/g" $CATKIN_WORKSPACE/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/vehicle.py
(venv) $ catkin_make
```

### PyInit_cv_bridge_boost
```python
Traceback (most recent call last):
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/sensor.py", line 103, in _callback_sensor_data
    self.sensor_data_updated(carla_sensor_data)
  File "/home/hjhee/dev/PS_AUT/catkin_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/camera.py", line 103, in sensor_data_updated
    img_msg = Camera.cv_bridge.cv2_to_imgmsg(image_data_array, encoding=encoding)
  File "/opt/ros/melodic/lib/python2.7/dist-packages/cv_bridge/core.py", line 259, in cv2_to_imgmsg
    if self.cvtype_to_name[self.encoding_to_cvtype2(encoding)] != cv_type:
  File "/opt/ros/melodic/lib/python2.7/dist-packages/cv_bridge/core.py", line 91, in encoding_to_cvtype2
    from cv_bridge.boost.cv_bridge_boost import getCvType
ImportError: dynamic module does not define module export function (PyInit_cv_bridge_boost)
```

Dieses Problem tritt auf, wenn man wieder den Befehl ausführt.
```console
(venv) $ roslaunch carla_ros_bridge carla_ros_bridge_with_example_ego_vehicle.launch
```

Es geht wieder um Aufruf von Modul aus der Systemumgebung, man kompiliert das Package mit Python 3.
```console
(venv) $ git clone https://github.com/ros-perception/vision_opencv.git $ROS_PYTHON3_CODE/vision_opencv
(venv) $ ln -s $ROS_PYTHON3_CODE/vision_opencv/cv_bridge $CATKIN_WORKSPACE/src/
(venv) $ cd $CATKIN_WORKSPACE
(venv) $ catkin_make
```
