# TP WebGl - Creation d'une table de billard


#### Laura Pascal 
#### Jean-baptiste Vimort 


## Introduction
Lors de ce projet nous allons mettre en place une simulation d'un jeu de billard en utilisant la technique webGl. Notre billard aura les spécificités suivantes:
- La table sera constituée d'un tapis (plan texturé), de 4 bandes (rectangles) ainsi que de 4 trous (représentés par des cylindres) placés dans chaque angle de la table
- Une boule blanche qui pourra être "frappée" par l'utilisateur grâce au click gauche de la souris
- Six boules de couleur qui disparaîtront aux contacts des trous
- La boule blanche ne disparaîtra pas lorsqu'elle sera en contact d'un trou afin de ne pas terminer le jeu

Pour ce faire nous avons choisi d'utiliser la bibliothèque Javascrip Babylon js qui est entièrement destinée à l'affichage de scènes LS pour les raisons suivantes:
- Elle permet de gérer les caméras et lumières d'une scène simplement 
- La création d'objets élémentaires tels que des sphères ou des plans est intuitive
- Son moteur physique est très complet, et permet de gérer les collisions simplement

## 0. Gestion de la caméra
Nous avons implementé la caméra de telle sorte qu'elle puisse bougée à l'aide de la souris: 
```
	camera.attachControl(canvas, true);
```

De plus, la caméra ne peut pas passer à travers la table de billard grâce au définition des paramètres _upperBetaLimit_ et _lowerRadiusLimit_ : 
```
	camera.upperBetaLimit = (Math.PI / 2) * 0.99; 
	camera.lowerRadiusLimit = 20;
```


## 1. Modélisation Géométrique de la table de billard
Afin de modéliser notre table de billard, nous avons utilisé plusieurs éléments basiques: 
- Le tapis a été modélisé grâce à un _ground_ d'une taille x=20 et z=40, qui a été positionné aux coordonnées (0,0,20) 
```
	var ground = BABYLON.Mesh.CreateGround("ground1", 20, 40, 2, scene); //BABYLON.Mesh.CreateGround(name, width, depth, subdivs, scene)
```

Une texture a été appliquée au tapis grâce au code suivant:
```
	var materialGround = new BABYLON.StandardMaterial("texturePlane", scene);
	materialGround.diffuseTexture = new BABYLON.Texture("data/grass.jpg", scene);
	ground.material = materialGround;
```

- les bandes ont été modélisées grâce à 4 _boxes_ marron d'une taille initiale 1. Une translation et un scaling spécifiques sont appliqués à chacun d'eux pour être positionnés au bord de notre tapis.
```
	box[i] = BABYLON.Mesh.CreateBox("Box", 1.0, scene); // BABYLON.Mesh.CreateBox(name, size, scene);
```

La couleur marron des boxes a été créé grâce au code suivant:
```
	var materialBoundaries = new BABYLON.StandardMaterial("textureBoundaries", scene);  //creation of a mateial for all the boundaries
	materialBoundaries.diffuseColor = new BABYLON.Color3(0.2, 0.1, 0); //setting of the brown color
	box[i].material = materialBoundaries;
```

- Les trous on été modélisés grâce à 4 _cylinder_ noires composés d'une base de rayon 2 et d'une hauteur égale à 1. Une translation spécifique est appliquée à chaque cylindre afin qu'ils soient placés aux 4 coins de notre tapis.
```
	cylinder[i] = BABYLON.Mesh.CreateCylinder("cylinder", 1, 2, 2, 60, 1, scene); // BABYLON.Mesh.CreateCylinder(name, height, diameterTop, diameterBottom, tessellation, subdivisions, scene)
```

- Les boules on été modélisées grâce à un tableau de taille 7 de _spheres_ de rayon 1. La boule blanche (_sphere[6]_) est placée initialement aux coordonnées (0,0.5,10). Les boules colorées (_sphere[i] avec i = {0,1...5}_) sont placés en forme de triangle en face de la boule blanche. 
```
	sphere[i] = BABYLON.Mesh.CreateSphere("sphere", 16, 1, scene); // BABYLON.Mesh.CreateSphere(name, segments, diameter, scene)
```

