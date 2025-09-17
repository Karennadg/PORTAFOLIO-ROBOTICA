# 📚 Práctica 3: Control de LED NeoPixel por comandos R,G,B (ESP32-C6 + Arduino IDE)

---

## 1) Resumen

- **Equipo / Autor(es):** _Karen Nájera y Arith Maldonado_  
- **Curso / Asignatura:** _Elementos programables II_  
- **Fecha:** _18/09/2025_  
- **Descripción breve:**  
  En esta práctica se implementó un programa en **Arduino IDE** para controlar un **NeoPixel** conectado a un **ESP32-C6** recibiendo comandos por **Serial** en el formato `R<r>,G<g>,B<b>` (0–255). Se reforzó el manejo de tipos de variables, el uso de la librería **Adafruit_NeoPixel** y la diferencia práctica entre **UART** y **USB nativo**.

> **Tip:** Mantén este resumen corto (máx. 5 líneas). Lo demás va en secciones específicas.

---

## 2) Objetivos

- **General:**  
  _Comprender y aplicar la comunicación serial en el ESP32-C6 para controlar un LED NeoPixel mediante comandos `R,G,B`._

- **Específicos:**  
  - Configurar el puerto **Serial** y verificar su velocidad.  
  - Parsear un comando de texto con tres canales (R, G, B).  
  - Limitar cada canal al rango válido **0–255** y actualizar el color del NeoPixel.  
  - Comparar el uso de **USB nativo** frente a **UART** para depuración y pruebas.

---

## 3) Alcance y Exclusiones

- **Incluye:**  
  - Control de **1** LED NeoPixel (`NUMPIXELS = 1`).  
  - Recepción de comandos por **Monitor Serial** (`R<r>,G<g>,B<b>`).  
  - Ajuste de **baud rate** y verificación de eco.

- **No incluye:**  
  - Conexión a sensores externos.  
  - Uso de Wi-Fi / Bluetooth.  
  - Efectos avanzados o animaciones en tiras LED.

---

## 4) Resultados

- **Recepción de comandos seriales** y aplicación inmediata del color en el NeoPixel.  
- **Velocidad usada:** **115200 baudios** (coincidente entre `Serial.begin` y Monitor Serial).  
- **Validación de entrada:** cada canal se restringe con `constrain(...)` a **0–255**.  
- **Ejemplos probados:** `R120,G110,B10`, `R255,G0,B0`, `R0,G0,B255`.

---

## 5) Protocolo de comandos y pruebas

- **Formato:** `R<r>,G<g>,B<b>`  
- **Rango:** `0–255` por canal.  
- **Delimitación:** valores separados por **coma** y finalizados con **Enter** (`\n`).  

**Pruebas sugeridas**
- `R255,G0,B0` (rojo)  
- `R0,G255,B0` (verde)  
- `R0,G0,B255` (azul)  
- `R255,G191,B0` (ámbar)  
- `R10,G10,B10` (atenuado)

---

## 6) Código Implementado

```cpp
#include <Adafruit_NeoPixel.h>
#ifdef __AVR__
  #include <avr/power.h>
#endif
 
#define PIN 8
#define NUMPIXELS 1
 
Adafruit_NeoPixel pixels(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);
 
String cmd = "";
int r = 0, g = 0, b = 0;
 
void setup() {
  Serial.begin(115200);
  pixels.begin();
}
 
void loop() {
  if (Serial.available() > 0) {
    cmd = Serial.readStringUntil('\n');
    Serial.println("Msj recibido: " + cmd);
 
    int pos1 = cmd.indexOf(',');      
    int pos2 = cmd.indexOf(',', pos1 + 1);
 
    String rPart = cmd.substring(0, pos1);                
    String gPart = cmd.substring(pos1 + 1, pos2);        
    String bPart = cmd.substring(pos2 + 1);              

    // Extrae el número después de la letra (R/G/B) y limita a 0-255
    r = constrain(rPart.substring(1).toInt(), 0, 255);
    g = constrain(gPart.substring(1).toInt(), 0, 255);
    b = constrain(bPart.substring(1).toInt(), 0, 255);
 
    pixels.clear();
    pixels.setPixelColor(0, pixels.Color(r, g, b));
    pixels.show();
  }
}

