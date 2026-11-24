# Procesamiento de imágenes por lotes

Motor de filtros en C++ con OpenMP, un trabajador que lo expone por HTTP y un
despachador que reparte el lote entre varios trabajadores.

## Integrantes

| Nombre completo | Código | Correo institucional |
|---|---|---|
|Yilmar Mauricio Riascos Moreno | 202459662 | yilmar.riascos@correounivalle.edu.co |
| Juan esteban perez ramirez | 202459486 | juan.perez.ramirez@correounivalle.edu.co |
|Marlen Lizeth Estrada López | 202359494 | marlen.estrada@correounivalle.edu.co|
| Scarllys del Valle Vallecilla Caicedo |202459608 | scarllys.vallecilla@correounivalle.edu.co|

El informe está en [`docs/`](docs/).


# Reparto de trabajo del proyecto

Cuatro personas, cuatro frentes de trabajo con carga comparable. Cada persona es duena de un conjunto de codigo, de un conjunto de mediciones y de una parte del informe. Nadie depende de otro para avanzar en la mayor parte del tiempo, y todos terminan con commits significativos en main.

El indice docs/README.md y la tabla criterio -> archivo se arman entre los cuatro en una sesion corta al inicio y se revisan al final. No se le asignan a una sola persona porque es el documento que amarra los cuatro archivos del informe.

---

## Persona A - Filtros de stencil: gaussiano separable y Sobel
# Perez

### Que implementa

- El desenfoque gaussiano separable secuencial: dos pasadas, la horizontal y luego la vertical, con el nucleo binomial 1/16 [1 4 6 4 1]. La imagen intermedia T se mantiene en double para evitar el redondeo intermedio.
- La forma directa 5x5 del gaussiano, solo para comparar tiempos contra la separable. No se entrega como filtro final, pero si se mide con un solo hilo y se reporta.
- El filtro de Sobel secuencial con la magnitud del gradiente, calculando gx y gy en un solo recorrido de la vecindad y saturando el resultado a 255.
- La version paralela de ambos con #pragma omp parallel for sobre las filas, con la politica de planificacion justificada (static o dynamic segun lo que midan).
- Las convenciones de borde por repeticion del pixel mas cercano y el redondeo al final con empates hacia arriba, en los dos filtros.

### Que mide

- El arnes del repositorio para gaussiano y Sobel con 1, 2, 4 y 8 hilos.
- La comparacion obligatoria entre el gaussiano separable y la forma directa 5x5 con un solo hilo. Esa comparacion es parte del criterio de instrumentacion.
- La verificacion byte a byte de la salida paralela contra la secuencial con 1 y 8 hilos, para los dos filtros.

### Que escribe en el informe

En 02-motor.md, la seccion de stencils:

- Como quedo implementado cada filtro y por que la pasada vertical del gaussiano castiga la localidad.
- La justificacion del reparto por filas y no por columnas, con la jerarquia de memoria como argumento.
- Las tablas de T(p), S(p) y E(p) de gaussiano y Sobel.
- La comparacion separable contra directo 5x5.
- La evidencia de la comparacion byte a byte.

---

## Persona B - Reducciones: histograma, ecualizacion, Otsu y optimizacion de localidad

# scarllys
### Que implementa

- El histograma, la ecualizacion y el umbral de Otsu secuenciales.
- Dos versiones paralelas del histograma: una con acumulador compartido protegido con atomic o critical, solo para comparar, y otra con histogramas privados por hilo y suma al final, con relleno para evitar false sharing.
- El barrido de los 255 umbrales de Otsu se deja secuencial y se estima con la Ley de Amdahl que fraccion del tiempo total representa.
- La optimizacion de localidad sobre el filtro que peor escale, con la medicion antes y despues. La decision de cual filtro optimizar se toma en equipo cuando esten las primeras tablas, no antes.

### Que mide

