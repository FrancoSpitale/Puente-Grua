Este código implementa un sistema de control para motores utilizando un joystick, botones de control y comunicación por Bluetooth. Está diseñado para operar dos motores conectados a un ESP32 a través de RS485 y relés. A continuación, se detalla su funcionalidad principal:

Descripción General
Hardware:
Joystick: Controla la dirección y velocidad de los motores en los ejes X e Y.
Botones:
"Up" y "Down" para controlar un motor adicional (malacate).
"Emergency" para detener todos los motores en caso de emergencia.
Bluetooth: Permite recibir comandos de control remotos.
RS485: Comunicación con los dos motores principales.
Finales de carrera: Limita el rango de movimiento de los motores en ambos ejes.
Funcionamiento del Código
Configuración Inicial (setup):

Configura los pines de los relés, control RS485 y finales de carrera.
Inicia la comunicación Bluetooth como maestro.
Establece la conexión con un módulo HC-06.
Ciclo Principal (loop):

Revisa si hay datos disponibles desde el Bluetooth.
Procesa los datos para controlar los motores y botones.
Control del Joystick y Botones:

Extrae los valores de los ejes X e Y y ajusta la velocidad y dirección de los motores.
Verifica el estado de los botones "Up", "Down" y "Emergency" para activar o desactivar los relés correspondientes.
Control de los Motores:

Los valores del joystick determinan la dirección y velocidad de los motores.
La velocidad se escala a un rango definido usando map.
Los finales de carrera previenen movimientos fuera de rango.
Los comandos se envían a los motores mediante el protocolo Modbus por RS485.
Envío de Comandos Modbus:

Utiliza la función sendModbusCommand para enviar datos al motor.
Calcula el CRC del mensaje para garantizar la integridad.
Funciones Clave
Control del Motor
cpp
Copiar código
void controlMotor(int joystickValue, int neutralMin, int neutralMax, int txControlPin, HardwareSerial& motorSerial, int limitSwitchMin, int limitSwitchMax, uint8_t motorAddress) {
  int scaledSpeed;
  uint16_t direction;

  if (joystickValue > neutralMax && !digitalRead(limitSwitchMax)) {
    scaledSpeed = map(joystickValue, neutralMax, 1023, minSpeed, maxSpeed);
    direction = 0x0001;  // Adelante
  } else if (joystickValue < neutralMin && !digitalRead(limitSwitchMin)) {
    scaledSpeed = map(joystickValue, neutralMin, 0, minSpeed, maxSpeed);
    direction = 0x0002;  // Atrás
  } else {
    scaledSpeed = 0;
    direction = 0x0006;  // Detener
  }

  setMotorSpeed(scaledSpeed, direction, txControlPin, motorSerial, motorAddress);
}
Comandos Modbus
cpp
Copiar código
void sendModbusCommand(uint8_t slaveID, uint16_t reg, uint16_t value, int txControlPin, HardwareSerial& motorSerial) {
  uint8_t message[8];
  message[0] = slaveID;
  message[1] = 0x06;
  message[2] = highByte(reg);
  message[3] = lowByte(reg);
  message[4] = highByte(value);
  message[5] = lowByte(value);

  uint16_t crc = calculateCRC(message, 6);
  message[6] = lowByte(crc);
  message[7] = highByte(crc);

  digitalWrite(txControlPin, RS485Transmit);
  for (int i = 0; i < 8; i++) {
    motorSerial.write(message[i]);
  }
  delay(10);
  digitalWrite(txControlPin, RS485Receive);
}
Cálculo de CRC
cpp
Copiar código
uint16_t calculateCRC(uint8_t *buffer, uint8_t length) {
  uint16_t crc = 0xFFFF;
  for (int pos = 0; pos < length; pos++) {
    crc ^= (uint16_t)buffer[pos];
    for (int i = 8; i != 0; i--) {
      if ((crc & 0x0001) != 0) {
        crc >>= 1;
        crc ^= 0xA001;
      } else {
        crc >>= 1;
      }
    }
  }
  return crc;
}
Mejoras Potenciales
