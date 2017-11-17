
**This demo was converted from [threejs](https://threejs.org/) version, you could check the [threejs version here](https://threejs.org/examples/#canvas_geometry_birds)**

[![](https://raw.githubusercontent.com/suntabu/cpu_birds/master/swords.gif)](https://www.youtube.com/watch?v=aFMCJBGomC8)

##### 1. You could check [this unity demo on github](https://github.com/suntabu/cpu_birds).


##### 2. About the code:

The Boid class struct:

- Fields of boid: 

``` C#
// fields
Vector3 _acceleration;
float _width = 500;
float _height = 500;
float _depth = 200;
Vector3 _goal;
float _neighborhoodRadius = 100;
float _maxSpeed = 4;
float _maxSteerForce = 0.1f;
bool _avoidWalls = false;
Vector3 mVelocity;
Vector3 mPosition;
Vector3 mAcceleration;
```
- Basic actions of boid, these methods defined the actions of a boid in the flock. Also you could define your own methods for simulating other flock actions, just like **attacking** which is what I would do next. 

``` C#
public Vector3 avoid (Vector3 target)

public void repulse (Boid target)

public Vector3 reach (Vector3 target, float amount)

public Vector3 alignment (Boid[] boids)

public Vector3 cohesion (Boid[] boids)

public Vector3 separation (Boid[] boids)

public void move ()

public void flock (Boid[] boids)
 
```

- Other methods

``` C#
public void setRandomPosition (int width)

public void setRandomVelocity ()

public void setGoal (Vector3 target)

public void setAvoidWalls (bool avoid)

public void setWorldSize (float width, float height, float depth)

public void run (Boid[] boids)
```


##### 3. TODO:
- predator
- attacker
- blow by wind etc.


##### 4. About Frost Mourne:

I extracted these models from my world of warcraft client by [WOWModelViewer](https://wowmodelviewer.net/wordpress/) for studying purpose. you could find more by contacting me via email.

