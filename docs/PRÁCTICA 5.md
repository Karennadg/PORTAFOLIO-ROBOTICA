# 📚 Práctica 5: ESP32-C6 como Beacon (BLE Advertising)

---

## 1) Resumen

- **Equipo / Autor(es):** _Karen Nájera y Arith Maldonado_  
- **Curso / Asignatura:** _Elementos programables II_  
- **Fecha:** _18/09/2025_  
- **Descripción breve:**  
  En esta práctica configuramos el **ESP32-C6** como un **dispositivo BLE** que anuncia su presencia (Advertising). Se crea un **servidor BLE**, un **servicio** con UUID propio y se inicia el **anuncio** para que pueda ser detectado por apps como *nRF Connect* o *LightBlue*.

---

## 2) Objetivos

- Inicializar el stack **BLE** del ESP32-C6 con un **nombre de dispositivo**.
- Crear un **servidor BLE**, un **servicio** y una **característica** básica.
- Iniciar y comprobar el **Advertising** desde el Monitor Serial y el teléfono.

---

## 3) Materiales

- ESP32-C6 Dev Module  
- Cable USB  
- Arduino IDE (core ESP32 instalado)  
- App BLE en smartphone (p. ej., **nRF Connect** / **LightBlue**)

---

## 4) Código

```cpp
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>

// ====== Configuración ======
static const char* DEVICE_NAME = "AKDevice";  // Nombre que verás en el escaneo
#define SERVICE_UUID        "12345678-1234-1234-1234-1234567890ab"
#define CHARACTERISTIC_UUID "abcd1234-5678-90ab-cdef-1234567890ab"

void setup() {
  Serial.begin(115200);
  delay(100);

  // 1) Inicializa BLE con nombre
  BLEDevice::init(DEVICE_NAME);

  // 2) Crea servidor y servicio
  BLEServer*      pServer  = BLEDevice::createServer();
  BLEService*     pService = pServer->createService(SERVICE_UUID);

  // 3) Característica de ejemplo (Lectura/Escritura)
  BLECharacteristic* pCharacteristic = pService->createCharacteristic(
      CHARACTERISTIC_UUID,
      BLECharacteristic::PROPERTY_READ   |
      BLECharacteristic::PROPERTY_WRITE
  );
  pCharacteristic->setValue("Hola BLE");

  // 4) Inicia el servicio
  pService->start();

  // 5) Configura el advertising
  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);   // Anuncia el UUID del servicio
  pAdvertising->setScanResponse(true);          // Info extra en la respuesta de escaneo
  pAdvertising->setMinPreferred(0x06);          // Parámetros recomendados
  pAdvertising->setMinPreferred(0x12);

  // 6) Inicia advertising
  BLEDevice::startAdvertising();
  Serial.println("BLE Advertising started");
}

void loop() {
  // Nada en el loop; el anuncio corre en segundo plano
  delay(1000);
}