- La comparacion entre el histograma con acumulador compartido y el de histogramas privados. Es requisito explicito de la rubrica.
- La medicion antes y despues de la optimizacion de localidad, con el mismo arnes.
- El perfilado con perf stat (ciclos, instrucciones y fallos de cache) y la conclusion que se saca de esos numeros.

### Que escribe en el informe

En 02-motor.md, la seccion de reducciones:

- La discusion de contencion y false sharing entre histogramas privados.
- La comparacion de las dos versiones del histograma.
- La estimacion de Amdahl para el barrido de Otsu.
- La evidencia del perfilador y de la optimizacion de localidad, con la explicacion de que cambio en el acceso a memoria.

### Coordinacion con Persona A

A y B escriben el mismo archivo 02-motor.md. Conviene que A sea el dueno del archivo y B mande sus secciones por pull request a una rama propia, o que desde el inicio dividan el archivo con marcadores claros. Los dos deben usar la misma maquina declarada para que las tablas sean comparables.

---

## Persona C - Servicio local: trabajador, despachador y Docker Compose
# Marlen
### Que implementa

- La capa HTTP del trabajador en Python con las rutas POST /procesar, GET /healthz y GET /readyz. POST /procesar recibe la imagen en crudo con content type application octet stream, la cadena de filtros en la consulta, y responde con la imagen resultante y las cabeceras X-Tiempo-Ms y X-Hilos. Una cadena con un filtro desconocido se rechaza con HTTP 400.
- La invocacion del motor C++ desde el trabajador. Los filtros se calculan en C++, no en Python.
- El despachador en Python con concurrent.futures o threading, con concurrencia configurable por variable de ambiente, leyendo el lote de entrada y escribiendo las salidas.
- El Dockerfile del trabajador y el del despachador.
- El archivo deploy/local/docker-compose.yml que levanta el servicio completo con una sola orden, con scale trabajador igual a 1 y a 4.

### Que mide

- El throughput sobre el lote de 64 imagenes de 1024x1024 con uno y con cuatro trabajadores.
- Las cifras quedan en mediciones/servicio.csv con las columnas entorno, trabajadores, concurrencia, imagenes y segundos.

### Que escribe en el informe

En 01-arquitectura.md:

- Los tres componentes y su interaccion.
- El diagrama Mermaid del servicio.
- El contrato HTTP del trabajador con sus rutas y codigos de estado.
- Como reparte el despachador.
- Por que las imagenes viajan en crudo y no en JSON con base64.
- Por que el almacenamiento lo monta solo el despachador.

En 03-despliegue.md, la seccion local:

- Los Dockerfiles de cada componente.
- El archivo de Compose comentado.
- La tabla de throughput local con uno y cuatro trabajadores.

---

## Persona D - Nube: GKE, GitHub Actions y SonarQube

### Que implementa

- Los manifiestos en deploy/nube/: Deployment del trabajador con tres replicas y sondas liveness y readiness apuntando a /healthz y /readyz, Service ClusterIP, ConfigMap con OMP_NUM_THREADS y la cadena de filtros por defecto, requests y limits de CPU y memoria justificados, y el despachador como Job con un PersistentVolumeClaim para entrada y salida.
- Las imagenes en Artifact Registry con etiqueta inmutable, usando el SHA del commit y nunca latest.
- El flujo de GitHub Actions con los cinco pasos minimos: compilar el motor con Wall y Wextra y correr las pruebas incluida la comparacion byte a byte, construir las imagenes, autenticarse contra Google Cloud, publicar en Artifact Registry con la etiqueta del SHA, y aplicar los manifiestos con kubectl rollout status.
- La integracion de SonarQube, como contenedor o como escaner dentro del flujo, con sonar-project.properties versionado. Hay que corregir al menos dos hallazgos y documentar el antes y el despues del codigo.

### Que mide

- El throughput con una y con tres replicas del trabajador en GKE, sobre el mismo lote de 64 imagenes.
- El gasto del credito tomado del tablero de facturacion de Google Cloud.
- La evidencia de que el cluster quedo eliminado, con la orden usada y la salida.

