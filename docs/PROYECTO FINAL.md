# Proyecto: Plataforma de Balanceo Activo para Mantener una Pelota Centrada
---

## 1) Resumen

- **Equipo / Autor(es):**  _Karen Nájera_  
- **Curso / Asignatura:** _Elementos Programables / Proyecto de Control_  
- **Fecha:** _05/12/2025_  
- **Descripción breve:**  
  _Este proyecto implementa una plataforma inclinable controlada por 3 servomotores (2 para eje X: izquierdo/derecho y 1 para eje Y: arriba/abajo) cuyo objetivo es mantener una pelota ligera en el centro. Se desarrollaron dos métodos de visión, uno basado en detección por color para ubicar una pelota y otro basado en detección de líneas para controlar la plataforma con la mano.
Además, se integró un controlador PD que recibe la posición detectada por la cámara y envía comandos al ESP32 mediante Bluetooth. El ESP32 ejecuta el movimiento con rampado, límites y retorno automático a posición neutral._

---

## 2) Objetivos

- **General:**  
  _Implementar y validar un sistema embebido que, combinando visión por computadora y control PD, mantenga una pelota centrada sobre una plataforma inclinable y también permita un segundo modo de control mediante gestos de la mano._

- **Específicos:**  
  - _Diseñar la plataforma y la distribución de servos (2 en X, 1 en Y)._  
  - _Implementar el nodo de visión que detecte la pelota y calcule acciones PD._  
  - _Implementar firmware en ESP32 que reciba comandos por Bluetooth y mueva los servos de forma segura (rampa y límites)._  
  - _Probar y ajustar ganancias PD para lograr compromiso entre rapidez y suavidad._

---

## 3) Alcance y Exclusiones

- **Incluye:**  
  - _Detección de pelota por color con OpenCV (Python)._  
  - _Cálculo de control PD en el nodo de visión (Python)._  
  - _Comunicación Bluetooth entre PC (visión) y ESP32._  
  - _Firmware ESP32 que aplica comandos a 3 servos con límite, rampa y timeout de seguridad._

- **Exclusiones / restricciones:**  
  - _No se implementa realimentación de posición (no hay encoders)._  
  - _No se implementa autenticación/TLS en la comunicación._  
  - _El sistema asume condiciones de iluminación y contraste adecuadas para la detección por color._

---

## 4) Resultados

_Al ejecutar el sistema: la cámara detecta la pelota y el nodo de visión calcula el error en pixeles respecto al centro de la imagen (X, Y). El controlador PD calcula correcciones y envía comandos `DX` y `DY` por Bluetooth al ESP32 (o bien puede enviar directamente ángulos `AX,AY,AZ`). El ESP32 aplica esos comandos a los servos con rampado, límites de ángulo y timeout de seguridad; si no recibe comandos vuelve a la posición neutral._

_Los parámetros PD (`Kp`, `Kd`), el suavizado EMA, `MAX_STEP` y `MAX_ACCEL` permiten ajustar el compromiso entre rapidez y suavidad. Para pelotas muy ligeras las ganancias altas producen movimientos bruscos que lanzan la pelota; por eso recomendamos limitar el paso y usar rampa._

_Durante las pruebas del sistema, la cámara logró detectar de manera efectiva la posición de la pelota en la imagen y generar un error en los ejes X y Y comparándolo con el centro teórico. El nodo de visión calculó las acciones de control PD y las envió al ESP32 como comandos de corrección o como ángulos finales, dependiendo del modo de operación. El ESP32 ejecutó estos movimientos de forma estable gracias al rampado, que evitó cambios de posición bruscos y mantuvo el movimiento continuo sin sacudidas. La plataforma pudo responder correctamente a los desplazamientos de la pelota, inclinándose de manera proporcional y derivativa para regresar la pelota al centro. El segundo modo de control, basado en la mano del usuario, también mostró un funcionamiento adecuado y permitió mover la plataforma con libertad mientras se visualizaban las líneas guía en pantalla. Durante las pruebas se comprobó que los ajustes de Kp y Kd afectan significativamente el comportamiento: valores altos causan oscilaciones intensas y movimientos excesivamente rápidos, mientras que valores moderados permiten un equilibrio entre estabilidad y capacidad de reacción. El sistema demostró ser funcional bajo ambos modos de visión, cumpliendo con los objetivos planteados._

