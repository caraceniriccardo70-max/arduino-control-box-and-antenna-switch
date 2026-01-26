

import processing.serial.*;
import java.util.*;
import java.text. SimpleDateFormat;

// ═══════════════════════════════════════════════════════════════════════════
//  CONFIGURAZIONE GLOBALE
// ═══════════════════════════════════════════════════════════════════════════

final String APP_NAME = "9A8DV Remote Control";
final String APP_VERSION = "2.1";
final String CALLSIGN = "9A8DV";
final String CALLSIGN_NAME = "HAM RADIO STATION";

// ═══════════════════════════════════════════════════════════════════════════
//  ANIMAZIONE STARTUP
// ═══════════════════════════════════════════════════════════════════════════

boolean showingSplash = true;
float splashProgress = 0;
long splashStartTime;
final int SPLASH_DURATION = 5000;

ArrayList<Particle> particles = new ArrayList<Particle>();
ArrayList<Star> stars = new ArrayList<Star>();

class Particle {
  float x, y, vx, vy, size, alpha, life;
  color col;
  
  Particle(float x, float y) {
    this.x = x;
    this.y = y;
    float angle = random(TWO_PI);
    float speed = random(1, 4);
    vx = cos(angle) * speed;
    vy = sin(angle) * speed;
    size = random(2, 6);
    alpha = 255;
    life = random(60, 120);
    col = color(0, 255, 136);
  }
  
  void update() {
    x += vx; y += vy;
    vx *= 0.98; vy *= 0.98;
    life--;
    alpha = map(life, 0, 60, 0, 255);
  }
  
  void draw() {
    noStroke();
    fill(red(col), green(col), blue(col), alpha);
    ellipse(x, y, size, size);
    fill(red(col), green(col), blue(col), alpha * 0.3);
    ellipse(x, y, size * 2, size * 2);
  }
  
  boolean isDead() { return life <= 0; }
}

class Star {
  float x, y, size, twinkle, speed;
  
  Star() {
    x = random(800); y = random(600);
    size = random(1, 3);
    twinkle = random(TWO_PI);
    speed = random(0.02, 0.08);
  }
  
  void update() { twinkle += speed; }
  
