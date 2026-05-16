#include <Keyboard.h> // Librería nativa para emular teclado USB
#include "MIDIUSB.h"  // Librería nativa para enviar notas MIDI por USB

// =========================================================================
// CONFIGURACIÓN PERIFÉRICOS DEL TELÉFONO (PINES 2 Y 3)
// =========================================================================
const int pinDisco = 2;   
const int pinGancho = 3;  

int estadoAnteriorDisco = HIGH;
int contadorPulsos = 0;
unsigned long ultimoCambioPinDisco = 0;
unsigned long ultimoPulsoDetectado = 0;

const unsigned long tiempoDebounceDisco = 35; 
const unsigned long tiempoFinNumero = 800; 

int estadoAnteriorGancho = HIGH;
unsigned long ultimoCambioGancho = 0;
const unsigned long tiempoDebounceGancho = 50; 

String numeroCompleto = ""; 
bool comandoEnProgreso = false;
bool modoMidiActivo = false; 

// =========================================================================
// VARIABLES Y BUFFER DE LA PANTALLA NEXTION (INTACTO)
// =========================================================================
char seq[4] = {0,0,0,0};
bool macroRunning = false;

void pushChar(char c) {
  seq[0] = seq[1];
  seq[1] = seq[2];
  seq[2] = seq[3];
  seq[3] = c;
}

bool match3(const char* s) {
  return (seq[1]==s[0] && seq[2]==s[1] && seq[3]==s[2]);
}

bool match2(const char* s) {
  return (seq[2]==s[0] && seq[3]==s[1]);
}

// =========================================================================
// FUNCIONES DE ENVÍO SEGURO (TECLADO NEXTION)
// =========================================================================
void sendKey(char key) {
  Keyboard.press(key);
  delay(40);
  Keyboard.release(key);
  delay(20);
}

void sendCombo(uint8_t mod1, uint8_t mod2, char key) {
  Keyboard.press(mod1);
  if (mod2 != 0) Keyboard.press(mod2);
  delay(20);
  Keyboard.press(key);
  delay(40);
  Keyboard.release(key);
  if (mod2 != 0) Keyboard.release(mod2);
  Keyboard.release(mod1);
  delay(30);
}

void sendArrow(uint8_t key) {
  Keyboard.press(key);
  delay(40);
  Keyboard.release(key);
  delay(20);
}

void sendShiftArrow(uint8_t key) {
  Keyboard.press(KEY_LEFT_SHIFT);
  delay(10);
  Keyboard.press(key);
  delay(40);
  Keyboard.release(key);
  Keyboard.release(KEY_LEFT_SHIFT);
  delay(30);
}

// =========================================================================
// FUNCIÓN AUXILIAR PARA ENVIAR NOTAS MIDI NATIVAS (TELÉFONO)
// =========================================================================
void enviarNotaMidiTelefono(byte canal, byte nota, byte velocidad) {
  midiEventPacket_t notaOn = {0x09, (byte)(0x90 | (canal & 0x0F)), nota, velocidad};
  MidiUSB.sendMIDI(notaOn);
  MidiUSB.flush();
  delay(100); 
  midiEventPacket_t notaOff = {0x08, (byte)(0x80 | (canal & 0x0F)), nota, 0};
  MidiUSB.sendMIDI(notaOff);
  MidiUSB.flush();
}