### Que escribe en el informe

En 03-despliegue.md, la seccion nube:

- Los manifiestos de Kubernetes con la justificacion de los requests y limits.
- El flujo de Actions con la forma de autenticacion elegida y por que.
- La evidencia del rollout.
- La integracion de SonarQube con los dos hallazgos corregidos.
- La tabla comparativa de throughput local contra nube.
- El gasto del credito y la evidencia del apagado.

En 04-conclusiones.md:

- La lectura de las cifras medidas.
- Que se intento y no funciono.
- Los limites que se encontraron.
- Que se haria distinto con mas tiempo.

### Riesgo de bloqueo

D depende de que el motor y el trabajador ya funcionen para poder desplegar. Para no quedar bloqueado, D empieza por los YAML, el workflow y SonarQube en seco con imagenes de prueba, y los sustituye por las reales cuando A, B y C las tengan listas.

---

## Tareas que se comparten entre los cuatro

Estas tareas son transversales. Si se le dejan a una sola persona, el reparto se desbalancea.

- El indice docs/README.md y la tabla criterio a archivo. Sesion corta al inicio para fijar el mapeo y revision al final.
- La verificacion byte a byte de la cadena completa con 1 y 8 hilos. La corre A o B, pero C y D deben reproducirla en su entorno antes de firmar la entrega.
- La maquina declarada y la metodologia de medicion. A y B miden el motor, C y D miden el servicio. Debe ser una sola maquina para las tablas del motor y un solo procedimiento para el throughput, escrito una vez en 02-motor.md.
- La revision cruzada del informe antes del push final. Cada quien lee el archivo de otro y deja comentarios.
- El ensayo de sustentacion. Como las preguntas son sobre todo el proyecto, cada quien debe poder explicar en dos minutos la parte de los otros tres.

---

## Reglas para que el reparto cuente en la nota

- Un dueno por archivo del informe. A y B comparten 02-motor.md con secciones delimitadas. C y D comparten 03-despliegue.md igual. Asi se evitan conflictos de merge.
- Ninguna credencial va al repositorio. D es el mas expuesto a esto: se usa Workload Identity Federation o un secreto del repositorio, nunca un archivo de cuenta de servicio.
- Las pruebas automatizadas en Actions se montan desde el principio, no al final. Es criterio de la rubrica y ademas protege a A y B de romper la salida byte a byte sin darse cuenta.
- La optimizacion de localidad es una sola, no una por persona. B la lidera, A aporta la parte del gaussiano si la optimizacion elegida es la pasada vertical por bloques. Se decide en equipo cuando esten las primeras tablas.

## Qué trae el repositorio

```
motor/            el motor en C++ con OpenMP
  src/            lectura y escritura de PPM y PGM, y los filtros
  tests/          pruebas de corrección y el comparador byte a byte
  bench/          arnés de medición y generador de la imagen de medición
trabajador/       capa HTTP sobre el motor
despachador/      reparto del lote
mediciones/       las tablas que alimentan el informe
deploy/local/     el servicio completo con Docker Compose
deploy/nube/      los manifiestos del clúster
docs/             el informe, en cuatro archivos más el índice
```

De los filtros vienen resueltos la escala de grises y el desenfoque de caja de
3x3, cada uno en su versión secuencial y en su versión paralela. El desenfoque
gaussiano separable, Sobel y el histograma con Otsu están declarados en
`motor/src/filtros.hpp` y sin implementar.

## Construir y correr el motor

```console
$ cd motor
$ make
g++ -std=c++17 -O2 -Wall -Wextra -fopenmp -Isrc -o motor src/main.cpp src/filtros.cpp src/imagen.cpp
g++ -std=c++17 -O2 -Wall -Wextra -fopenmp -Isrc -o comparar tests/comparar.cpp
g++ -std=c++17 -O2 -Wall -Wextra -fopenmp -Isrc -o generar bench/generar.cpp src/imagen.cpp
g++ -std=c++17 -O2 -Wall -Wextra -fopenmp -Isrc -o arnes bench/arnes.cpp src/filtros.cpp src/imagen.cpp
```

