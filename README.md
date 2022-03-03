# Tutoriel d'utilisation du C++ pour Carla Simulator
## Lancement d'un Client C++
### Dépendances
* Installation UnrealEngine
* Installation Carla Simulator

Dans ce tuto nous verrons comment développer et utiliser un client C++ dans carla simulator.
- [Liens tutos d'installation/utilisation carla](https://carla.readthedocs.io/en/0.9.12/build_linux/)
- [Liens doc Doxygen Carla C++](carla.org/Doxygen/html/index.html)

Les tutos officiels sont majoritairement en *Python* (Carla supporte les deux languages), il faut donc
fréquemment consulter la *doc Doxygen* pour faire le parallèle entre les fonctions *Python* et *C++*

### Arborescence
(Res/arbo.png)
[image de l'arborescense](https://carla.readthedocs.io/en/0.9.13/img/pipeline.png)
Carla fonctionne avec un *Server* qui comprend la partie _UnrealEngine_ et un plugin de communication TCP avec *Carla*.
Et aussi un ou plusieurs *Client* (C++ ou Python) qui fonctionne uniquement avec du code *Carla*. 

#### Fichiers

Lors de l'installation carla, plusieurs dossiers importants ont été créé :

| Dossiers | Descriptions |
| ------ | ------ |
| *carla/Unreal/CarlaUE4* | Contient les fichiers plugins qui vont être intégrer à Unreal |
| *carla/LibCarla* | Contient les codes sources pour Carla coté client |
| *carla/PythonAPI* | Contient les codes sources pour Carla en Python ainsi que des exemples de scripts |
| *carla/Examples/CppClient* | Contient un exemple de Client en C++ |

### Lancement

Commencer par lancer le *Server* qui lancera lui même *UnrealEngine* :
```sh
cd carla
make launch
```
Si le *Server* a besoin d'être recompilé (1ère fois ou modification des codes sources liés à celui-ci) :
```sh
make clean
make PythonAPI
make launch
```
Dans un nouveau terminal, lancer un *Client*, ici _C++_ : 
```sh
cd carla/Examples/CppClient
make run
```
Si ce *Client* a besoin d'être recompilé :
```sh
cd rm bin/*
make clean
make build
```
Si nécessaire, lancer un nouveau terminal pour exécuter un client supplémentaire *C++* ou *Python*,
Pour lancer un client *Python* (Remplacer python3 par python pour une version 2.7) :
```sh
cd carla/PythonAPI/examples
python3 tutorial.py
```

## Modification d'un Client C++
### Explication du code de l'exemple
Avant tout, on utilise des namespaces particuliers déclarés par carla : 
```cpp
namespace cc = carla::client;
namespace cg = carla::geom;
namespace csd = carla::sensor::data;
```

On déclare le *Client* qui va nous permettre d'accéder au *Server* dans la suite du code :
```cpp
auto client = cc::Client(host, port);
client.SetTimeout(120s);
```
On charge ensuite une carte présente dans le chemin **carla/Unreal/CarlaUE4/Content/Carla/Maps/**, les cartes nommées avec
le suffixe _Opt signifie que la carte a été divisé en plusieurs parties qui peuvent être activées ou désactivées pour
amélorer les performances :
```cpp
auto world = client.LoadWorld("Town01_Opt", true, carla::rpc::MapLayer::None);
std::cout << "Loading world: Town01_Opt" << std::endl;
```
**carla::rpc::MapLayer::None** indique qu'on active aucune des couches de la carte, cependant les routes et les signalisations ne peuvent pas être désactivées. 

Les objets (_Actors_, _Sensors_, _Vehicle_, etc.) qui peuvent être ajouter dans la simulation sont stockés sous forme de **Blueprints**, ils fonctionnent commes des modèles et sont liés à des fichiers déclarés dans carla et unreal.
Pendant la phase de compilation de carla, les Blueprints sont détectés dans les codes sources et sont listés dans la **BlueprintLibrary**. 
On récupère la librairie à partir de la carte, on peut ensuite la parcourir pour rechercher un blueprint précis ou par catégorie :
```cpp
auto blueprint_library = world.GetBlueprintLibrary();

//Filtre uniquement les véhicules dans la librairie puis en choisi un aléatoirement
auto vehicles = blueprint_library->Filter("vehicle");
auto blueprint = RandomChoice(*vehicles, rng);

//Récupère directement un blueprint précis 
auto boxpb = blueprint_library->Find("static.prop.box02");
```

Pour afficher la liste des blueprints présent dans la librairie de la carte actuelle, on peut afficher leurs IDs : 
```cpp
auto allbp = blueprint_library->Filter("*");
for( auto bp : *allbp ) {
    std::cout << bp.GetId() << "\n";
}
```

Pour faire apparaitre un objet dans le monde, il nous faut le *Blueprint* associé mais aussi une position précise.
Pour cela on récupère les postions de spawn de la map, il s'agit d'endroits prédefinies ou peuvent apparaître des véhicules :
```cpp
// Find a valid spawn point.
auto map = world.GetMap();
auto transform = RandomChoice(map->GetRecommendedSpawnPoints(), rng);
```

Enfin, la méthode *SpawnActor* permet de lier le blueprint et la position pour faire apparaitre l'objet dans le monde en tant qu'*Actor*, il faut ensuite le caster pour récupérer la référence sous forme de *Vehicle* (il hérite de *Actor*) :
```cpp
// Spawn the vehicle.
auto actor = world.SpawnActor(blueprint, transform);
std::cout << "Spawned " << actor->GetDisplayId() << '\n';
auto vehicle = boost::static_pointer_cast<cc::Vehicle>(actor);
```

La référence spécifique de *Vehicle* plutôt que *Actor* permet d'accéder à de nouvelle fonctions et attributs de classes, comme par exemple, le contrôle du véhicule : 
```cpp
// Apply control to vehicle.
cc::Vehicle::Control control;
control.throttle = 1.0f;
control.steer = 1.0f;
vehicle->ApplyControl(control);
```

L'objet qui contrôle la position "*Transform*" peut ensuite être modifié pour controlé la rotation, ici on utilise le lieux de spawn de véhicule pour référence, puis on s'en éloigne un peu afin déplacer le spéctateur qui regarde la scène : 
```cpp
auto spectator = world.GetSpectator();
transform.location += 32.0f * transform.GetForwardVector();
transform.location.z += 2.0f;
transform.rotation.yaw += 180.0f;
transform.rotation.pitch = -15.0f;
spectator->SetTransform(transform);
```

Une fois lancé, le client doit être mis en pause pour laissé le temps au véhicules d'avancer, à la fin de son exécution, le client doit détruire les objets utilisés pour ne pas causé de problème avec le C++ : 
```cpp
// Pause the client for 10 seconds
std::this_thread::sleep_for(10s);

// Remove actors from the simulation.
actor->Destroy();
vehicle->Destroy();
```

### Ajout d'un Sensor **Caméra de segmentation**
Les *Sensor* permettent de détecter des évènements, comme les véhicules, ils héritent de *Actor*.

On commence par trouver le blueprint de la caméra (il existe d'autres type de *Sensors*) : 
```cpp
// Find a camera blueprint.
auto camera_bp = blueprint_library->Find("sensor.camera.semantic_segmentation");
```

On créé un nouveau objet *Transform* pour définir la position et rotation de la caméra, comme on attache ici la caméra à un véhicule, l'origine des coordonnées sera partagée avec le véhicule attaché. *camera_transform* est donc relative :
```cpp
// Spawn a camera attached to the vehicle.
auto camera_transform = cg::Transform{
    cg::Location{-5.5f, 0.0f, 2.8f},   // x, y, z.
    cg::Rotation{-15.0f, 0.0f, 0.0f}}; // pitch, yaw, roll.
auto cam_actor = world.SpawnActor(*camera_bp, camera_transform, actor.get());
auto camera = boost::static_pointer_cast<cc::Sensor>(cam_actor);
```

Après avoir récupéré la référence caméra, on lui attache un callback qui renverra l'image detecté : 
```cpp
// Register a callback to save images to disk.
camera->Listen([](auto data) {
    auto image = boost::static_pointer_cast<csd::Image>(data);
    EXPECT_TRUE(image != nullptr);
    SaveSemSegImageToDisk(*image);
});
```

On sauvegarde les images dans le dossier **_images/** dans le chemin de main.c avec une palette de couleurs particulières :
```cpp
/// Save a semantic segmentation image to disk converting to CityScapes palette.
static void SaveSemSegImageToDisk(const csd::Image &image) {
  using namespace carla::image;

  char buffer[9u];
  std::snprintf(buffer, sizeof(buffer), "%08zu", image.GetFrame());
  auto filename = "_images/"s + buffer + ".png";

  auto view = ImageView::MakeColorConvertedView(
      ImageView::MakeView(image),
      ColorConverter::CityScapesPalette());
  ImageIO::WriteView(filename, view);
}
```

Il ne faut pas oublier de détruire la caméra avant de quitter le programme : 
```cpp
camera->Destroy();
```

Résultat des caméras : 
(Res/velo.gif)
[gif : vélo](https://im4.ezgif.com/tmp/ezgif-4-6e471d157b.gif)
(Res/scooter.gif)
[gif : scooter](https://im4.ezgif.com/tmp/ezgif-4-c76ebd4528.gif)