---

## 5) Archivos Adjuntos / Código

_En este apartado se integrarán los archivos correspondientes al firmware del ESP32 y a los dos scripts de visión. Se incluirá un archivo destinado al control autónomo mediante detección de la pelota y otro para el modo controlado por la mano del usuario. También se agregarán las imágenes y diagramas asociados al montaje físico de la plataforma y a las pruebas realizadas._

- ****  

```cpp
#include <Arduino.h>
#include "BluetoothSerial.h"
 
BluetoothSerial SerialBT;
 
// === Buffer para lectura BT no bloqueante ===
String btBuffer;
 
// === Pines de los servos ===
#define SERVO_IZQ   25
#define SERVO_DER   15
#define SERVO_VERT  33
 
// === PWM ===
const uint32_t FREQ_HZ = 50;
const uint8_t  RES_BITS = 12;
const uint16_t DUTY_MIN = 205;   // ~1.0 ms
const uint16_t DUTY_MAX = 410;   // ~2.0 ms
 
// Convierte grados físicos 0..180 a duty
uint16_t dutyFromDeg(int deg){
  deg = constrain(deg,0,180);
  return map(deg,0,180,DUTY_MIN,DUTY_MAX);
}
 
// Convierte de ángulo lógico (0..180, centro 90) a físico (invertido)
int logicalToPhysical(int logicalDeg){
  logicalDeg = constrain(logicalDeg, 0, 180);
  // 0 lógico → 180 físico, 180 lógico → 0 físico
  return 180 - logicalDeg;
}
 
// Escribe usando grados lógicos
void writeServoLogical(int pin, int logicalDeg){
  int fisico = logicalToPhysical(logicalDeg);
  ledcWrite(pin, dutyFromDeg(fisico));
}
 
// Configurar servo con ángulo lógico inicial
void configServo(int pin, int initialLogical){
  pinMode(pin,OUTPUT);
  ledcAttach(pin,FREQ_HZ,RES_BITS);   // usa el pin como canal
  writeServoLogical(pin,initialLogical);
}
 
// === Rango y rampa ===
const int LIM_MIN = 0;
const int LIM_MAX = 180;
const int PASO_RAMPA = 45;          // tamaño de paso en rampa
const uint32_t DT_RAMP_MS = 2;
const uint32_t TIMEOUT_MS = 700;
 
// Estado en grados LÓGICOS
int posIzq = 90;
int posDer = 90;
int posVert= 60;
 
int tgtIzq = 90;
int tgtDer = 90;
int tgtVert= 60;
 
uint32_t tPrevRamp = 0;
uint32_t tLastCmd  = 0;
 
// Rampa suave hacia el objetivo
void aplicarRampa(){
  uint32_t now = millis();
  if(now - tPrevRamp < DT_RAMP_MS) return;
  tPrevRamp = now;
 
  auto go = [&](int actual,int target){
    if(actual < target) return min(actual + PASO_RAMPA, target);
    if(actual > target) return max(actual - PASO_RAMPA, target);
    return actual;
  };
 
  posIzq = go(posIzq, tgtIzq);
  posDer = go(posDer, tgtDer);
  posVert= go(posVert,tgtVert);
 
  // Escribimos usando grados LÓGICOS, se invierten adentro
  writeServoLogical(SERVO_IZQ, posIzq);
  writeServoLogical(SERVO_DER, posDer);
  writeServoLogical(SERVO_VERT,posVert);
}
 
// Parsea "ANG:izq,der,vert"
bool parseAngulos(const String &msg, int &aIzq, int &aDer, int &aVert){
  if (!msg.startsWith("ANG:")) return false;
  String data = msg.substring(4);  // después de "ANG:"
 
  int c1 = data.indexOf(',');
  if (c1 < 0) return false;
  int c2 = data.indexOf(',', c1 + 1);
  if (c2 < 0) return false;
 
  String sIzq  = data.substring(0, c1);
  String sDer  = data.substring(c1 + 1, c2);
  String sVert = data.substring(c2 + 1);
 
  sIzq.trim();
  sDer.trim();
  sVert.trim();
 
  aIzq  = sIzq.toInt();
  aDer  = sDer.toInt();
  aVert = sVert.toInt();
 
  return true;
}
 
// ======== SETUP ========
void setup(){
  Serial.begin(115200);
  SerialBT.begin("ESP32-BallPlatform");
 
  configServo(SERVO_IZQ, posIzq);
  configServo(SERVO_DER, posDer);
  configServo(SERVO_VERT,posVert);
 
  Serial.println("ESP32 listo");
  tLastCmd = millis();
}
 
// ======== LOOP ========
void loop(){
 
  // --- Lectura Bluetooth no bloqueante ---
  while (SerialBT.available()) {
    char c = (char)SerialBT.read();
 
    if (c == '\n') {
      // Tenemos una línea completa en btBuffer
      String msg = btBuffer;
      btBuffer = "";        // limpiar para el siguiente mensaje
 
      msg.trim();
      if (msg.length() > 0) {
        tLastCmd = millis();
 
        if (msg == "LOST") {
          // PC perdió mano/objetivo → "centro"
          tgtIzq  = 90;
          tgtDer  = 90;
          tgtVert = 60;
          Serial.println("Comando LOST: centro.");
 
        } else if (msg == "ZERO") {
          // 🔹 TODOS los servos a 180 físicos
          //    ⇒ 0 lógico por el mapeo invertido
          tgtIzq  = 0;
          tgtDer  = 0;
          tgtVert = 0;
          Serial.println("Comando ZERO: servos → 180° fisico");
 
        } else {
          int aIzq, aDer, aVert;
          if (parseAngulos(msg, aIzq, aDer, aVert)) {
            tgtIzq  = constrain(aIzq,  LIM_MIN, LIM_MAX);
            tgtDer  = constrain(aDer,  LIM_MIN, LIM_MAX);
            tgtVert = constrain(aVert, LIM_MIN, LIM_MAX);
            Serial.printf("ANG -> IZQ:%d DER:%d VERT:%d\n", tgtIzq, tgtDer, tgtVert);
          } else {
            Serial.print("Comando desconocido: ");
            Serial.println(msg);
          }
        }
      }
    } else if (c != '\r') {
      // Acumulamos caracteres, ignorando CR
      btBuffer += c;
    }
  }
 
  // Si pasa mucho tiempo sin recibir comandos, vuelve al centro
  if(millis() - tLastCmd > TIMEOUT_MS){
    tgtIzq = 90;
    tgtDer = 90;
    tgtVert= 60;
  }
 
  aplicarRampa();
  delay(1);
}
 

```


## 6) Conclusión

_El proyecto permitió comprobar que la combinación de visión por computadora, control PD y servomotores puede implementarse de manera efectiva para mantener una pelota centrada sobre una plataforma inclinable. La separación en dos modos de operación —detección automática por filtros de color y control manual mediante el seguimiento de la mano— permitió validar tanto la capacidad del sistema para operar en lazo cerrado como su respuesta a órdenes directas. La integración entre Python y el ESP32 funcionó de forma estable y permitió el envío continuo de datos para corregir la posición en tiempo real. Se concluye que la plataforma es una base sólida para futuras implementaciones más complejas, como controles PID completos, superficies móviles de mayor tamaño, o incluso sistemas de estabilización aplicados a robótica móvil._
