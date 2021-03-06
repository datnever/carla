# 1st. World and client

The client and the world are two of the fundamentals of CARLA, a necessary abstraction to operate the simulation and its actors.  

This tutorial goes from defining the basics and creation of these elements, to describing their possibilities. If any doubt or issue arises during the reading, the [CARLA forum](https://forum.carla.org/) is there to solve them.  

* [__The client__](#the-client)
	* Client creation
	* World connection 
	* Other client utilities
* [__The world__](#the-world)
	* World life cycle   
	* Get() from the world
	* Weather 
	* World snapshots 
	* Settings  

---
## The client

Clients are one of the main elements in the CARLA architecture. They connect to the server, retrieve information, and command changes. That is done via scripts. The client identifies itself, and connects to the world to then operate with the simulation.  

Besides that, clients are able to access other CARLA modules, features, and apply command batches. Only command batches will be covered in this section. These are useful for basic things such as spawning lots of actors. The rest of features are more complex, and will be dealt with in other sections.  

Take a look at [__carla.Client__](python_api.md#carla.Client) in the Python API reference to learn on specific methods and variables of the class. 


### Client creation

Two things are needed. The __IP__ address identifying it, and __two TCP ports__ to communicate with the server. An optional third parameter sets the amount of working threads. By default this is set to all (`0`). [This code recipe](ref_code_recipes.md#parse-client-creation-arguments) shows how to parse these as arguments when running the script. 

```py
client = carla.Client('localhost', 2000)
```
By default, CARLA uses local host IP, and port 2000 to connect but these can be changed at will. The second port will always be `n+1`, 2001 in this case.  

Once the client is created, set its __time-out__. This limits all networking operations so that these don't block the client forever. An error will be returned if connection fails. 

```py
client.set_timeout(10.0) # seconds
```

It is possible to have many clients connected, as it is common to have more than one script running at a time. Working in a multiclient scheme with advanced CARLA features, such as the traffic manager, is bound to make communication more complex.  

!!! Note
    Client and server have different `libcarla` modules. If the versions differ, issues may arise. This can be checked using the `get_client_version()` and `get_server_version()` methods. 

### World connection

A client can connect and retrieve the current world fairly easily. 

```py
world = client.get_world()


The client can also get a list of available maps to change the current one. This will destroy the current world and create a new one.
```py
print(client.get_available_maps())
...
world = client.load_world('Town01')
# client.reload_world() creates a new instance of the world with the same map. 
```

Every world object has an `id` or episode. Everytime the client calls for `load_world()` or `reload_world()` the previous one is destroyed. A new one is created from scratch with a new episode. Unreal Engine is not rebooted in the process. 

### Other client utilities

The main purpose of the client object is to get or change the world. Usually it is no longer used after that. However, the client also applies command batches and accesses advanced features. These features are.  

* __Traffic manager.__ This module is in charge of every vehicle set to autopilot to recreate urban traffic. 
* __[Recorder](adv_recorder.md).__ Allows to reenact a previous simulation. Uses [snapshots](core_world.md#world-snapshots) summarizing the simulation state per frame. 

As for command batches, the latest sections in the Python API describe the [available commands](python_api.md#command.ApplyAngularVelocity). These are common functions prepared to be executed in lots during the same step of the simulation. They can, for instance, destroy all the vehicles contained in `vehicles_list` at once.  
```py
client.apply_batch([carla.command.DestroyActor(x) for x in vehicles_list])
```

`apply_batch_sync()` is only available in [synchronous mode](adv_synchrony_timestep.md). It returns a [command.Response](python_api.md#command.Response) per command applied.

---
## The world

The major ruler of the simulation. Its instance should be retrieved by the client. It does not contain the model of the world itself, that is part of the [Map](core_map.md) class. Instead, most of the information, and general settings can be accessed from this class.

* Actors in the simulation and the spectator. 
* Blueprint library. 
* Map. 
* Simulation settings. 
* Snapshots. 

Some of its most important methods are _getters_. Take a look at [carla.World](python_api.md#carla.World) to learn more about it. 

### Actors

The world has different methods related with actors that allow for different functionalities.  

* Spawn actors (but not destroy them). 
* Get every actor on scene, or find one in particular.  
* Access the blueprint library.  
* Access the spectator actor, the simulation's point of view.  
* Retrieve a random location that is fitting to spawn an actor.  

Spawning will be explained in [2nd. Actors and blueprints](core_actors.md). It requires some understanding on the blueprint library, attributes, etc.  

### Weather

The weather is not a class on its own, but a set of parameters accessible from the world. The parametrization includes sun orientation, cloudiness, wind, fog, and much more. The helper class [carla.WeatherParameters](python_api.md#carla.WeatherParameters) is used to define a custom weather.  
```py
weather = carla.WeatherParameters(
    cloudiness=80.0,
    precipitation=30.0,
    sun_altitude_angle=70.0)

world.set_weather(weather)

print(world.get_weather())
```

There are some weather presets that can be directly applied to the world. These are listed in [carla.WeatherParameters](python_api.md#carla.WeatherParameters) and accessible as an enum.  

```py
world.set_weather(carla.WeatherParameters.WetCloudySunset)
```

!!! Note
    Changes in the weather do not affect physics. They are only visuals that can be captured by the camera sensors. 

### Debugging

World objects have a [carla.DebugHelper](python_api.md#carla.DebugHelper) object as a public attribute. It allows for different shapes to be drawn during the simulation. These are used to  trace the events happening. The following example would draw a red box at an actor's location and rotation. 

```py
debug = world.debug
debug.draw_box(carla.BoundingBox(actor_snapshot.get_transform().location,carla.Vector3D(0.5,0.5,2)),actor_snapshot.get_transform().rotation, 0.05, carla.Color(255,0,0,0),0)
```

This example is extended in a [code recipe](ref_code_recipes.md#debug-bounding-box-recipe) to draw boxes for every actor in a world snapshot. 

### World snapshots

Contains the state of every actor in the simulation at a single frame. A sort of still image of the world with a time reference. The information comes from the same simulation step, even in asynchronous mode.  

```py
# Retrieve a snapshot of the world at current frame.
world_snapshot = world.get_snapshot()
```

A [carla.WorldSnapshot](python_api.md#carla.WorldSnapshot) contains a [carla.Timestamp](python_api.md#carla.Timestamp) and a list of [carla.ActorSnapshot](python_api.md#carla.ActorSnapshot). Actor snapshots can be searched using the `id` of an actor. A snapshot lists the `id` of the actors appearing in it.  

```py
timestamp = world_snapshot.timestamp # Get the time reference 

for actor_snapshot in world_snapshot: # Get the actor and the snapshot information
    actual_actor = world.get_actor(actor_snapshot.id)
    actor_snapshot.get_transform()
    actor_snapshot.get_velocity()
    actor_snapshot.get_angular_velocity()
    actor_snapshot.get_acceleration()  

actor_snapshot = world_snapshot.find(actual_actor.id) # Get an actor's snapshot
```

### World settings

The world has access to some advanced configurations for the simulation. These determine rendering conditions, simulation time-steps, and synchrony between clients and server. They are accessible from the helper class [carla.WorldSettings](python_api.md#carla.WorldSettings).  

For the time being, default CARLA runs with the best graphics quality, a variable time-step, and asynchronously. To dive further in this matters take a look at the __Advanced steps__ section. The pages on [synchrony and time-step](adv_synchrony_timestep.md), and [rendering options](adv_rendering_options.md) could be a great starting point.

---
That is a wrap on the world and client objects. The next step takes a closer look into actors and blueprints to give life to the simulation.  

Keep reading to learn more. Visit the forum to post any doubts or suggestions that have come to mind during this reading.  

<div text-align: center>
<div class="build-buttons">
<p>
<a href="https://forum.carla.org/" target="_blank" class="btn btn-neutral" title="CARLA forum">
CARLA forum</a>
</p>
</div>
<div class="build-buttons">
<p>
<a href="../core_actors" target="_blank" class="btn btn-neutral" title="2nd. Actors and blueprints">
2nd. Actors and blueprints</a>
</p>
</div>
</div>