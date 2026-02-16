# Remote Control - Versione Raspberry Pi 7"

## Modifiche per Raspberry Pi 7" (800x480)

Questo file `RemoteControl_RaspberryPi.pde` è una versione modificata del codice Processing originale, ottimizzata per l'uso con display Raspberry Pi 7" touchscreen (800x480).

## Caratteristiche Principali

### 1. **Risoluzione Ottimizzata**
- Risoluzione predefinita: **800x480** (Raspberry Pi 7")
- Finestra ridimensionabile abilitata

### 2. **Scala UI Regolabile**
- Scala predefinita: **0.6x** (ottimale per 800x480)
- Range scala: **0.4x - 1.5x**
- Incremento/Decremento: **0.05** per pressione tasto

### 3. **Controlli da Tastiera**

| Tasto | Funzione |
|-------|----------|
| **↑** (Freccia SU) | Aumenta scala UI (+0.05) |
| **↓** (Freccia GIÙ) | Diminuisce scala UI (-0.05) |
| **F** | Toggle fullscreen |
| **H** | Emergency halt |
| **R** | Scansiona porte seriali |
| **D** | Toggle debug mode |
| **P** | Toggle rotator power |
| **1-6** | Seleziona antenna |
| **ESC** | Torna a schermata controllo |

### 4. **Indicatore Scala**
Un indicatore in tempo reale in basso a destra mostra:
- Valore corrente della scala (es. "0.60x")
- Icona frecce ↑↓ come promemoria dei controlli

## Come Usare

1. **Aprire il file con Processing**
   ```
   processing-java --sketch=/path/to/RemoteControl_RaspberryPi --run
   ```

2. **Regolare la scala**
   - Usa le frecce ↑↓ per trovare la dimensione ottimale per il tuo schermo
   - L'indicatore in basso a destra mostra il valore corrente

3. **Fullscreen**
   - Premi 'F' per passare alla modalità fullscreen
   - Premi nuovamente 'F' per tornare alla modalità finestra

## Modifiche Tecniche Implementate

### Variabili Globali
```java
float uiScale = 0.6;
final float MIN_SCALE = 0.4;
final float MAX_SCALE = 1.5;
final float SCALE_STEP = 0.05;
```

### Helper Functions
```java
float getScaledMouseX() { return mouseX / uiScale; }
float getScaledMouseY() { return mouseY / uiScale; }
```

### Trasformazione Draw Loop
Tutto il contenuto UI è avvolto in una trasformazione di scala:
```java
pushMatrix();
scale(uiScale);
// ... contenuto UI ...
popMatrix();
```

### Mouse Input Scalato
Tutti i controlli mouse ora usano `getScaledMouseX()` e `getScaledMouseY()` invece di `mouseX` e `mouseY` diretti, garantendo un corretto funzionamento con qualsiasi scala.

## Compatibilità

✅ Raspberry Pi 7" (800x480)  
✅ Schermi più grandi (con regolazione scala)  
✅ Touch screen  
✅ Mouse/Trackpad  
✅ Tutte le funzionalità originali mantenute

## Note

- Il file originale `README.md` contiene il codice Processing originale non modificato
- Questa versione (`RemoteControl_RaspberryPi.pde`) è pronta per l'uso su Raspberry Pi
- Tutte le funzionalità ESP32, antenne, rotatori e debug sono mantenute
