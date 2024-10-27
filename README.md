# SIEX 24/25. Ejemplo de sistema CBR (Razonamiento Basado en Casos)



[TOC]

Ejemplo de una herramienta de tasación de coches usados a partir de sus características (marca y modelo, color, millas, etc) utilizando razonamiento basado en casos.

- Emplea un "mimiframework" que estructura las fases del ciclo CBR (recuperar, reutilizar, revisar, retener)

- Hace uso de la [librería CBRKit](https://github.com/wi2trier/cbrkit/)  para (1) la carga y gestión de la base de casos y para (2) la implementación de las métricas de similaridad utilizadas en la fase`recuperar`



## 1. Dependencias y ejecución del proyecto

### 1.1 Dependencias

CBRKit es un proyecto en desarrollo y la librería está en cambio constante. Para este ejemplo y para la realización de la práctica de CBR se ha trabajará con la **versión 0.14.2** (la última compatible con Python 3.11, [código fuente descargable](https://github.com/wi2trier/cbrkit/archive/refs/tags/v0.14.2.zip))

```sh
$ pip install -r requirements.txt
```

ó

```sh
$ pip install cbrkit[nlp]==0.14.2
$ pip install argparse
```

**Nota:** Para utilizar las métricas de similaridad entre Strings basadas en modelos de lenguaje es necesario instalar módulos adicionales de `cbrkit`
- `cbrkit[transformers]` para utilizar los modelos de la librería [`sentence-transformers`](https://sbert.net/) 
- `cbrkit[llm]` para utilizar  las APIs de OpenIA y similares


### 1.2 Ejecución
#### Descarga desde github

- Clonado del proyecto con el comando `git`
    ```sh
    git clone https://github.com/esei-si-dagss/tasador-24.git
    cd tasador-24
    ```
- Descarga directa del proyecto comprimido en ZIP (desde (https://github.com/esei-si-dagss/tasador-24/archive/refs/heads/main.zip))
    ```sh
    wget https://github.com/esei-si-dagss/tasador-24/archive/refs/heads/main.zip
    unzip main.zip
    cd tasador-24-main
    ```

#### Ejemplo de ejecución
- Parámetros de línea de comandos
    ```sh
    python main.py --help
    usage: main.py [-h] [-b BASE_CASOS] [-r CASOS] [-c COLORES] [-f FABRICANTES] [-n NUM_SIMILARES] [-u UMBRAL_PRECIO] [-d]
    
    Ejemplo de uso del modelo CBR para tasaciones de coches
    
    options:
      -h, --help            show this help message and exit
      -b BASE_CASOS, --base_casos BASE_CASOS
                            Path al fichero JSON con la base de casos (lista de objetos coche)
      -r CASOS, --casos CASOS
                            Path al fichero JSON con los casos a resolver (lista de objetos coche)
      -c COLORES, --colores COLORES
                            Path al fichero YAML con la taxonomia de colores
      -f FABRICANTES, --fabricantes FABRICANTES
                            Path al fichero YAML con la taxonomia de fabricantes
      -n NUM_SIMILARES, --num_similares NUM_SIMILARES
                            Num. de casos similares a utilizar
      -u UMBRAL_PRECIO, --umbral_precio UMBRAL_PRECIO
                            porcentaje de error en el precio predicho admitido como correcto en la fase Revision
      -d, --debug           Activar DEBUG de fases del ciclo CBR
    ```
- Procesamiento del fichero de casos `datos/casos_a_resolver.json` (con DEBUG activado)
    ```sh
    python main.py --base_casos datos/base_de_casos.json \
                   --casos datos/casos_a_resolver.json \
                   --colores datos/paint_color.yaml \
                   --fabricantes datos/cars-taxonomy.yaml \
                   --num_similares 10 --umbral_precio 5 --debug
    ```
    Carga la base de casos de ejemplo y procesa los 20 casos (coches) del fichero `datos/casos_a_resolver.json`. 
    - Para definir la similaridad entre casos se cargan los dos ficheros con taxonomías (colores y fabricantes) disponibles
    - En la fase `reutilizar` se usan 10 casos similares para predecir el precio del coche descrito en el caso a resolver
    - En la fase `revisar` se considera que la predicción fue exitosa si la diferencia entre el precio predico y el precio real es menor al 5%


### 1.3 Estructura del proyecto

```sh
$ tree tasador-24
datos
├── cars-1k.original.json
├── base_de_casos.json
├── casos_a_resolver.json
├── cars-taxonomy.yaml
└── paint_color.yaml
core.py 
tasador.py 
main.py 
dividir_base_casos_json.py 
README.md
```
El **directorio `datos` **contiene ficheros de ejemplo para la ejecución del "tasador" de vehículos:

- `cars-1k.original.json`: base casos empleada en los ejemplos y la demo de CBRKit ([enlace](https://github.com/wi2trier/cbrkit-demo)) [almacena los casos como un array de objetos JSON que será cargado con [`cbrkit.loaders.json()`](https://wi2trier.github.io/cbrkit/cbrkit/loaders.html)]
  - cada **caso** es un objeto JSON con la siguiente estructura:

    ```json
    {
    "engine": {
      "drive": "rwd",
      "fuel": "gas",
      "transmission": "manual"
    },
    "miles": 150000,
    "model": {
      "make": "xsara",
      "manufacturer": "citroen"
    },
    "paint_color": "black",
    "price": 1295,
    "title_status": "clean",
    "type": "compact",
    "year": 1360
    }
    ```
  
- `base_casos.json`: porción de la base de casos`cars-1k.original.json` excluyendo los casos de `casos_a_resolver.json`

- `casos_a_resolver.json`: 20 casos tomados de `cars-1k.original.json` para ser usados como entrada al tasador

- `cars-taxonomy.yaml`: taxonomía de fabricantes de vehículos, utilizada para calcular la similaridad entre fabricantes [en formato YAML para ser cargado con [`cbrkit.sim.strings.taxonomy.load()`](https://wi2trier.github.io/cbrkit/cbrkit/sim/strings/taxonomy.html)]

- `paint-colot.yaml`: taxonomía de colores, utilizada para calcular la similaridad entre colores de pintura

El **fichero `core.py`** define un "miniframework" que encapsula las fases del ciclo básico de los sistemas CBR en la clase `CBR`.  

- La clase `CBR` es el equivalente a una "clase abstracta" de Java que implementa el método `ciclo_cbr()` delegando las fases del ciclo CBR a métodos "abstractos" `recuperar()`,  `reutilizar()`, `revisar()` y `retener()` que tendrán que ser implementados por las subclases concretas.
- Incluye  también una clase de utilidad, `CBR_DEBUG`, que implementa la traza por pantalla de las fases CBR

El **fichero `tasador.py`** incluye la implementación del tasador de vehículos en una clase `TasadorCBR` 

- Es una clase que hereda de `core.CBR` y proporciona implementaciones de las fases del ciclo CBR utilizando parte de las funcionalidades de CBRKit

El **fichero `main.py`** ofrece un ejemplo de uso de la clase `TasadorCBR`: carga la base de casos, crea y configura una instancia de `TasadorCBR` y la alimenta con los casos a resolver (coches a tasar)

El **fichero `*dividir_base_casos_json.py` * es un script de utilidad que a partir de una base de casos almacenada como array JSON extrae una serie de casos a resolver al azar y crea otra base de casos sin incluirlos.


## 2. Uso de CBRkit

CBRkit es un toolkit modular en Python para el Razonamiento Basado en Casos. Actualmente se encuentra en desarrollo y en su versión actual permite cargar casos, definir medidas de similitud y realizar la recuperación sobre bases de casos en memoria. 

Más detalles: 

- [Codigo y descarga](https://github.com/wi2trier/cbrkit/) (descarga de la [versión 0.14.2](https://github.com/wi2trier/cbrkit/archive/refs/tags/v0.14.2.zip)))
- [Documentación](https://wi2trier.github.io/cbrkit/cbrkit.html)
- [Paper](https://www.mirkolenz.com/static/ca607f149265ea90aea9579bd78a04bc/Lenz2024CBRkitIntuitiveCaseBased.pdf) y [video](https://www.youtube.com/watch?v=27dG4MagDhE) de presentación

### 2.1 Resumen CBRkit

- [Resumen CBRkit (v. 0.14.2)](doc/resumen_cbrkit.md) [[pdf](doc/resument_cbrkit.pdf)]



## 3. Uso del "miniframework" CBR

### 3.1 Módulo `core.py`


<img src="doc/ciclo_cbr.png" alt="ciclo_cbr" style="zoom:80%;" />  


### 3.2 Módulo `tasador.py`



<img src="doc/clases.png" alt="ciclo_cbr" style="zoom:80%;" />