El motor recibe la cadena de filtros separados por comas, la imagen de entrada
y la de salida. Con `--secuencial` corre la versión de un hilo:

```console
$ ./motor gris tests/datos/entrada.ppm /tmp/salida.pgm
$ ./motor caja,caja tests/datos/entrada.ppm /tmp/suave.ppm --secuencial
```

La cadena del lote es `gris,gaussiano,sobel,otsu`: entra un PPM de color y
sale un PGM binario. Cada filtro necesita cierto número de canales y entrega
otro, así que el motor recorre la cadena llevando la cuenta y la rechaza antes
de aplicar nada cuando un filtro no va a recibir lo que necesita. `gris` pide
tres canales y devuelve uno; `sobel` y `otsu` piden uno; `caja` y `gaussiano`
trabajan con cualquiera de los dos.

## Las pruebas

```console
$ make pruebas
Salida esperada, calculada aparte:
  ok    gris sobre la imagen de prueba
  ok    caja sobre la imagen de prueba
Lectura y escritura:
  ok    gris rechaza una imagen de un canal
  ok    caja sobre un canal escribe P5
Paralelo contra secuencial, imagen de 512x512:
  ok    gris idéntico con uno y con varios hilos
  ok    caja idéntico con uno y con varios hilos
Filtro desconocido:
  ok    la cadena con un filtro inexistente se rechaza
Cadenas que no se pueden aplicar:
  ok    sobel se rechaza antes de aplicar nada
  ok    gris,gris se rechaza antes de aplicar nada
  ok    otsu se rechaza antes de aplicar nada

todas las pruebas pasaron
```

Las salidas esperadas de `tests/datos/` se calcularon con una implementación
aparte de las mismas definiciones, así que una prueba que pasa dice que las
dos coinciden. Un filtro nuevo se prueba igual: su salida paralela tiene que
ser idéntica byte a byte a la secuencial, y `comparar` es quien lo dice.

## Medir

La imagen de medición es de 4096x4096 y no se versiona: la escribe el
generador, que produce siempre el mismo archivo.

```console
$ ./generar /tmp/medicion.ppm 4096 4096
escrito /tmp/medicion.ppm: 4096x4096, 50331648 bytes de píxeles
```

El arnés cronometra solo la llamada al filtro, con la imagen ya en memoria, y
escribe una fila por repetición. El número de hilos sale de
`OMP_NUM_THREADS`; el resumen de la mediana descarta la primera corrida.

```console
$ OMP_NUM_THREADS=1 ./arnes gris /tmp/medicion.ppm 3 --encabezado
filtro,hilos,repeticion,milisegundos
gris,1,1,2.51226
gris,1,2,2.46808
gris,1,3,2.09325
gris con 1 hilo: mediana 2.28067 ms sobre 2 corridas
```

`bench/medir.sh` recorre 1, 2, 4 y 8 hilos para los filtros que se le pasen y
deja la tabla completa en `mediciones/motor.csv`:

```console
$ bash bench/medir.sh /tmp/medicion.ppm gris caja
escrito ../mediciones/motor.csv con 40 filas
```

## Convenciones que no cambian

De ellas depende que la comparación byte a byte signifique algo.

1. **Bordes por repetición del píxel más cercano.** La vecindad que se sale de
   la imagen usa el píxel del borde: las coordenadas se recortan a
   `[0, ancho-1]` y `[0, alto-1]`. No se rellena con ceros ni se recorta la
   salida.
2. **Acumulación en `double` y redondeo al final**, al entero más cercano con
   los empates hacia arriba, saturando a `[0, 255]`. Están en `redondear()`.
3. **Las reducciones son sobre enteros.** El histograma acumula contadores
   enteros y Otsu trabaja sobre el histograma, así que el resultado no depende
   del orden en que los hilos acumulen.
