# Proyecto: Filtrado y Limpieza de Imagen para Detección de Monedas mediante Operaciones Morfológicas
---

## 1) Resumen

- **Equipo / Autor(es):**  _Karen Nájera_  
- **Curso / Asignatura:** _Elementos Programables / Proyecto de Control_  
- **Fecha:** _05/12/2025_  
- **Descripción breve:**  
  _Esta práctica implementa un proceso de filtrado digital para identificar monedas dentro de una imagen estática mediante el uso de OpenCV. Se utilizan técnicas como manipulación de canales RGB, umbralización, enmascaramiento, selección de rangos en HSV y operaciones morfológicas tipo opening (erosión + dilatación). Finalmente, el sistema detecta contornos, marca las monedas encontradas y etiqueta cada una. El objetivo es comprender cómo la secuencia correcta de transformaciones permite limpiar ruido y aislar objetos de interés._

---

## 2) Objetivos

- **General:**  
  _Aplicar técnicas fundamentales de procesamiento digital de imágenes para segmentar monedas dentro de una escena, eliminando ruido y resaltando contornos mediante operadores morfológicos._

- **Específicos:**  
  - _Manipular canales de color para resaltar regiones de interés._  
  - _Implementar umbralización y creación de máscaras binarias._  
  - _Utilizar operaciones morfológicas (erosión, dilatación y opening) para limpiar la imagen._  
  - _Detectar contornos y calcular centroides usando momentos._

---

## 3) Alcance y Exclusiones

- **Incluye:**  
  - _Lectura y preprocesamiento de imágenes con OpenCV._  
  - _Aplicación de umbralización y conversión de espacios de color._  
  - _Obtención de contornos y centroides de cada moneda._  
  - _Uso de kernels elípticos para filtrado morfológico._

- **Exclusiones / restricciones:**  
  - _No se realizan técnicas avanzadas como segmentación adaptativa, watershed o clustering._  
  - _El análisis se limita a una imagen estática; no se incluye video ni seguimiento temporal._  
  - _No se aplica clasificación basada en tamaño o tipo de moneda; solo se detectan._

---
## 4) Desarrollo
_En esta práctica se utilizó una imagen con varias monedas sobre un fondo uniforme. El procesamiento inició con la lectura de la imagen original y su reducción de tamaño para facilitar el análisis. Posteriormente, se manipularon los canales RGB eliminando selectivamente los canales rojo y azul para conservar únicamente la información del canal verde, el cual proporcionaba el mejor contraste entre las monedas y el fondo.

Una vez aislado el canal de interés, se aplicó una umbralización binaria, que permitió crear una máscara preliminar de regiones brillantes. Para refinar la segmentación, la máscara fue convertida al espacio HSV, donde se aplicó un nuevo filtrado mediante selección de rangos, lo cual mejoró la separación entre monedas y ruido.

El núcleo de la práctica se centró en las operaciones morfológicas, particularmente el opening, que combina erosión seguida de dilatación. Estas operaciones son esenciales para eliminar pequeños puntos de ruido y regularizar bordes sin afectar en gran medida el tamaño real de los objetos. Se utilizó un kernel elíptico, adecuado para preservar la geometría circular de las monedas.

Tras limpiar la imagen, se realizó la detección de contornos mediante findContours. Para cada contorno se calcularon sus momentos, necesarios para determinar el centroide, el cual se marcó en la imagen mediante un punto verde. Finalmente, cada moneda se etiquetó con un número consecutivo, empleando texto superpuesto, lo que permitió identificar de manera clara cuántas monedas fueron detectadas y su posición._

## 5) Resultados

_El sistema logró segmentar correctamente las monedas presentes en la imagen, eliminando la mayor parte del ruido y resaltando únicamente los objetos deseados. Gracias al uso del canal verde y a la selección adecuada del umbral, la máscara inicial fue suficientemente precisa para que las operaciones morfológicas terminaran de limpiar la escena.

La detección de contornos identificó cada moneda de manera clara y estable. Los centroides calculados mediante los momentos geométricos fueron precisos y permitieron etiquetar correctamente cada objeto. El resultado final muestra la imagen original con contornos marcados en color azul y con un número asociado a cada moneda, confirmando la efectividad del método implementado._

[Video control mano](https://youtu.be/zepItAOh-Lk)
[Video control PD](https://youtu.be/MQ0QVBZc3m0)

---

## 6) Archivos Adjuntos / Código

_En este apartado se integrarán los archivos correspondientes al firmware del ESP32. Se incluirá un archivo destinado al control autónomo mediante detección de la pelota y otro para el modo controlado por la mano del usuario. También se agregarán las imágenes y diagramas asociados al montaje físico de la plataforma y a las pruebas realizadas._

```cpp
import cv2
import numpy as np
'''OPENING'''
# Cargar imagen
image = cv2.imread('E2_week2-main/images\monedas.jpg')
image = cv2.resize(image, (0,0), fx=0.5, fy=0.5)
image1 = image.copy()
image1 [:, :, 2] = 0
image1 [:, :, 0] = 0
 
 
 
green_channel = image[:, :, 1]
th, imThresh = cv2.threshold(image1, 70, 255, cv2.THRESH_BINARY)
mascara = cv2.cvtColor(imThresh, cv2.COLOR_BGR2HSV)
mask1=cv2.inRange(mascara,(50,100,100),(80,255,255))
 
 
 
# Crear kernel
kSize = 3
kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (2*kSize+1, 2*kSize+1), (kSize, kSize))
'''OPEN Metodo 1'''
# Perform Erosion
 
# Perform Dilation
imOpen = cv2.dilate(mask1, kernel, iterations=1)
imEroded = cv2.erode(imOpen, kernel, iterations=3)
imEroded  = cv2.dilate(imEroded, kernel, iterations=1)
 
'''OPEN Metodo 2'''
#imageMorphOpened = cv2.morphologyEx(mask1, cv2.MORPH_OPEN,
                        #kernel,iterations=3)
# Mostrar Imagenes
contours, hierarchy = cv2.findContours(imEroded, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
cv2.drawContours(image, contours, -1, (255, 0, 0), 3)
for index,cnt in enumerate(contours):
    M = cv2.moments(cnt)
    x = int(round(M["m10"]/M["m00"]))
    y = int(round(M["m01"]/M["m00"]))
    cv2.circle(image, (x,y), 10, (0,255,0), -1)
    # Marcar Texto
    cv2.putText(image, "{}".format(index + 1), (x-10, y+10), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
cv2.imshow("OG", image)
 
cv2.imshow("Eroded", imEroded)
cv2.waitKey(0)
cv2.destroyAllWindows()
```

## 7) Conclusión}
_La práctica permitió comprender de manera integral cómo el preprocesamiento adecuado de una imagen influye directamente en la calidad de la segmentación y en la detección de objetos de interés. A través de la manipulación de canales de color, la aplicación de umbralización y el uso de operaciones morfológicas, fue posible limpiar eficazmente la imagen y aislar las monedas presentes en la escena. El uso de un kernel elíptico resultó especialmente adecuado debido a la geometría circular de los objetos, permitiendo conservar su forma mientras se eliminaba ruido no deseado.

Asimismo, la obtención de contornos y el cálculo de centroides evidenciaron la importancia de las etapas finales del procesamiento, pues permiten no solo identificar los objetos, sino también describir su posición y características básicas dentro de la imagen. En conjunto, esta práctica demuestra cómo la combinación secuencial de filtros y transformaciones puede resolver problemas de segmentación de manera robusta y eficiente, preparando el camino para aplicaciones más avanzadas de visión computacional._