// =========================================================================
// MACRO R01 ORIGINAL NEXTION (INTACTO)
// =========================================================================
void runMacro_R01() {
  while (Serial1.available()) Serial1.read();
  delay(120);
  sendKey('p');
  delay(80);
  sendCombo(KEY_LEFT_CTRL, KEY_LEFT_SHIFT, 'm');
  delay(80);
  sendKey('l');
  delay(80);
  sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, 'a');
  delay(80);
  sendCombo(KEY_LEFT_CTRL, KEY_LEFT_SHIFT, '3');
  delay(80);
  for (int i = 0; i < 4; i++) {
    Keyboard.press(KEY_LEFT_CTRL);
    delay(15);
    Keyboard.press('y');
    delay(50);
    Keyboard.release('y');
    Keyboard.release(KEY_LEFT_CTRL);
    delay(100);
    Keyboard.press(KEY_LEFT_ALT);
    delay(10);
    Keyboard.press('x');
    delay(40);
    Keyboard.release('x');
    Keyboard.release(KEY_LEFT_ALT);
    delay(120);
  }
  sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, 'a'); 
  delay(120);
  Keyboard.press(KEY_LEFT_CTRL);
  Keyboard.press(KEY_LEFT_ALT);
  delay(30);
  Keyboard.press('u');
  delay(120);
  Keyboard.release('u');
  delay(80);
  Keyboard.release(KEY_LEFT_ALT);
  Keyboard.release(KEY_LEFT_CTRL);
  delay(120);
}

// =========================================================================
// 3. MATRIZ EXCLUSIVA DEL TELÉFONO (CORREGIDA PARA 3 UNOS SEGUIDOS)
// =========================================================================
void ejecutarAtajoTelefono(String codigo) {
  
  // CORRECCIÓN COMPLETA: Al marcar 1 -> 1 -> 1, el código interno es "1"
  if (codigo == "1") {
    modoMidiActivo = !modoMidiActivo; 
    Serial.print("\n[Sistema] MODO CAMBIADO: ");
    Serial.println(modoMidiActivo ? ">>> MIDI <<<" : ">>> SHORTCUTS DE TECLADO <<<");
    return; 
  }

  // 2. EJECUCIÓN DE COMANDOS PERMITIDOS
  if (modoMidiActivo) {
    // ---- CAPA MIDI ----
    if (codigo == "2") { // Secuencia: 1 -> 2 -> 1
      Serial.println("[MIDI] Enviando Nota C4 (Do Central)");
      enviarNotaMidiTelefono(0, 60, 127); 
    }
    else {
      Serial.println("[MIDI] Codigo no asignado.");
    }
  } 
  else {
    // ---- CAPA SHORTCUTS / SISTEMA ----
    if (codigo == "0") { // Secuencia: 1 -> 0 -> 1
      Serial.println("Comando CRÍTICO: Apagando PC mediante menú WinX...");
      Keyboard.press(KEY_LEFT_GUI); Keyboard.press('x'); delay(100); Keyboard.releaseAll();
      delay(250); 
      Keyboard.press('u'); delay(100); Keyboard.releaseAll();
      delay(150); 
      Keyboard.press('u'); delay(100); Keyboard.releaseAll();
    }
    else {
      Serial.println("[Shortcut] Codigo no asignado.");
    }
  }
}

// =========================================================================
// SETUP UNIFICADO
// =========================================================================
void setup() {
  Serial.begin(9600);
  Serial1.begin(9600); 
  
  pinMode(pinDisco, INPUT); 
  pinMode(pinGancho, INPUT_PULLUP); 
  
  Keyboard.begin();
  Serial.println("[Sistema] Ecosistema unificado listo. Nextion y Teléfono activos.");
}

