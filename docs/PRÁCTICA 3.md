# 📚 Práctica 3: Comunicación Serial con ESP32-C6 en Arduino

---

## 1) Resumen

- **Equipo / Autor(es):**  _Karen Nájera y Arith Maldonado_  
- **Curso / Asignatura:** _Elementos programables II_  
- **Fecha:** _18/09/2025_  
- **Descripción breve:**  
  En esta práctica se implementó un código en **Arduino IDE** para controlar un **NeoPixel** desde un **ESP32-C6** mediante comandos enviados por **Serial** con el formato `R<r>,G<g>,B<b>`, donde cada valor está en el rango **0–255**. Se repasaron los tipos de variables y se compararon los puertos **UART** y **USB nativo** del ESP32.

---

## 2) Objetivos

- **General:**  
  _Comprender el funcionamiento de la comunicación serial en el ESP32-C6 para el control de un LED RGB (NeoPixel) usando comandos `R,G,B`._

- **Específicos:**  
  - Identificar los principales tipos de variables usados en Arduino.  
  - Implementar un programa que reciba un comando por Serial y aplique un color al NeoPixel.  
  - Diferenciar el puerto **UART** y el **USB nativo** del ESP32-C6.  
  - Verificar la correcta decodificación de comandos y la actualización del color.

---

## 3) Alcance y Exclusiones

- **Incluye:**  
  - Uso del ESP32-C6 como dispositivo receptor de comandos seriales.  
  - Configuración del **baud rate** y pruebas con el **Monitor Serial**.  
  - Control de un **NeoPixel** (1 LED) con comandos `R,G,B`.

- **No incluye:**  
  - Conexión a sensores externos.  
  - Programación de librerías adicionales fuera de **Adafruit NeoPixel**.  
  - Uso de Wi-Fi / Bluetooth.

---

## 4) Resultados

Durante la práctica se logró:  

- **Recepción de comandos seriales** en formato `R<r>,G<g>,B<b>` y aplicación inmediata del color al NeoPixel.  
- **Velocidad serial efectiva:** **115200 baudios** (el Monitor Serial y `Serial.begin` deben coincidir).  
- **Validación de rangos:** cada canal se limita a `0–255` usando `constrain(...)`.  
- **Ejemplos probados:** `R120,G110,B10`, `R255,G0,B0`, `R0,G0,B255`.

---

**Código Implementado**

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

    // Extrae el número después de la letra (R/G/B)
    r = constrain(rPart.substring(1).toInt(), 0, 255);
    g = constrain(gPart.substring(1).toInt(), 0, 255);
    b = constrain(bPart.substring(1).toInt(), 0, 255);
 
    pixels.clear();
    pixels.setPixelColor(0, pixels.Color(r, g, b));
    pixels.show();
  }
}