## 2. Collisions entre les boules 
Afin de gérer les différentes collisions (collisions des balles entre elles et collisions des balles avec la table de billard), on utilise le moteur physique de babylonjs. Pour se faire on commence par activer le moteur physique dans notre scène grâce à la fonction suivante:
```
scene.enablePhysics();
```
On assignera ensuite des propriétés physiques à chaque objet de la manière suivante:
```
var body = sphere[i].setPhysicsState(BABYLON.PhysicsEngine.SphereImpostor, { mass: 1, restitution: 1 });
```
On allouera une masse de 1 à toutes les boules et une masse de 0 aux objets constituants la table de billard. De cette manière la table va rester immobile à sa position initiale, et les boules vont être attirées, grâce à leur masse, dans la direction des z négatifs. Cela va parfaitement mimer la réalité et ainsi permettre aux billes de rouler sur la table. De plus la restitution du sol sera allouée à 0 et la restitution des bandes ainsi que celle des boules vont être allouée à 1, permettant ainsi aux objets de rebondir les uns contre les autres sans perte de vitesse.
Dans la configuration actuelle une fois la bille blanche lancée, les boules vont s'animer en fonction des différentes collisions sans jamais perdre d'énergie, elles vont donc rouler pendant un temps infinie sans jamais ralentir. Afin de régler ce problème, nous allons simuler la perte d'énergie (et le ralentissement) dû au frottement entre les boules et le tapis en ajoutant un /linear damping/ à toutes les boules de la manière suivante:
```
body.linearDamping = 0.2;
```
Cela va permettre aux boules de s'arrêter plus ou moins rapidement en fonction de la valeur de ce /linear damping/.

## 3. Animation de la boule blanche
L'interface avec l'utilisateur se fera grâce aux cliques gauche de la souris, on utilisera pour cela la fonction suivante:
```
scene.onPointerDown = function (evt, pickResult) 
```
De plus, au billard seul la boule blanche peut être tapée par le joueur, l'utilisateur pourra donc interagir uniquement avec la boule blanche (qui est notre 7e boule dans notre code):
```
if (pickResult.hit && pickResult.pickedMesh == sphere[6])
```
Lorsque l'utilisateur clique sur cette boule, une vitesse lui est appliquée. La direction de la vitesse va dépendre de l'endroit où l'utilisateur aura cliqué, elle sera opposée à la position du clique sur la boule:
```
sphere[6].physicsImpostor.setLinearVelocity(new BABYLON.Vector3(-(pickResult.pickedPoint.x - sphere[6].position.x) * 40,
                                                                0,
                                                                -(pickResult.pickedPoint.z - sphere[6].position.z)*40));
```


## 4. Intersection et disparition des boules colorées
#### Objectif:
Modélisation de la disparition d'une boule colorée dans un trou.
#### Implémentation:
Afin de faire disparaître nos boules colorées dans les trous du billard nous avons attaché un _BABYLON.ActionManager_ à nos différentes sphères. Lorsqu'il y a une intersection entre une sphère (= boule colorée) et un cylindre (= trou), le rayon de la sphère sera réduit à 0. Cette action est réalisable en appliquant la fonction _registerAction_ à notre _BABYLON.ActionManager_ pour chacune de nos sphères avec chaque cylindre et en précisant qu'une action scaling de paramètre (0,0,0) doit être appliquée à notre sphère: 
```
	BABYLON.SetValueAction( triggerOptions, action's target, action, value);
```

Afin que cette action soit déclenchée, on utilisera un signal déclencheur _trigger_ qui sera appliqué lorsqu'il y aura une intersection _BABYLON.ActionManager.OnIntersectionEnterTrigger_ entre la sphère le cylindre _parameter: cylinder[j]_. On réalise la même action quand la sphère n'est plus en contact avec le cylindre. 
```
	// i = {0,1...5} indice des sphère
	// j = {0,1,2,3} indice des cylindres

	sphere[i].actionManager.registerAction(new BABYLON.SetValueAction(
	{ trigger: BABYLON.ActionManager.OnIntersectionEnterTrigger, parameter: cylinder[j] },
	sphere[i], "scaling", new BABYLON.Vector3(0, 0, 0)));
	sphere[i].actionManager.registerAction(new BABYLON.SetValueAction(
	{ trigger: BABYLON.ActionManager.OnIntersectionExitTrigger, parameter: cylinder[j] },
	sphere[i], "scaling", new BABYLON.Vector3(0, 0, 0)));
```

## Conclusion
L'utilisation du WebGL et de babylon.js nous a permis de mettre en place simplement une simulation relativement réaliste d'un jeu de billard. De plus l'un des avantages majeurs de ce code est sa portabilité, il est de nos jours facilement interpretable par la plupart des navigateurs web indépendamment du système d'exploitation de la machine. 