  void draw() {
    float alpha = 100 + 155 * sin(twinkle);
    fill(255, 255, 255, alpha);
    noStroke();
    ellipse(x, y, size, size);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  TEMA COLORI
// ═══════════════════════════════════════════════════════════════════════════

class ThemeColors {
  color primary = #000000;
  color secondary = #1A1A1A;
  color accent = #00FF88;
  color background = #0A0A0A;
  color panel = #111111;
  color panelLight = #1E1E1E;
  color text = #FFFFFF;
  color textDim = #888888;
  color success = #00FF88;
  color warning = #FFAA00;
  color error = #FF4444;
  color disabled = #333333;
  color hover = #2A2A2A;
  color border = #444444;
  color cwColor = #00FF00;
  color ccwColor = #FF8800;
  color haltColor = #FF3030;
  color rotorPowerOn = #00AAFF;
  color rotorPowerOff = #555555;
}

ThemeColors theme = new ThemeColors();

// ═══════════════════════════════════════════════════════════════════════════
//  NOTIFICHE
// ═══════════════════════════════════════════════════════════════════════════

class NotificationManager {
  ArrayList<Notification> items = new ArrayList<Notification>();
  
  void add(String msg, int type) {
    items.add(new Notification(msg, type));
    while (items.size() > 3) items.remove(0);
  }
  
  void update() {
    for (int i = items.size() - 1; i >= 0; i--) {
      items.get(i).update();
      if (items.get(i).isDead()) items.remove(i);
    }
  }
  
  void draw() {
    for (int i = 0; i < items.size(); i++) {
      Notification n = items.get(i);
      n.targetY = 70 + i * 60;
      n.draw();
    }
  }
}

class Notification {
  String message;
  int type;
  long timestamp;
  float alpha = 255, y = -60, targetY = 70;
  boolean removing = false;
  
  Notification(String msg, int t) { message = msg; type = t; timestamp = millis(); }
  
  void update() {
    y = lerp(y, targetY, 0.15);
    if (millis() - timestamp > 4000) removing = true;
    if (removing) alpha -= 8;
  }
  
  boolean isDead() { return removing && alpha <= 0; }
  
  void draw() {
    if (alpha <= 0) return;
    color bgColor = type == SUCCESS ? theme.success : type == WARNING ? theme.warning : type == ERROR ? theme.error : theme.accent;
    fill(0, 0, 0, alpha * 0.4);
    noStroke();
    rect(width - 264, y + 3, 220, 50, 10);
    fill(red(bgColor), green(bgColor), blue(bgColor), alpha);
    rect(width - 260, y, 220, 48, 10);
    fill(255, 255, 255, alpha);
    textFont(fontRegular);
    textSize(11);
    textAlign(LEFT, CENTER);
    text(message, width - 245, y + 24);
    float progress = 1.0 - (float)(millis() - timestamp) / 4000.0;
    if (progress > 0 && ! removing) {
      fill(255, 255, 255, alpha * 0.5);
      rect(width - 260, y + 42, 220 * progress, 3, 0, 0, 10, 10);
    }
  }
}

final int INFO = 0, SUCCESS = 1, WARNING = 2, ERROR = 3;
NotificationManager notificationManager = new NotificationManager();

void addNotification(String msg, int type) { notificationManager.add(msg, type); }

// ═══════════════════════════════════════════════════════════════════════════
//  SETTINGS MANAGER
// ═══════════════════════════════════════════════════════════════════════════

class SettingsManager {
  JSONObject config;
  String settingsFile = "config.json";
  
  SettingsManager() { loadSettings(); }
  
  void loadSettings() {
    try {
      File f = new File(sketchPath() + "/" + settingsFile);
      if (f.exists()) {
        config = loadJSONObject(settingsFile);
        applySettings();
      } else {
        createDefaultSettings();
      }
    } catch (Exception e) {
      createDefaultSettings();
    }
  }
  
  void createDefaultSettings() {
    config = new JSONObject();
    JSONArray antennasArray = new JSONArray();
    for (int i = 0; i < 6; i++) {
      JSONObject antenna = new JSONObject();
      antenna.setString("name", defaultAntennaNames[i]);
      antenna.setInt("pin", i + 4);
      antenna.setBoolean("directive", defaultAntennaDirective[i]);
      antennasArray.setJSONObject(i, antenna);
    }
    config.setJSONArray("antennas", antennasArray);
    config.setString("comPort", "COM4");
    config.setInt("baudRate", 9600);
    config.setBoolean("autoConnect", false);
    config.setBoolean("debugMode", true);
    saveSettings();
  }
  
  void applySettings() {
    try {
      JSONArray antennasArray = config.getJSONArray("antennas");
      for (int i = 0; i < min(6, antennasArray.size()); i++) {
        JSONObject antenna = antennasArray. getJSONObject(i);
        antennaNames[i] = antenna.getString("name");
        antennaPins[i] = antenna.getInt("pin");
        antennaDirective[i] = antenna.getBoolean("directive");
      }
      comPort = config.getString("comPort");
      baudRate = config. getInt("baudRate");
      autoConnect = config.getBoolean("autoConnect");
      debugMode = config.getBoolean("debugMode");
    } catch (Exception e) { }
  }
  
  void saveSettings() {
    try {
      JSONArray antennasArray = new JSONArray();
      for (int i = 0; i < 6; i++) {
        JSONObject antenna = new JSONObject();
        antenna.setString("name", antennaNames[i]);
        antenna.setInt("pin", antennaPins[i]);
        antenna.setBoolean("directive", antennaDirective[i]);
        antennasArray.setJSONObject(i, antenna);
      }
      config.setJSONArray("antennas", antennasArray);
      config.setString("comPort", comPort);
      config.setInt("baudRate", baudRate);
      config.setBoolean("autoConnect", autoConnect);
      config.setBoolean("debugMode", debugMode);
      saveJSONObject(config, settingsFile);
    } catch (Exception e) { }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  VARIABILI GLOBALI
// ═══════════════════════════════════════════════════════════════════════════

Serial arduino;
boolean arduinoConnected = false;
String comPort = "COM4";
int baudRate = 9600;
boolean autoConnect = false;
boolean debugMode = true;

boolean waitingForLeonardo = false;
long leonardoWaitStart = 0;
final long LEONARDO_WAIT_TIME = 3000;

SettingsManager settings;

String[] defaultAntennaNames = {"DxCommander", "3El 10-15-20", "Delta Loop 11m", "9el V/UHF", "Dipolo 80", "Dipolo 40"};
boolean[] defaultAntennaDirective = {true, false, false, true, false, false};

String[] antennaNames = new String[6];
int[] antennaPins = new int[6];
boolean[] antennaStates = new boolean[6];
boolean[] antennaDirective = new boolean[6];
int selectedAntenna = -1;

boolean systemOn = true;
boolean rotorPowerOn = false;  // NUOVO: Stato ON/OFF rotore
int currentScreen = 0;
float screenTransition = 0;
boolean transitioning = false;
int targetScreen = 0;

boolean rotorCW = false;
boolean rotorCCW = false;
boolean cwButtonPressed = false;
boolean ccwButtonPressed = false;
float currentAzimuth = 0;
float displayAzimuth = 0;
float mapCenterX, mapCenterY;
float mapRadius = 110;

boolean[] buttonHover = new boolean[30];
float[] buttonAnim = new float[30];

PFont fontRegular, fontBold, fontLarge, fontMono, fontCallsign;

ArrayList<String> debugLog = new ArrayList<String>();

String[] tempAntennaNames = new String[6];
int[] tempAntennaPins = new int[6];
boolean[] tempAntennaDirective = new boolean[6];
String tempComPort = "COM4";
int editingField = -1;
String inputBuffer = "";
int currentSettingsTab = 0;

String[] availablePorts;

// ═══════════════════════════════════════════════════════════════════════════
//  SETUP
// ═══════════════════════════════════════════════════════════════════════════

void setup() {
  size(800, 600);
  smooth(8);
  frameRate(60);
  surface.setTitle(APP_NAME + " v" + APP_VERSION);
  
  fontRegular = createFont("Segoe UI", 12);
  fontBold = createFont("Segoe UI Bold", 12);
  fontLarge = createFont("Segoe UI Bold", 20);
  fontMono = createFont("Consolas", 10);
  fontCallsign = createFont("Arial Black", 72);
  
  for (int i = 0; i < 6; i++) {
    antennaNames[i] = defaultAntennaNames[i];
    antennaPins[i] = i + 4;
    antennaStates[i] = false;
    antennaDirective[i] = defaultAntennaDirective[i];
  }
  
  arrayCopy(antennaNames, tempAntennaNames);
  arrayCopy(antennaPins, tempAntennaPins);
  arrayCopy(antennaDirective, tempAntennaDirective);
  tempComPort = comPort;
  
  for (int i = 0; i < 100; i++) stars.add(new Star());
  
  settings = new SettingsManager();
  scanSerialPorts();
  
  splashStartTime = millis();
  
  addDebugLog("═══════════════════════════════════════");
  addDebugLog("  " + APP_NAME + " v" + APP_VERSION);
  addDebugLog("═══════════════════════════════════════");
  addDebugLog("Sistema inizializzato");
  
  if (autoConnect && availablePorts != null && availablePorts.length > 0) {
    addDebugLog("Auto-connect attivo.. .");
    thread("autoConnectDelayed");
  }
}

void autoConnectDelayed() {
  delay(SPLASH_DURATION + 1000);
  if (! arduinoConnected && autoConnect) {
    connectArduino();
  }
}

void scanSerialPorts() {
  availablePorts = Serial.list();
  addDebugLog("Porte trovate: " + availablePorts.length);
}

String getTimestamp() {
  return new SimpleDateFormat("HH:mm:ss"). format(new Date());
}

void addDebugLog(String msg) {
  String entry = "[" + getTimestamp() + "] " + msg;
  debugLog.add(entry);
  while (debugLog.size() > 100) debugLog.remove(0);
  println(entry);
}

// ═══════════════════════════════════════════════════════════════════════════
//  MAIN DRAW LOOP
// ═══════════════════════════════════════════════════════════════════════════

void draw() {
  if (showingSplash) {
    drawSplashScreen();
    return;
  }
  
  drawBackground();
  updateAnimations();
  
  if (waitingForLeonardo && millis() - leonardoWaitStart >= LEONARDO_WAIT_TIME) {
    waitingForLeonardo = false;
    finishLeonardoConnection();
  }
  
  if (transitioning) {
    screenTransition += 0.1;
    if (screenTransition >= 1.0) {
      screenTransition = 0;
      transitioning = false;
      currentScreen = targetScreen;
    }
  }
  
  pushMatrix();
  if (transitioning) translate(-width * screenTransition, 0);
  drawCurrentScreen();
  popMatrix();
  
  if (transitioning) {
    pushMatrix();
    translate(width * (1 - screenTransition), 0);
    drawTargetScreen();
    popMatrix();
  }
  
  drawTopBar();
  drawNavigationBar();
  
  notificationManager.update();
  notificationManager.draw();
  
  if (waitingForLeonardo) drawLeonardoWaitOverlay();
}

void drawBackground() {
  for (int i = 0; i < height; i++) {
    float inter = map(i, 0, height, 0, 1);
    stroke(lerpColor(theme.background, color(5, 5, 10), inter));
    line(0, i, width, i);
  }
}

void updateAnimations() {
  for (int i = 0; i < buttonAnim.length; i++) {
    buttonAnim[i] = lerp(buttonAnim[i], buttonHover[i] ? 1.0 : 0.0, 0.2);
  }
  displayAzimuth = lerp(displayAzimuth, currentAzimuth, 0.15);
}

void drawCurrentScreen() {
  switch(currentScreen) {
    case 0: drawControlScreen(); break;
    case 1: drawSettingsScreen(); break;
    case 2: drawDebugScreen(); break;
  }
}

void drawTargetScreen() {
  switch(targetScreen) {
    case 0: drawControlScreen(); break;
    case 1: drawSettingsScreen(); break;
    case 2: drawDebugScreen(); break;
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  SPLASH SCREEN
// ═══════════════════════════════════════════════════════════════════════════

void drawSplashScreen() {
  long elapsed = millis() - splashStartTime;
  float progress = (float)elapsed / SPLASH_DURATION;
  
  if (progress >= 1.0) { showingSplash = false; return; }
  
  background(5, 5, 15);
  
  for (Star s : stars) { s.update(); s.draw(); }
  
  for (int i = particles.size() - 1; i >= 0; i--) {
    Particle p = particles. get(i);
    p. update(); p.draw();
    if (p.isDead()) particles.remove(i);
  }
  
  float phase1End = 0.15, phase2End = 0.35, phase3End = 0.55, phase4End = 0.75, phase5End = 1.0;
  
  pushMatrix();
  translate(width/2, height/2);
  
  if (progress < phase2End) {
    float circleProgress = min(1, progress / phase1End);
    float circleSize = 150 * circleProgress;
    float pulseSize = circleSize + 20 * sin(millis() * 0.005);
    
    for (int i = 5; i >= 0; i--) {
      fill(0, 255, 136, 10 * circleProgress);
      noStroke();
      ellipse(0, 0, pulseSize + i * 30, pulseSize + i * 30);
    }
    
    stroke(0, 255, 136, 255 * circleProgress);
    strokeWeight(3);
    noFill();
    ellipse(0, 0, circleSize, circleSize);
    
    float rotation = millis() * 0.002;
    stroke(0, 255, 136, 100 * circleProgress);
    strokeWeight(2);
    for (int i = 0; i < 4; i++) {
      float angle = rotation + i * HALF_PI;
      line(cos(angle) * 20, sin(angle) * 20, cos(angle) * circleSize * 0.4, sin(angle) * circleSize * 0.4);
    }
    
    if (frameCount % 3 == 0 && circleProgress > 0.5) {
      float angle = random(TWO_PI);
      particles.add(new Particle(width/2 + cos(angle) * circleSize/2, height/2 + sin(angle) * circleSize/2));
    }
  }
  
  if (progress > phase1End && progress < phase5End) {
    float textProgress = constrain((progress - phase1End) / (phase2End - phase1End), 0, 1);
    float textAlpha = 255 * textProgress;
    
    fill(0, 255, 136, textAlpha * 0.3);
    textFont(fontCallsign);
    textSize(80);
    textAlign(CENTER, CENTER);
    text(CALLSIGN, 4, -24);
    
    fill(0, 255, 136, textAlpha);
    text(CALLSIGN, 0, -30);
    
    float lineWidth = 300 * textProgress;
    stroke(0, 255, 136, textAlpha);
    strokeWeight(2);
    line(-lineWidth/2, 15, lineWidth/2, 15);
  }
  
  if (progress > phase2End && progress < phase5End) {
    float subProgress = constrain((progress - phase2End) / (phase3End - phase2End), 0, 1);
    float subAlpha = 255 * subProgress;
    
    fill(255, 255, 255, subAlpha);
    textFont(fontBold);
    textSize(16);
    textAlign(CENTER, CENTER);
    text(CALLSIGN_NAME, 0, 45);
    
    fill(150, 150, 150, subAlpha);
    textSize(12);
    text("Remote Control System v" + APP_VERSION, 0, 70);
  }
  
  if (progress > phase3End && progress < phase5End) {
    float barProgress = constrain((progress - phase3End) / (phase4End - phase3End), 0, 1);
    float barWidth = 250, barHeight = 6, barY = 110;
    
    fill(50, 50, 50, 200);
    noStroke();
    rect(-barWidth/2, barY, barWidth, barHeight, 3);
    
    fill(0, 255, 136);
    rect(-barWidth/2, barY, barWidth * barProgress, barHeight, 3);
    
    fill(150, 150, 150);
    textFont(fontRegular);
    textSize(10);
    textAlign(CENTER, CENTER);
    String[] loadingTexts = {"Initializing...", "Loading config...", "Preparing UI...", "Ready!"};
    text(loadingTexts[min((int)(barProgress * loadingTexts.length), loadingTexts.length - 1)], 0, barY + 25);
  }
  
  popMatrix();
  
  if (progress > phase4End) {
    float fadeProgress = (progress - phase4End) / (phase5End - phase4End);
    fill(5, 5, 15, 255 * fadeProgress);
    noStroke();
    rect(0, 0, width, height);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  SCHERMATA CONTROLLO
// ═══════════════════════════════════════════════════════════════════════════

void drawControlScreen() {
  drawAntennaPanel();
  drawRotorPanel();
  drawStatusBar();
}

void drawAntennaPanel() {
  float px = 20, py = 55, pw = 290, ph = 420;
  drawPanel(px, py, pw, ph, "ANTENNA SELECTOR", true);
  
  fill(theme.textDim);
  textFont(fontRegular);
  textSize(10);
  textAlign(LEFT, TOP);
  text("Clicca per selezionare", px + 20, py + 45);
  
  float startX = px + 15, startY = py + 70;
  float btnW = 125, btnH = 52, gapX = 10, gapY = 8;
  
  for (int i = 0; i < 6; i++) {
    int col = i % 2, row = i / 2;
    float bx = startX + col * (btnW + gapX);
    float by = startY + row * (btnH + gapY);
    drawAntennaButton(i, bx, by, btnW, btnH);
  }
}

void drawAntennaButton(int idx, float x, float y, float w, float h) {
  boolean hover = mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h && systemOn;
  buttonHover[idx] = hover;
  boolean selected = (selectedAntenna == idx);
  boolean active = antennaStates[idx];
  
  pushMatrix();
  if (hover) translate(0, -2 * buttonAnim[idx]);
  
  fill(0, 0, 0, 60 + 40 * buttonAnim[idx]);
  noStroke();
  rect(x + 3, y + 3, w, h, 10);
  
  color bgColor = ! systemOn ? theme.disabled : selected ? theme.accent : hover ? theme.hover : theme.secondary;
  fill(bgColor);
  stroke(selected ? theme.accent : theme.border);
  strokeWeight(selected ? 2 : 1);
  rect(x, y, w, h, 10);
  
  float ledX = x + w - 14, ledY = y + 12;
  fill(active ? theme.success : theme.disabled);
  noStroke();
  ellipse(ledX, ledY, 8, 8);
  if (active) { fill(theme.success, 100); ellipse(ledX, ledY, 14, 14); }
  
  if (antennaDirective[idx]) {
    fill(theme.warning);
    ellipse(x + 12, y + 12, 8, 8);
    fill(0);
    textFont(fontBold);
    textSize(6);
    textAlign(CENTER, CENTER);
    text("D", x + 12, y + 12);
  }
  
  fill(selected ? theme.primary : theme.text);
  textFont(fontBold);
  textSize(10);
  textAlign(CENTER, CENTER);
  String name = antennaNames[idx];
  if (name.length() > 14) name = name.substring(0, 12) + ".. ";
  text(name, x + w/2, y + h/2 - 6);
  
  fill(selected ? color(0, 0, 0, 150) : theme.textDim);
  textFont(fontRegular);
  textSize(8);
  text("PIN " + antennaPins[idx], x + w/2, y + h/2 + 10);
  
  popMatrix();
}

void drawRotorPanel() {
  float px = 330, py = 55, pw = 450, ph = 420;
  drawPanel(px, py, pw, ph, "ROTOR CONTROL", true);
  
  mapCenterX = px + pw/2;
  mapCenterY = py + 180;
  
  // NUOVO: Pulsante ON/OFF rotore
  drawRotorPowerSwitch(px + 20, py + 45);
  
  drawAzimuthMap();
  drawRotorButtons();
}

// NUOVO: Switch ON/OFF rotore
void drawRotorPowerSwitch(float x, float y) {
  float w = 120, h = 30;
  
  boolean hover = mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h && systemOn;
  buttonHover[27] = hover;
  
  // Background
  fill(rotorPowerOn ? theme.rotorPowerOn : theme.rotorPowerOff);
  stroke(rotorPowerOn ? theme.rotorPowerOn : theme.border);
  strokeWeight(hover ? 2 : 1);
  rect(x, y, w, h, 8);
  
  // LED indicatore
  float ledX = x + 15;
  fill(rotorPowerOn ? theme.success : theme.error);
  noStroke();
  ellipse(ledX, y + h/2, 10, 10);
  if (rotorPowerOn) {
    fill(theme.success, 100);
    ellipse(ledX, y + h/2, 16, 16);
  }
  
  // Testo
  fill(theme.text);
  textFont(fontBold);
  textSize(11);
  textAlign(LEFT, CENTER);
  text("ROTOR: " + (rotorPowerOn ? "ON" : "OFF"), x + 28, y + h/2);
  
  // Etichetta
  fill(theme. textDim);
  textFont(fontRegular);
  textSize(9);
  textAlign(LEFT, CENTER);
  text("Power (A3)", x + w + 10, y + h/2);
}

void drawAzimuthMap() {
  pushMatrix();
  translate(mapCenterX, mapCenterY);
  
  noFill();
  for (int i = 3; i >= 1; i--) {
    stroke(theme.border, 40 + i * 20);
    strokeWeight(1);
    ellipse(0, 0, mapRadius * 2 * i / 3, mapRadius * 2 * i / 3);
  }
  
  stroke(theme.accent, 180);
  strokeWeight(2);
  ellipse(0, 0, mapRadius * 2, mapRadius * 2);
  
  for (int deg = 0; deg < 360; deg += 30) {
    float angle = radians(deg - 90);
    float innerR = (deg % 90 == 0) ? mapRadius - 15 : mapRadius - 10;
    stroke(deg % 90 == 0 ? theme.text : theme.textDim, deg % 90 == 0 ? 200 : 100);
    strokeWeight(deg % 90 == 0 ? 2 : 1);
    line(cos(angle) * innerR, sin(angle) * innerR, cos(angle) * mapRadius, sin(angle) * mapRadius);
    
    if (deg % 30 == 0) {
      fill(theme.textDim);
      textFont(fontRegular);
      textSize(deg % 90 == 0 ? 10 : 8);
      textAlign(CENTER, CENTER);
      text(deg + "°", cos(angle) * (mapRadius + 18), sin(angle) * (mapRadius + 18));
    }
  }
  
  String[] cardinals = {"N", "E", "S", "W"};
  for (int i = 0; i < 4; i++) {
    float angle = radians(i * 90 - 90);
    fill(theme.text);
    textFont(fontBold);
    textSize(14);
    textAlign(CENTER, CENTER);
    text(cardinals[i], cos(angle) * (mapRadius + 35), sin(angle) * (mapRadius + 35));
  }
  
  if (selectedAntenna >= 0 && antennaDirective[selectedAntenna]) {
    float patternAngle = radians(displayAzimuth - 90);
    float beamWidth = radians(50);
    fill(theme.accent, 25);
    stroke(theme.accent, 60);
    strokeWeight(1);
    beginShape();
    vertex(0, 0);
    for (float a = patternAngle - beamWidth/2; a <= patternAngle + beamWidth/2; a += 0.05) {
      vertex(cos(a) * mapRadius * 0.75, sin(a) * mapRadius * 0.75);
    }
    endShape(CLOSE);
  }
  
  float needleAngle = radians(displayAzimuth - 90);
  
  stroke(0, 0, 0, 80);
  strokeWeight(4);
  line(2, 2, cos(needleAngle) * (mapRadius - 15) + 2, sin(needleAngle) * (mapRadius - 15) + 2);
  
  stroke(theme.accent);
  strokeWeight(3);
  line(0, 0, cos(needleAngle) * (mapRadius - 15), sin(needleAngle) * (mapRadius - 15));
  
  pushMatrix();
  translate(cos(needleAngle) * (mapRadius - 15), sin(needleAngle) * (mapRadius - 15));
  rotate(needleAngle + HALF_PI);
  fill(theme.accent);
  noStroke();
  triangle(-5, -10, 5, -10, 0, 0);
  popMatrix();
  
  if (rotorCW || rotorCCW) {
    stroke(rotorCW ? theme.cwColor : theme.ccwColor, 120 + 80 * sin(millis() * 0.01));
    strokeWeight(2);
    noFill();
    ellipse(0, 0, 50, 50);
  }
  
  fill(theme.secondary);
  stroke(rotorPowerOn ? theme.rotorPowerOn : theme.border);
  strokeWeight(2);
  ellipse(0, 0, 65, 65);
  
  if (systemOn && rotorPowerOn) {
    fill(theme.accent, 30);
    noStroke();
    ellipse(0, 0, 75, 75);
  }
  
  fill(systemOn && rotorPowerOn ? theme. accent : theme.disabled);
  textFont(fontBold);
  textSize(16);
  textAlign(CENTER, CENTER);
  text(nf(displayAzimuth, 1, 1) + "°", 0, -5);
  
  fill(theme.textDim);
  textSize(9);
  String status = "STOP";
  if (! rotorPowerOn) status = "OFF";
  else if (rotorCW) status = "→ CW";
  else if (rotorCCW) status = "← CCW";
  text(status, 0, 12);
    text(status, 0, 12);
  
  popMatrix();
}

void drawRotorButtons() {
  float centerX = mapCenterX;
  float btnY = mapCenterY + 155;
  float btnW = 75, btnH = 38, gap = 10;
  
  // Disabilita pulsanti se rotore è OFF
  boolean rotorEnabled = systemOn && rotorPowerOn;
  
  drawMomentaryButton("◄ CCW", centerX - btnW - gap/2 - btnW/2, btnY, btnW, btnH, 20, ccwButtonPressed, theme.ccwColor, rotorEnabled);
  drawHaltButton("HALT", centerX - btnW/2, btnY, btnW, btnH, 21, rotorEnabled);
  drawMomentaryButton("CW ►", centerX + gap/2 + btnW/2, btnY, btnW, btnH, 22, cwButtonPressed, theme.cwColor, rotorEnabled);
}

void drawMomentaryButton(String label, float x, float y, float w, float h, int idx, boolean pressed, color activeColor, boolean enabled) {
  boolean hover = mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h && enabled;
  buttonHover[idx] = hover;
  
  pushMatrix();
  if (hover || pressed) translate(0, -2 * buttonAnim[idx]);
  
  fill(0, 0, 0, 80);
  noStroke();
  rect(x + 2, y + 2, w, h, 8);
  
  color bgColor = ! enabled ? theme.disabled : pressed ? activeColor : theme.secondary;
  fill(bgColor);
  stroke(pressed ? activeColor : theme.border);
  strokeWeight(pressed ? 2 : 1);
  rect(x, y, w, h, 8);
  
  if (pressed && enabled) {
    stroke(activeColor, 100);
    strokeWeight(4);
    noFill();
    rect(x - 2, y - 2, w + 4, h + 4, 10);
  }
  
  fill(enabled ? (pressed ? theme.primary : theme.text) : theme. textDim);
  textFont(fontBold);
  textSize(11);
  textAlign(CENTER, CENTER);
  text(label, x + w/2, y + h/2);
  
  popMatrix();
}

void drawHaltButton(String label, float x, float y, float w, float h, int idx, boolean enabled) {
  boolean hover = mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h && enabled;
  buttonHover[idx] = hover;
  
  pushMatrix();
  if (hover) translate(0, -2 * buttonAnim[idx]);
  
  fill(0, 0, 0, 80);
  noStroke();
  rect(x + 2, y + 2, w, h, 8);
  
  color bgColor = !enabled ? theme.disabled : hover ? color(255, 60, 60) : theme.haltColor;
  fill(bgColor);
  stroke(theme.haltColor);
  strokeWeight(hover ? 2 : 1);
  rect(x, y, w, h, 8);
  
  fill(enabled ? theme.text : theme.textDim);
  textFont(fontBold);
  textSize(11);
  textAlign(CENTER, CENTER);
  text(label, x + w/2, y + h/2);
  
  popMatrix();
}

void drawStatusBar() {
  float px = 20, py = 485, pw = 760, ph = 45;
  
  fill(theme.panel);
  stroke(theme.border);
  strokeWeight(1);
  rect(px, py, pw, ph, 8);
  
  float iconY = py + ph/2;
  float startX = px + 20;
  float spacing = 115;
  
  drawStatusItem(startX, iconY, "Arduino", arduinoConnected ? "OK" : "DISC", arduinoConnected ? theme.success : theme.error);
  drawStatusItem(startX + spacing, iconY, "Sistema", systemOn ? "ON" : "OFF", systemOn ? theme. success : theme.warning);
  drawStatusItem(startX + spacing * 2, iconY, "Rotor", rotorPowerOn ?  "ON" : "OFF", rotorPowerOn ? theme.rotorPowerOn : theme.disabled);
  
  String rotorSt = "STOP";
  color rotorCol = theme.disabled;
  if (rotorCW) { rotorSt = "CW"; rotorCol = theme. cwColor; }
  else if (rotorCCW) { rotorSt = "CCW"; rotorCol = theme.ccwColor; }
  drawStatusItem(startX + spacing * 3, iconY, "Dir", rotorSt, rotorCol);
  
  String antSt = selectedAntenna >= 0 ?  "ANT" + (selectedAntenna + 1) : "NONE";
  drawStatusItem(startX + spacing * 4, iconY, "Ant", antSt, selectedAntenna >= 0 ?  theme.accent : theme.disabled);
  
  fill(theme.text);
  textFont(fontBold);
  textSize(11);
  textAlign(LEFT, CENTER);
  text("AZ:" + nf(displayAzimuth, 1, 1) + "°", startX + spacing * 5, iconY);
  
  fill(theme.textDim);
  textFont(fontRegular);
  textSize(10);
  textAlign(RIGHT, CENTER);
  text(getTimestamp(), px + pw - 15, iconY);
}

void drawStatusItem(float x, float y, String label, String value, color c) {
  fill(c);
  noStroke();
  ellipse(x, y, 10, 10);
  fill(c, 50);
  ellipse(x, y, 16, 16);
  
  fill(theme.text);
  textFont(fontRegular);
  textSize(9);
  textAlign(LEFT, CENTER);
  text(label + ":" + value, x + 10, y);
}

// ═══════════════════════════════════════════════════════════════════════════
//  SCHERMATA IMPOSTAZIONI
// ═══════════════════════════════════════════════════════════════════════════

void drawSettingsScreen() {
  float px = 40, py = 60, pw = 720, ph = 400;
  drawPanel(px, py, pw, ph, "IMPOSTAZIONI", false);
  drawSettingsTabs(px + 20, py + 45);
  
  switch(currentSettingsTab) {
    case 0: drawAntennaSettings(px, py + 90); break;
    case 1: drawConnectionSettings(px, py + 90); break;
    case 2: drawSystemSettings(px, py + 90); break;
  }
  
  drawSettingsButtons(px, py + ph - 50);
}

void drawSettingsTabs(float x, float y) {
  String[] tabs = {"Antenne", "Connessione", "Sistema"};
  float tabW = 110, tabH = 32, gap = 10;
  
  for (int i = 0; i < tabs.length; i++) {
    float tx = x + i * (tabW + gap);
    boolean hover = mouseX > tx && mouseX < tx + tabW && mouseY > y && mouseY < y + tabH;
    boolean active = (currentSettingsTab == i);
    
    fill(active ? theme.accent : hover ? theme.hover : theme.secondary);
    stroke(active ? theme.accent : theme.border);
    strokeWeight(1);
    rect(tx, y, tabW, tabH, 6, 6, 0, 0);
    
    fill(active ? theme.primary : theme.text);
    textFont(fontBold);
    textSize(11);
    textAlign(CENTER, CENTER);
    text(tabs[i], tx + tabW/2, y + tabH/2);
  }
}

void drawAntennaSettings(float px, float py) {
  float fieldW = 140, fieldH = 26, rowH = 34;
  
  fill(theme.textDim);
  textFont(fontBold);
  textSize(10);
  textAlign(LEFT, CENTER);
  text("ANTENNA", px + 30, py + 5);
  text("NOME", px + 100, py + 5);
  text("PIN", px + 260, py + 5);
  text("DIR", px + 320, py + 5);
  
  for (int i = 0; i < 6; i++) {
    float rowY = py + 25 + i * rowH;
    
    fill(theme.accent);
    textFont(fontBold);
    textSize(11);
    textAlign(LEFT, CENTER);
    text("ANT " + (i + 1), px + 30, rowY + fieldH/2);
    
    drawTextField(px + 100, rowY, fieldW, fieldH, i, tempAntennaNames[i], "Nome");
    drawTextField(px + 250, rowY, 50, fieldH, i + 10, str(tempAntennaPins[i]), "Pin");
    drawCheckbox(px + 320, rowY + 5, tempAntennaDirective[i], i);
  }
}

void drawTextField(float x, float y, float w, float h, int idx, String value, String placeholder) {
  boolean editing = (editingField == idx);
  boolean hover = mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h;
  
  fill(editing ? theme.panelLight : hover ? theme.hover : theme.panel);
  stroke(editing ? theme.accent : theme.border);
  strokeWeight(editing ? 2 : 1);
  rect(x, y, w, h, 4);
  
  String display = editing ? inputBuffer : value;
  fill(display. length() == 0 ? theme.textDim : theme.text);
  if (display.length() == 0) display = placeholder;
  
  textFont(fontRegular);
  textSize(10);
  textAlign(LEFT, CENTER);
  if (display.length() > 16) display = display.substring(0, 14) + ".. ";
  text(display, x + 8, y + h/2);
  
  if (editing && millis() % 1000 < 500) {
    float tw = textWidth(inputBuffer);
    stroke(theme.accent);
    strokeWeight(1);
    line(x + 8 + tw, y + 5, x + 8 + tw, y + h - 5);
  }
}

void drawCheckbox(float x, float y, boolean checked, int idx) {
  float size = 18;
  boolean hover = mouseX > x && mouseX < x + size && mouseY > y && mouseY < y + size;
  
  fill(checked ? theme.accent : hover ? theme.hover : theme.panel);
  stroke(checked ? theme.accent : theme.border);
  strokeWeight(1);
  rect(x, y, size, size, 4);
  
  if (checked) {
    stroke(theme.primary);
    strokeWeight(2);
    noFill();
    line(x + 4, y + size/2, x + size/2 - 1, y + size - 5);
    line(x + size/2 - 1, y + size - 5, x + size - 4, y + 4);
  }
}

void drawConnectionSettings(float px, float py) {
  fill(theme.text);
  textFont(fontBold);
  textSize(12);
  textAlign(LEFT, TOP);
  text("PORTE DISPONIBILI:", px + 30, py);
  
  if (availablePorts != null && availablePorts.length > 0) {
    float btnW = 100, btnH = 30, gap = 10;
    
    for (int i = 0; i < min(availablePorts.length, 6); i++) {
      float bx = px + 30 + (i % 3) * (btnW + gap);
      float by = py + 30 + (i / 3) * (btnH + gap);
      
      boolean selected = availablePorts[i]. equals(comPort);
      boolean hover = mouseX > bx && mouseX < bx + btnW && mouseY > by && mouseY < by + btnH;
      
      fill(selected ? theme.accent : hover ? theme.hover : theme.secondary);
      stroke(selected ? theme.accent : theme.border);
      strokeWeight(selected ? 2 : 1);
      rect(bx, by, btnW, btnH, 6);
      
      fill(selected ?  theme.primary : theme.text);
      textFont(fontRegular);
      textSize(10);
      textAlign(CENTER, CENTER);
      text(availablePorts[i], bx + btnW/2, by + btnH/2);
    }
  } else {
    fill(theme.textDim);
    textFont(fontRegular);
    textSize(11);
    text("Nessuna porta trovata", px + 30, py + 35);
  }
  
  float btnY = py + 110;
  
  color connBtnColor = arduinoConnected ? theme.error : theme.success;
  String connBtnText = arduinoConnected ? "DISCONNETTI" : "CONNETTI";
  boolean connHover = mouseX > px + 30 && mouseX < px + 180 && mouseY > btnY && mouseY < btnY + 38;
  
  fill(connHover ? lerpColor(connBtnColor, theme.text, 0.2) : connBtnColor);
  stroke(connBtnColor);
  rect(px + 30, btnY, 150, 38, 8);
  
  fill(theme.primary);
  textFont(fontBold);
  textSize(11);
  textAlign(CENTER, CENTER);
  text(connBtnText, px + 105, btnY + 19);
  
  boolean scanHover = mouseX > px + 200 && mouseX < px + 310 && mouseY > btnY && mouseY < btnY + 38;
  fill(scanHover ? lerpColor(theme.warning, theme.text, 0.2) : theme.warning);
  stroke(theme.warning);
  rect(px + 200, btnY, 110, 38, 8);
  
  fill(theme.primary);
  text("SCAN", px + 255, btnY + 19);
  
  // Auto-connect checkbox
  float autoY = btnY + 55;
  float checkX = px + 30;
  float checkSize = 18;
  
  fill(autoConnect ? theme.accent : theme. panel);
  stroke(autoConnect ?  theme.accent : theme.border);
  strokeWeight(1);
  rect(checkX, autoY, checkSize, checkSize, 4);
  
  if (autoConnect) {
    stroke(theme.primary);
    strokeWeight(2);
    noFill();
    line(checkX + 4, autoY + checkSize/2, checkX + checkSize/2 - 1, autoY + checkSize - 5);
    line(checkX + checkSize/2 - 1, autoY + checkSize - 5, checkX + checkSize - 4, autoY + 4);
  }
  
  fill(theme.text);
  textFont(fontRegular);
  textSize(11);
  textAlign(LEFT, CENTER);
  text("Connessione automatica all'avvio", checkX + checkSize + 8, autoY + checkSize/2);
  
  float statusY = autoY + 35;
  fill(theme.textDim);
  textSize(10);
  textAlign(LEFT, TOP);
  text("Status: " + (arduinoConnected ? "Connesso a " + comPort : "Non connesso"), px + 30, statusY);
  text("Arduino Leonardo richiede 3 secondi dopo la connessione", px + 30, statusY + 15);
}

void drawSystemSettings(float px, float py) {
  fill(theme.text);
  textFont(fontBold);
  textSize(12);
  textAlign(LEFT, CENTER);
  text("OPZIONI:", px + 30, py + 15);
  
  drawCheckbox(px + 30, py + 40, debugMode, 100);
  fill(theme.text);
  textFont(fontRegular);
  textSize(11);
  textAlign(LEFT, CENTER);
  text("Modalità Debug", px + 55, py + 49);
  
  float btnY = py + 90;
  boolean resetHover = mouseX > px + 30 && mouseX < px + 150 && mouseY > btnY && mouseY < btnY + 35;
  
  fill(resetHover ? lerpColor(theme.warning, theme.text, 0.2) : theme.warning);
  stroke(theme.warning);
  rect(px + 30, btnY, 120, 35, 8);
  
  fill(theme.primary);
  textFont(fontBold);
  textSize(11);
  textAlign(CENTER, CENTER);
  text("RESET DEFAULT", px + 90, btnY + 17);
  
  fill(theme.textDim);
  textFont(fontRegular);
  textSize(10);
  textAlign(LEFT, TOP);
  text("Versione: " + APP_VERSION, px + 30, btnY + 55);
  text("Relè antenne: Pin 4-9", px + 30, btnY + 70);
  text("Relè rotore: CW=A0, CCW=A1, PWR=A3", px + 30, btnY + 85);
  text("LED: CW=Pin16, CCW=Pin10 (40% PWM)", px + 30, btnY + 100);
}

void drawSettingsButtons(float px, float py) {
  float btnW = 100, btnH = 35;
  float centerX = px + 360;
  
  boolean saveHover = mouseX > centerX - btnW - 10 && mouseX < centerX - 10 && mouseY > py && mouseY < py + btnH;
  fill(saveHover ? lerpColor(theme.success, theme.text, 0.2) : theme.success);
  stroke(theme.success);
  rect(centerX - btnW - 10, py, btnW, btnH, 8);
  
  fill(theme.primary);
  textFont(fontBold);
  textSize(11);
  textAlign(CENTER, CENTER);
  text("SALVA", centerX - btnW/2 - 10, py + btnH/2);
  
  boolean cancelHover = mouseX > centerX + 10 && mouseX < centerX + btnW + 10 && mouseY > py && mouseY < py + btnH;
  fill(cancelHover ? lerpColor(theme.warning, theme.text, 0.2) : theme.warning);
  stroke(theme.warning);
  rect(centerX + 10, py, btnW, btnH, 8);
  
  fill(theme.primary);
  text("ANNULLA", centerX + btnW/2 + 10, py + btnH/2);
}

// ═══════════════════════════════════════════════════════════════════════════
//  SCHERMATA DEBUG
// ═══════════════════════════════════════════════════════════════════════════

void drawDebugScreen() {
  float px = 40, py = 60, pw = 720, ph = 400;
  drawPanel(px, py, pw, ph, "DEBUG CONSOLE", false);
  
  fill(theme.primary);
  noStroke();
  rect(px + 15, py + 50, pw - 30, ph - 100, 6);
  
  textFont(fontMono);
  textSize(10);
  textAlign(LEFT, TOP);
  
  int maxLines = (int)((ph - 120) / 14);
  int startIdx = max(0, debugLog.size() - maxLines);
  
  for (int i = startIdx; i < debugLog.size(); i++) {
    String line = debugLog.get(i);
    color lineColor = theme.success;
    if (line.contains("ERRORE") || line.contains("ERROR")) lineColor = theme.error;
    else if (line.contains("WARNING") || line.contains("HALT")) lineColor = theme.warning;
    else if (line.contains("TX:") || line.contains("RX:")) lineColor = theme.accent;
    
    fill(lineColor);
    text(line, px + 25, py + 60 + (i - startIdx) * 14);
  }
  
  float btnX = px + pw - 90, btnY = py + ph - 40;
  boolean clearHover = mouseX > btnX && mouseX < btnX + 70 && mouseY > btnY && mouseY < btnY + 28;
  
  fill(clearHover ? lerpColor(theme.warning, theme.text, 0.2) : theme.warning);
  stroke(theme.warning);
  rect(btnX, btnY, 70, 28, 6);
  
  fill(theme.primary);
  textFont(fontBold);
  textSize(10);
  textAlign(CENTER, CENTER);
  text("CLEAR", btnX + 35, btnY + 14);
}

// ═══════════════════════════════════════════════════════════════════════════
//  UI COMPONENTS
// ═══════════════════════════════════════════════════════════════════════════

void drawPanel(float x, float y, float w, float h, String title, boolean showGlow) {
  fill(0, 0, 0, 50);
  noStroke();
  rect(x + 5, y + 5, w, h, 15);
  
  fill(theme.panel);
  stroke(theme.border);
  strokeWeight(1);
  rect(x, y, w, h, 15);
  
  if (showGlow) {
    stroke(theme.accent, 80);
    strokeWeight(2);
    noFill();
    arc(x + w/2, y, w - 40, 30, PI, TWO_PI);
  }
  
  fill(theme.accent);
  textFont(fontLarge);
  textSize(15);
  textAlign(LEFT, TOP);
  text(title, x + 20, y + 15);
  
  stroke(theme.border);
  strokeWeight(1);
  line(x + 20, y + 38, x + w - 20, y + 38);
}

void drawTopBar() {
  fill(theme.secondary);
  noStroke();
  rect(0, 0, width, 45);
  
  stroke(theme.border);
  strokeWeight(1);
  line(0, 45, width, 45);
  
  fill(theme.accent);
  textFont(fontLarge);
  textSize(16);
  textAlign(LEFT, CENTER);
  text(APP_NAME, 20, 22);
  
  fill(theme.textDim);
  textFont(fontRegular);
  textSize(9);
  text("v" + APP_VERSION, 195, 22);
  
  float ledX = width - 180;
  float pulse = 0.5 + 0.5 * sin(millis() * 0.005);
  
  fill(arduinoConnected ? lerpColor(theme.success, color(255), pulse * 0.3) : theme. error);
  noStroke();
  ellipse(ledX, 22, 10, 10);
  
  fill(theme.text);
  textFont(fontRegular);
  textSize(10);
  textAlign(LEFT, CENTER);
  text(arduinoConnected ? "ONLINE" : "OFFLINE", ledX + 12, 22);
  
  drawPowerSwitch(width - 75, 12);
}

void drawPowerSwitch(float x, float y) {
  float w = 55, h = 22;
  
  fill(systemOn ? theme.success : theme. disabled);
  stroke(systemOn ? theme.success : theme.border);
  strokeWeight(1);
  rect(x, y, w, h, 11);
  
  float handleX = systemOn ? x + w - 18 : x + 3;
  fill(255);
  noStroke();
  ellipse(handleX + 7, y + 11, 16, 16);
  
  if (systemOn) {
    fill(theme.success, 60);
    ellipse(handleX + 7, y + 11, 22, 22);
  }
  
  fill(theme. textDim);
  textFont(fontBold);
  textSize(8);
  textAlign(RIGHT, CENTER);
  text("PWR", x - 8, y + 11);
}

void drawNavigationBar() {
  String[] items = {"CONTROLLO", "IMPOSTAZIONI", "DEBUG"};
  float barW = 320, barH = 40;
  float startX = (width - barW) / 2;
  float startY = height - 48;
  float itemW = barW / items.length - 8;
  
  for (int i = 0; i < items.length; i++) {
    float ix = startX + i * (itemW + 8);
    boolean hover = mouseX > ix && mouseX < ix + itemW && mouseY > startY && mouseY < startY + barH;
    boolean active = (currentScreen == i);
    
    buttonHover[23 + i] = hover;
    
    pushMatrix();
    if (hover && ! active) translate(0, -2 * buttonAnim[23 + i]);
    
    fill(0, 0, 0, 60);
    noStroke();
    rect(ix + 2, startY + 2, itemW, barH, 8);
    
    fill(active ? theme.accent : hover ? theme.hover : theme.secondary);
    stroke(active ? theme.accent : theme.border);
    strokeWeight(active ? 2 : 1);
    rect(ix, startY, itemW, barH, 8);
    
    fill(active ? theme.primary : theme.text);
    textFont(fontBold);
    textSize(10);
    textAlign(CENTER, CENTER);
    text(items[i], ix + itemW/2, startY + barH/2);
    
    popMatrix();
  }
}

void drawLeonardoWaitOverlay() {
  fill(0, 0, 0, 200);
  rect(0, 0, width, height);
  
  float bw = 320, bh = 140;
  float bx = (width - bw) / 2, by = (height - bh) / 2;
  
  fill(theme.panel);
  stroke(theme.accent);
  strokeWeight(2);
  rect(bx, by, bw, bh, 15);
  
  fill(theme.accent);
  textFont(fontLarge);
  textSize(14);
  textAlign(CENTER, CENTER);
  text("Connessione Arduino Leonardo", width/2, by + 30);
  
  long elapsed = millis() - leonardoWaitStart;
  float progress = constrain((float)elapsed / LEONARDO_WAIT_TIME, 0, 1);
  
  float pbW = 250, pbH = 20;
  float pbX = (width - pbW) / 2, pbY = by + 60;
  
  fill(theme.secondary);
  noStroke();
  rect(pbX, pbY, pbW, pbH, 5);
  
  fill(theme.accent);
  rect(pbX, pbY, pbW * progress, pbH, 5);
  
  fill(theme. text);
  textFont(fontBold);
  textSize(11);
  text(int(progress * 100) + "%", width/2, pbY + pbH/2);
  
  fill(theme.textDim);
  textFont(fontRegular);
  textSize(10);
  text("Attendere reset Leonardo...", width/2, by + 105);
}

// ═══════════════════════════════════════════════════════════════════════════
//  EVENT HANDLERS
// ═══════════════════════════════════════════════════════════════════════════

void mousePressed() {
  if (showingSplash) { showingSplash = false; return; }
  if (waitingForLeonardo) return;
  
  if (currentScreen == 0) {
    checkAntennaClick();
    checkRotorPowerClick();
    checkRotorButtonsPressed();
  } else if (currentScreen == 1) {
    checkSettingsClick();
  } else if (currentScreen == 2) {
    checkDebugClick();
  }
  
  checkTopBarClick();
  checkNavigationClick();
}

void mouseReleased() {
  if (showingSplash || waitingForLeonardo) return;
  
  if (cwButtonPressed || ccwButtonPressed) {
    deactivateRotorRelays();
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  ROTOR POWER CONTROL (ON/OFF)
// ═══════════════════════════════════════════════════════════════════════════

void checkRotorPowerClick() {
  if (! systemOn) return;
  
  float px = 330, py = 55;
  float x = px + 20, y = py + 45;
  float w = 120, h = 30;
  
  if (mouseX > x && mouseX < x + w && mouseY > y && mouseY < y + h) {
    toggleRotorPower();
  }
}

void toggleRotorPower() {
  rotorPowerOn = !rotorPowerOn;
  
  if (! rotorPowerOn) {
    // Spegni movimento quando si spegne il rotore
    deactivateRotorRelays();
  }
  
  addDebugLog("Rotor Power: " + (rotorPowerOn ? "ON" : "OFF"));
  sendSerialCommand("ROTOR_PWR:" + (rotorPowerOn ?  "1" : "0"));
  addNotification("Rotor " + (rotorPowerOn ?  "ON" : "OFF"), rotorPowerOn ? SUCCESS : WARNING);
}

// ═══════════════════════════════════════════════════════════════════════════
//  PULSANTI NA - CONTROLLO RELÈ ROTORE
// ═══════════════════════════════════════════════════════════════════════════

void deactivateRotorRelays() {
  if (cwButtonPressed) {
    cwButtonPressed = false;
    rotorCW = false;
    addDebugLog("CW: Rilasciato → Relè A0 OFF");
    sendSerialCommand("CW:0");
  }
  
  if (ccwButtonPressed) {
    ccwButtonPressed = false;
    rotorCCW = false;
    addDebugLog("CCW: Rilasciato → Relè A1 OFF");
    sendSerialCommand("CCW:0");
  }
}

void activateCWRelay() {
  if (!cwButtonPressed && systemOn && rotorPowerOn) {
    if (ccwButtonPressed) {
      ccwButtonPressed = false;
      rotorCCW = false;
      sendSerialCommand("CCW:0");
    }
    
    cwButtonPressed = true;
    rotorCW = true;
    addDebugLog("CW: Premuto → Relè A0 ON");
    sendSerialCommand("CW:1");
  }
}

void activateCCWRelay() {
  if (!ccwButtonPressed && systemOn && rotorPowerOn) {
    if (cwButtonPressed) {
      cwButtonPressed = false;
      rotorCW = false;
      sendSerialCommand("CW:0");
    }
    
    ccwButtonPressed = true;
    rotorCCW = true;
    addDebugLog("CCW: Premuto → Relè A1 ON");
    sendSerialCommand("CCW:1");
  }
}

void sendSerialCommand(String cmd) {
  if (arduinoConnected && arduino != null) {
    try {
      arduino.write(cmd + "\n");
      addDebugLog("TX: " + cmd);
    } catch (Exception e) {
      addDebugLog("ERRORE TX: " + e.getMessage());
      arduinoConnected = false;
    }
  }
}

void checkRotorButtonsPressed() {
  if (!systemOn || !rotorPowerOn) return;
  
  float centerX = mapCenterX;
  float btnY = mapCenterY + 155;
  float btnW = 75, gap = 10;
  
  // CCW
  float ccwX = centerX - btnW - gap/2 - btnW/2;
  if (mouseX > ccwX && mouseX < ccwX + btnW && mouseY > btnY && mouseY < btnY + 38) {
    activateCCWRelay();
    return;
  }
  
  // HALT
  float haltX = centerX - btnW/2;
  if (mouseX > haltX && mouseX < haltX + btnW && mouseY > btnY && mouseY < btnY + 38) {
    emergencyHalt();
    return;
  }
  
  // CW
  float cwX = centerX + gap/2 + btnW/2;
  if (mouseX > cwX && mouseX < cwX + btnW && mouseY > btnY && mouseY < btnY + 38) {
    activateCWRelay();
    return;
  }
}

void emergencyHalt() {
  cwButtonPressed = false;
  ccwButtonPressed = false;
  rotorCW = false;
  rotorCCW = false;
  
  addDebugLog("! !! EMERGENCY HALT !!!");
  sendSerialCommand("CW:0");
  sendSerialCommand("CCW:0");
  sendSerialCommand("HALT:1");
  addNotification("EMERGENCY HALT!", ERROR);
}

// ═══════════════════════════════════════════════════════════════════════════
//  CONTROLLI ANTENNE
// ═══════════════════════════════════════════════════════════════════════════

void checkAntennaClick() {
  if (!systemOn) return;
  
  float px = 20, py = 55;
  float startX = px + 15, startY = py + 70;
  float btnW = 125, btnH = 52, gapX = 10, gapY = 8;
  
  for (int i = 0; i < 6; i++) {
    int col = i % 2, row = i / 2;
    float bx = startX + col * (btnW + gapX);
    float by = startY + row * (btnH + gapY);
    
    if (mouseX > bx && mouseX < bx + btnW && mouseY > by && mouseY < by + btnH) {
      selectAntenna(i);
      break;
    }
  }
}

void selectAntenna(int idx) {
  if (idx < 0 || idx >= 6) return;
  
  if (selectedAntenna == idx) {
    selectedAntenna = -1;
    for (int i = 0; i < 6; i++) antennaStates[i] = false;
    addDebugLog("Antenna " + antennaNames[idx] + " OFF");
  } else {
    for (int i = 0; i < 6; i++) antennaStates[i] = false;
    selectedAntenna = idx;
    antennaStates[idx] = true;
    addDebugLog("Antenna " + antennaNames[idx] + " ON");
  }
  
  sendSerialCommand("ANT:" + idx + ":" + antennaPins[idx]);
}

// ═══════════════════════════════════════════════════════════════════════════
//  CONTROLLI IMPOSTAZIONI
// ═══════════════════════════════════════════════════════════════════════════

void checkSettingsClick() {
  float px = 40, py = 60;
  
  // Tabs
  float tabY = py + 45, tabW = 110, tabH = 32, gap = 10;
  for (int i = 0; i < 3; i++) {
    float tx = px + 20 + i * (tabW + gap);
    if (mouseX > tx && mouseX < tx + tabW && mouseY > tabY && mouseY < tabY + tabH) {
      currentSettingsTab = i;
      editingField = -1;
      return;
    }
  }
  
  if (currentSettingsTab == 0) checkAntennaSettingsClick(px, py + 90);
  else if (currentSettingsTab == 1) checkConnectionSettingsClick(px, py + 90);
  else if (currentSettingsTab == 2) checkSystemSettingsClick(px, py + 90);
  
  // Salva/Annulla
  float btnY = py + 350, centerX = px + 360, btnW = 100, btnH = 35;
  
  if (mouseX > centerX - btnW - 10 && mouseX < centerX - 10 && mouseY > btnY && mouseY < btnY + btnH) {
    saveSettings();
    return;
  }
  
  if (mouseX > centerX + 10 && mouseX < centerX + btnW + 10 && mouseY > btnY && mouseY < btnY + btnH) {
    cancelSettings();
    return;
  }
}

void checkAntennaSettingsClick(float px, float py) {
  float fieldW = 140, fieldH = 26, rowH = 34;
  
  for (int i = 0; i < 6; i++) {
    float rowY = py + 25 + i * rowH;
    
    if (mouseX > px + 100 && mouseX < px + 100 + fieldW && mouseY > rowY && mouseY < rowY + fieldH) {
      editingField = i;
      inputBuffer = tempAntennaNames[i];
      return;
    }
    
    if (mouseX > px + 250 && mouseX < px + 300 && mouseY > rowY && mouseY < rowY + fieldH) {
      editingField = i + 10;
      inputBuffer = str(tempAntennaPins[i]);
      return;
    }
    
    if (mouseX > px + 320 && mouseX < px + 338 && mouseY > rowY + 5 && mouseY < rowY + 23) {
      tempAntennaDirective[i] = ! tempAntennaDirective[i];
      return;
    }
  }
  
  editingField = -1;
}

void checkConnectionSettingsClick(float px, float py) {
  if (availablePorts != null) {
    float btnW = 100, btnH = 30, gap = 10;
    
    for (int i = 0; i < min(availablePorts.length, 6); i++) {
      float bx = px + 30 + (i % 3) * (btnW + gap);
      float by = py + 30 + (i / 3) * (btnH + gap);
      
      if (mouseX > bx && mouseX < bx + btnW && mouseY > by && mouseY < by + btnH) {
        comPort = availablePorts[i];
        tempComPort = comPort;
        addDebugLog("Porta: " + comPort);
        return;
      }
    }
  }
  
  float btnY = py + 110;
  
  if (mouseX > px + 30 && mouseX < px + 180 && mouseY > btnY && mouseY < btnY + 38) {
    if (arduinoConnected) disconnectArduino();
    else connectArduino();
    return;
  }
  
  if (mouseX > px + 200 && mouseX < px + 310 && mouseY > btnY && mouseY < btnY + 38) {
    scanSerialPorts();
    addNotification("Porte scansionate", INFO);
    return;
  }
  
  // Auto-connect
  float autoY = btnY + 55, checkX = px + 30, checkSize = 18;
  if (mouseX > checkX && mouseX < checkX + checkSize + 150 && mouseY > autoY && mouseY < autoY + checkSize) {
    autoConnect = !autoConnect;
    addDebugLog("Auto-connect: " + (autoConnect ? "ON" : "OFF"));
    addNotification("Auto-connect " + (autoConnect ? "attivato" : "disattivato"), autoConnect ? SUCCESS : INFO);
    return;
  }
}

void checkSystemSettingsClick(float px, float py) {
  if (mouseX > px + 30 && mouseX < px + 48 && mouseY > py + 40 && mouseY < py + 58) {
    debugMode = !debugMode;
    addDebugLog("Debug: " + (debugMode ? "ON" : "OFF"));
    return;
  }
  
  float btnY = py + 90;
  if (mouseX > px + 30 && mouseX < px + 150 && mouseY > btnY && mouseY < btnY + 35) {
    resetToDefaults();
    addNotification("Reset completato", WARNING);
    return;
  }
}

void checkDebugClick() {
  float px = 40, py = 60, pw = 720, ph = 400;
  float btnX = px + pw - 90, btnY = py + ph - 40;
  
  if (mouseX > btnX && mouseX < btnX + 70 && mouseY > btnY && mouseY < btnY + 28) {
    debugLog. clear();
    addDebugLog("Log cancellato");
  }
}

void checkTopBarClick() {
  float switchX = width - 75, switchY = 12, switchW = 55, switchH = 22;
  
  if (mouseX > switchX && mouseX < switchX + switchW && mouseY > switchY && mouseY < switchY + switchH) {
    systemOn = !systemOn;
    addDebugLog("Sistema: " + (systemOn ? "ON" : "OFF"));
    
    if (! systemOn) {
      emergencyHalt();
      rotorPowerOn = false;
      selectedAntenna = -1;
      for (int i = 0; i < 6; i++) antennaStates[i] = false;
    }
    
    sendSerialCommand("PWR:" + (systemOn ? "1" : "0"));
  }
}

void checkNavigationClick() {
  float barW = 320, barH = 40;
  float startX = (width - barW) / 2, startY = height - 48;
  float itemW = barW / 3 - 8;
  
  for (int i = 0; i < 3; i++) {
    float ix = startX + i * (itemW + 8);
    
    if (mouseX > ix && mouseX < ix + itemW && mouseY > startY && mouseY < startY + barH) {
      if (currentScreen != i) {
        targetScreen = i;
        transitioning = true;
        screenTransition = 0;
        editingField = -1;
      }
      return;
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  KEYBOARD
// ═══════════════════════════════════════════════════════════════════════════

void keyPressed() {
  if (showingSplash) { showingSplash = false; return; }
  if (waitingForLeonardo) return;
  
  if (editingField >= 0) {
    if (key == ENTER || key == RETURN) {
      editingField = -1;
    } else if (key == BACKSPACE) {
      if (inputBuffer.length() > 0) {
        inputBuffer = inputBuffer.substring(0, inputBuffer.length() - 1);
        updateFieldFromBuffer();
      }
    } else if (key == ESC) {
      editingField = -1;
      key = 0;
    } else if (key >= 32 && key <= 126) {
      if (editingField < 6 && inputBuffer.length() < 20) {
        inputBuffer += key;
        tempAntennaNames[editingField] = inputBuffer;
      } else if (editingField >= 10 && editingField < 16 && key >= '0' && key <= '9' && inputBuffer.length() < 2) {
        inputBuffer += key;
        tempAntennaPins[editingField - 10] = int(inputBuffer);
      }
    }
    return;
  }
  
  if (key == 'h' || key == 'H') { emergencyHalt(); }
  if (key == 'c' || key == 'C') { if (arduinoConnected) disconnectArduino(); else connectArduino(); }
  if (key == 'r' || key == 'R') { scanSerialPorts(); addNotification("Porte scansionate", INFO); }
  if (key == 'd' || key == 'D') { debugMode = !debugMode; }
  if (key == 'p' || key == 'P') { if (systemOn) toggleRotorPower(); }
  if (systemOn && key >= '1' && key <= '6') { selectAntenna(key - '1'); }
  if (key == ESC) { if (currentScreen != 0) { targetScreen = 0; transitioning = true; screenTransition = 0; } key = 0; }
}

void updateFieldFromBuffer() {
  if (editingField >= 0 && editingField < 6) {
    tempAntennaNames[editingField] = inputBuffer;
  } else if (editingField >= 10 && editingField < 16 && inputBuffer.length() > 0) {
    tempAntennaPins[editingField - 10] = int(inputBuffer);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  SETTINGS
// ═══════════════════════════════════════════════════════════════════════════

void saveSettings() {
  arrayCopy(tempAntennaNames, antennaNames);
  arrayCopy(tempAntennaPins, antennaPins);
  arrayCopy(tempAntennaDirective, antennaDirective);
  comPort = tempComPort;
  settings.saveSettings();
  addNotification("Impostazioni salvate", SUCCESS);
  targetScreen = 0; transitioning = true; screenTransition = 0;
}

void cancelSettings() {
  arrayCopy(antennaNames, tempAntennaNames);
  arrayCopy(antennaPins, tempAntennaPins);
  arrayCopy(antennaDirective, tempAntennaDirective);
  tempComPort = comPort;
  editingField = -1;
  targetScreen = 0; transitioning = true; screenTransition = 0;
}

void resetToDefaults() {
  for (int i = 0; i < 6; i++) {
    tempAntennaNames[i] = defaultAntennaNames[i];
    tempAntennaPins[i] = i + 4;
    tempAntennaDirective[i] = defaultAntennaDirective[i];
  }
  tempComPort = "COM4";
  addDebugLog("Reset ai valori predefiniti");
}

// ═══════════════════════════════════════════════════════════════════════════
//  SERIAL COMMUNICATION
// ═══════════════════════════════════════════════════════════════════════════

void connectArduino() {
  if (arduinoConnected) return;
  
  addDebugLog("Connessione a " + comPort + "...");
  
  try {
    if (arduino != null) { try { arduino.stop(); } catch (Exception e) {} arduino = null; }
    
    boolean found = false;
    for (String p : Serial.list()) { if (p.equals(comPort)) { found = true; break; } }
    
    if (! found) {
      addNotification("Porta " + comPort + " non trovata", ERROR);
      return;
    }
    
    arduino = new Serial(this, comPort, baudRate);
    waitingForLeonardo = true;
    leonardoWaitStart = millis();
    addDebugLog("Attesa reset Leonardo.. .");
    
  } catch (Exception e) {
    addNotification("Errore connessione", ERROR);
    addDebugLog("ERRORE: " + e.getMessage());
    arduinoConnected = false;
    arduino = null;
  }
}

void finishLeonardoConnection() {
  try {
    if (arduino != null) {
      arduino.bufferUntil('\n');
      arduinoConnected = true;
      addNotification("Connesso a " + comPort, SUCCESS);
      addDebugLog("Leonardo connesso!");
      delay(100);
      sendSerialCommand("PWR:" + (systemOn ? "1" : "0"));
    }
  } catch (Exception e) {
    addNotification("Errore connessione", ERROR);
    addDebugLog("ERRORE: " + e. getMessage());
    arduinoConnected = false;
    arduino = null;
  }
}

void disconnectArduino() {
  addDebugLog("Disconnessione.. .");
  
  try {
    if (arduinoConnected && arduino != null) {
      sendSerialCommand("CW:0");
      sendSerialCommand("CCW:0");
      sendSerialCommand("ROTOR_PWR:0");
      sendSerialCommand("PWR:0");
      delay(100);
    }
    
    if (arduino != null) { arduino.stop(); arduino = null; }
    
    arduinoConnected = false;
    rotorCW = false; rotorCCW = false;
    cwButtonPressed = false; ccwButtonPressed = false;
    rotorPowerOn = false;
    
    addNotification("Disconnesso", WARNING);
    addDebugLog("Arduino disconnesso");
    
  } catch (Exception e) {
    addDebugLog("ERRORE: " + e.getMessage());
    arduino = null;
    arduinoConnected = false;
  }
}

void serialEvent(Serial p) {
  if (p != arduino || arduino == null) return;
  
  try {
    String data = p.readStringUntil('\n');
    if (data == null || data.length() == 0) return;
    
    data = data.trim();
    if (data.length() == 0) return;
    
    addDebugLog("RX: " + data);
    
    if (data.startsWith("AZI:")) {
      try { currentAzimuth = Float.parseFloat(data.substring(4)); } catch (Exception e) {}
    }
    else if (data.startsWith("ANT:")) {
      try {
        int idx = Integer.parseInt(data.substring(4));
        if (idx >= -1 && idx < 6) {
          selectedAntenna = idx;
          for (int i = 0; i < 6; i++) antennaStates[i] = (i == idx);
        }
      } catch (Exception e) {}
    }
    else if (data.startsWith("ROTOR:")) {
      String status = data.substring(6);
      if (status.equals("CW")) { rotorCW = true; rotorCCW = false; }
      else if (status.equals("CCW")) { rotorCW = false; rotorCCW = true; }
      else { rotorCW = false; rotorCCW = false; cwButtonPressed = false; ccwButtonPressed = false; }
    }
    else if (data.startsWith("ROTOR_PWR:")) {
      rotorPowerOn = data.substring(10).equals("ON");
    }
    else if (data.startsWith("STATUS:") && (data.contains("stopped") || data.contains("halt") || data.contains("timeout"))) {
      rotorCW = false; rotorCCW = false;
      cwButtonPressed = false; ccwButtonPressed = false;
    }
    
  } catch (Exception e) {
    addDebugLog("ERRORE RX: " + e.getMessage());
  }
}

// ═══════════════════════════════════════════════════════════════════════════
//  EXIT
// ═══════════════════════════════════════════════════════════════════════════

void exit() {
  addDebugLog("Chiusura.. .");
  
  if (arduinoConnected && arduino != null) {
    try {
      arduino.write("CW:0\n");
      arduino.write("CCW:0\n");
      arduino.write("ROTOR_PWR:0\n");
      arduino.write("PWR:0\n");
      delay(150);
      arduino.stop();
    } catch (Exception e) {}
  }
  
  super.exit();
}