// =========================================================================
// LOOP UNIFICADO (PROCESAMIENTO EN PARALELO)
// =========================================================================
void loop() {
  unsigned long tiempoActual = millis();

  // -------------------------------------------------------------------------
  // MOTOR 1: CONTROL DEL TELÉFONO A DISCO
  // -------------------------------------------------------------------------
  int lecturaGancho = digitalRead(pinGancho);
  
  if (lecturaGancho != estadoAnteriorGancho) {
    if ((tiempoActual - ultimoCambioGancho) > tiempoDebounceGancho) {
      estadoAnteriorGancho = lecturaGancho;
      ultimoCambioGancho = tiempoActual;
      
      if (lecturaGancho == HIGH) { 
        Serial.println("\n[Línea] Tubo DESCOLGADO. Listo para marcar.");
      } 
      else { 
        Serial.println("\n[Línea] Tubo COLGADO. Memoria limpia.");
        comandoEnProgreso = false;
        numeroCompleto = "";
        contadorPulsos = 0;
      }
    }
  }

  if (estadoAnteriorGancho == HIGH) { 
    int estadoActualDisco = digitalRead(pinDisco);

    if (estadoActualDisco != estadoAnteriorDisco) {
      if ((tiempoActual - ultimoCambioPinDisco) > tiempoDebounceDisco) {
        if (estadoActualDisco == HIGH) {
          contadorPulsos++;
          ultimoPulsoDetectado = tiempoActual;
        }
        ultimoCambioPinDisco = tiempoActual;
        estadoAnteriorDisco = estadoActualDisco;
      }
    }

    if (contadorPulsos > 0 && (tiempoActual - ultimoPulsoDetectado) > tiempoFinNumero) {
      int numeroMarcado = contadorPulsos;
      if (numeroMarcado == 10) numeroMarcado = 0;
      
      if (numeroMarcado <= 9) {
        if (!comandoEnProgreso) {
          if (numeroMarcado == 1) {
            comandoEnProgreso = true;
            numeroCompleto = ""; 
            Serial.println("[Teléfono] Escuchando código...");
          } else {
            Serial.println("[Error] Para iniciar un comando debes marcar el número 1");
          }
        } 
        else {
          // Cierra el comando al marcar el "1" final y tener al menos un dígito acumulado
          if (numeroMarcado == 1 && numeroCompleto.length() > 0) {
            Serial.print("[Teléfono] Cierre detectado. Procesando código interno: ");
            Serial.println(numeroCompleto);
            
            ejecutarAtajoTelefono(numeroCompleto);
            
            comandoEnProgreso = false;
            numeroCompleto = "";
          } else {
            numeroCompleto += String(numeroMarcado);
            Serial.print("Dígito guardado: ");
            Serial.println(numeroCompleto);
          }
        }
      }
      contadorPulsos = 0; 
    }
  }

  // -------------------------------------------------------------------------
  // MOTOR 2: LECTURA DE PANTALLA NEXTION Y MACROS (ORIGINAL INTACTO)
  // -------------------------------------------------------------------------
  while (Serial1.available()) {
    char c = Serial1.read();
    pushChar(c);

    if (match3("R01") && !macroRunning) {
      macroRunning = true;
      runMacro_R01();
      macroRunning = false;
    }

    else if (match3("Q01")) sendKey('q');

    else if (match2("B1")) sendCombo(KEY_LEFT_ALT, 0, '3');
    else if (match2("B2")) sendCombo(KEY_LEFT_ALT, 0, '4');
    else if (match2("B3")) sendCombo(KEY_LEFT_ALT, 0, '5');
    else if (match2("B4")) sendCombo(KEY_LEFT_ALT, 0, '1');
    else if (match2("B5")) sendCombo(KEY_LEFT_ALT, 0, '2');
    else if (match2("B6")) sendCombo(KEY_LEFT_ALT, 0, '6');
    else if (match2("B7")) sendCombo(KEY_LEFT_ALT, 0, '7');
    else if (match2("B8")) sendCombo(KEY_LEFT_ALT, 0, ',');
    else if (match2("B9")) sendCombo(KEY_LEFT_ALT, 0, '.');

    else if (match3("B10")) sendShiftArrow(KEY_DOWN_ARROW);
    else if (match3("B11")) sendShiftArrow(KEY_UP_ARROW);

    else if (match3("B13")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_SHIFT, 'a');
    else if (match3("B14")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, 'a');

    else if (match3("B15")) sendArrow(KEY_LEFT_ARROW);
    else if (match3("B16")) sendArrow(KEY_RIGHT_ARROW);

    else if (match3("B17")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, '1');
    else if (match3("B18")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, '2');
    else if (match3("B19")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_SHIFT, 'm');
    else if (match3("B21")) sendCombo(KEY_LEFT_CTRL, KEY_LEFT_ALT, '5');

    else if (match3("B20")) {
      Keyboard.press(KEY_F3);
      delay(50);
      Keyboard.release(KEY_F3);
      delay(30);
    }

    else if (match3("B22")) sendCombo(KEY_LEFT_CTRL, 0, 'z');
  }
}
