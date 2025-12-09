#include <WiFi.h>
#include <WiFiManager.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Preferences.h>

// OLED Display settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Preferences for storing settings
Preferences prefs;

// Global variables
float goldPrice = 0.0;
float previousPrice = 0.0;
String currency = "USD";
String metalSymbol = "XAU"; // Gold symbol
unsigned long updateInterval = 300000; // 5 minutes default
unsigned long lastUpdate = 0;
bool wifiConnected = false;
String lastUpdateTime = "";
String lastUpdateDate = "";
int timezoneOffset = -5;

// Custom parameters for WiFiManager
WiFiManagerParameter custom_currency("currency", "Currency (USD/EUR/GBP/etc)", "USD", 4);
WiFiManagerParameter custom_interval("interval", "Update Interval (seconds)", "300", 6);
WiFiManagerParameter custom_metal("metal", "Metal Symbol (XAU/XAG/XPT)", "XAU", 4);
WiFiManagerParameter custom_timezone("timezone", "Timezone offset (hours from UTC)", "-5", 4);

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n\n=== ESP32-C3 Gold Price Monitor ===");
  
  // Initialize OLED
  if(!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println(F("SSD1306 allocation failed"));
    for(;;);
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Gold-API.com");
  display.println("Price Monitor");
  display.println();
  display.println("Initializing...");
  display.display();
  
  // Initialize preferences
  prefs.begin("gold-monitor", false);
  currency = prefs.getString("currency", "USD");
  metalSymbol = prefs.getString("metal", "XAU");
  updateInterval = prefs.getULong("interval", 300) * 1000;
  timezoneOffset = prefs.getInt("timezone", -5);
  
  // Set initial timezone
  configTime(timezoneOffset * 3600, 0, "pool.ntp.org", "time.nist.gov");
  
  // WiFiManager setup
  WiFiManager wm;
  
  // Uncomment this line to reset WiFi settings for testing
  // wm.resetSettings();
  
  // Set debug output
  wm.setDebugOutput(true);
  
  // Add custom parameters
  wm.addParameter(&custom_currency);
  wm.addParameter(&custom_interval);
  wm.addParameter(&custom_metal);
  wm.addParameter(&custom_timezone);
  
  // Set callback for saving parameters
  wm.setSaveParamsCallback(saveParamsCallback);
  
  // Remove config portal timeout (portal stays open until configured)
  wm.setConfigPortalTimeout(0);
  
  // Set minimum signal quality
  wm.setMinimumSignalQuality(10);
  
  // Display setup instructions
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("WiFi Setup");
  display.println("-------------");
  display.println();
  display.println("Connect to:");
  display.println("GoldPriceMonitor");
  display.println();
  display.println("Then go to:");
  display.println("192.168.4.1");
  display.display();
  
  Serial.println("\n=== WiFi Configuration ===");
  Serial.println("Connect to: GoldPriceMonitor");
  Serial.println("No password required");
  Serial.println("Open browser: 192.168.4.1");
  
  // Start WiFiManager - autoConnect will create AP if no saved credentials
  // No password parameter = open network
  if (!wm.autoConnect("GoldPriceMonitor")) {
    Serial.println("Failed to connect - timeout");
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("WiFi Timeout!");
    display.println("Restarting...");
    display.display();
    delay(3000);
    ESP.restart();
  }
  
  wifiConnected = true;
  Serial.println("\n=== WiFi Connected ===");
  Serial.println("SSID: " + WiFi.SSID());
  Serial.println("IP: " + WiFi.localIP().toString());
  
  // Configure time with saved timezone
  configTime(timezoneOffset * 3600, 0, "pool.ntp.org", "time.nist.gov");
  Serial.println("Waiting for NTP time sync...");
  
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("WiFi Connected!");
  display.println();
  display.println("SSID:");
  display.println(WiFi.SSID());
  display.println();
  display.print("IP: ");
  display.println(WiFi.localIP());
  display.println();
  display.println("Syncing time...");
  display.display();
  
  // Wait for time sync
  int retries = 0;
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo) && retries < 10) {
    delay(1000);
    retries++;
    Serial.print(".");
  }
  Serial.println();
  
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Ready!");
  display.display();
  delay(2000);
  
  // Fetch initial gold price
  fetchGoldPrice();
}

void loop() {
  // Check WiFi connection
  if (WiFi.status() != WL_CONNECTED) {
    wifiConnected = false;
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("WiFi Lost!");
    display.println("Reconnecting...");
    display.display();
    WiFi.reconnect();
    delay(5000);
    return;
  }
  
  wifiConnected = true;
  
  // Update gold price at specified interval
  if (millis() - lastUpdate >= updateInterval) {
    fetchGoldPrice();
  }
  
  // Update display
  updateDisplay();
  
  delay(1000);
}

void fetchGoldPrice() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("Fetching...");
    display.display();
    
    // gold-api.com endpoint
    String url = "https://api.gold-api.com/price/" + metalSymbol;
    
    Serial.println("Fetching: " + url);
    
    http.begin(url);
    http.setTimeout(10000); // 10 second timeout
    
    // Add headers to get timestamp info
    http.addHeader("Accept", "application/json");
    
    int httpCode = http.GET();
    
    Serial.printf("HTTP Code: %d\n", httpCode);
    
    if (httpCode == 200) {
      String payload = http.getString();
      Serial.println("Response: " + payload);
      
      // Parse JSON response
      JsonDocument doc;
      DeserializationError error = deserializeJson(doc, payload);
      
      if (!error) {
        // Check if price exists in response
        if (doc.containsKey("price")) {
          previousPrice = goldPrice;
          goldPrice = doc["price"].as<float>();
          
          // If currency is in response, use it
          if (doc.containsKey("currency")) {
            currency = doc["currency"].as<String>();
          }
          
          lastUpdate = millis();
          Serial.printf("%s Price: %.2f %s/oz\n", metalSymbol.c_str(), goldPrice, currency.c_str());
          
          // Get current date/time
          updateDateTime();
          
        } else if (doc.containsKey(currency)) {
          // Alternative format where currency is the key
          previousPrice = goldPrice;
          goldPrice = doc[currency].as<float>();
          lastUpdate = millis();
          Serial.printf("%s Price: %.2f %s/oz\n", metalSymbol.c_str(), goldPrice, currency.c_str());
          
          // Get current date/time
          updateDateTime();
          
        } else {
          Serial.println("Price field not found in JSON");
          displayError("Invalid response");
        }
      } else {
        Serial.print("JSON parse error: ");
        Serial.println(error.c_str());
        displayError("Parse error");
      }
    } else if (httpCode == 404) {
      Serial.println("Endpoint not found - check metal symbol");
      displayError("Invalid symbol");
    } else if (httpCode > 0) {
      Serial.printf("HTTP error: %d\n", httpCode);
      displayError("HTTP: " + String(httpCode));
    } else {
      Serial.printf("Connection error: %s\n", http.errorToString(httpCode).c_str());
      displayError("Connection fail");
    }
    
    http.end();
  }
}

void displayError(String msg) {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println("Error!");
  display.println();
  display.println(msg);
  display.display();
  delay(2000);
}

void updateDateTime() {
  // Get current time from NTP
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    char timeStr[20];
    char dateStr[20];
    strftime(timeStr, sizeof(timeStr), "%H:%M:%S", &timeinfo);
    strftime(dateStr, sizeof(dateStr), "%m/%d/%Y", &timeinfo);
    lastUpdateTime = String(timeStr);
    lastUpdateDate = String(dateStr);
  } else {
    lastUpdateTime = "";
    lastUpdateDate = "";
  }
}

String formatPrice(float price) {
  // Format price with thousands separator
  int wholePart = (int)price;
  int decimalPart = (int)((price - wholePart) * 100);
  
  // Add thousands separator
  String result = "";
  String wholeStr = String(wholePart);
  int len = wholeStr.length();
  
  for (int i = 0; i < len; i++) {
    if (i > 0 && (len - i) % 3 == 0) {
      result += ",";
    }
    result += wholeStr[i];
  }
  
  // Add decimal part
  if (decimalPart < 10) {
    result += ".0" + String(decimalPart);
  } else {
    result += "." + String(decimalPart);
  }
  
  return result;
}

void updateDisplay() {
  display.clearDisplay();
  
  // Header
  display.setTextSize(1);
  display.setCursor(0, 0);
  
  // Show metal name
  String metalName = "GOLD";
  if (metalSymbol == "XAG") metalName = "SILVER";
  else if (metalSymbol == "XPT") metalName = "PLATINUM";
  else if (metalSymbol == "XPD") metalName = "PALLADIUM";
  
  display.println(metalName + " PRICE");
  display.drawLine(0, 10, SCREEN_WIDTH, 10, SSD1306_WHITE);
  
  // Price display
  if (goldPrice > 0) {
    // Format and display price with currency symbol
    String formattedPrice = formatPrice(goldPrice);
    
    display.setTextSize(2);
    display.setCursor(0, 20);
    
    // Show currency symbol before price for USD, after for others
    if (currency == "USD") {
      display.print("$");
      display.print(formattedPrice);
    } else if (currency == "EUR") {
      display.print(formattedPrice);
      display.print("E");
    } else if (currency == "GBP") {
      display.print("L");
      display.print(formattedPrice);
    } else {
      display.print(formattedPrice);
    }
    
    // Show price change indicator
    display.setTextSize(1);
    if (previousPrice > 0 && previousPrice != goldPrice) {
      display.setCursor(0, 40);
      if (goldPrice > previousPrice) {
        display.print("^UP");
      } else {
        display.print("vDOWN");
      }
    }
  } else {
    display.setTextSize(1);
    display.setCursor(0, 25);
    display.println("Waiting for");
    display.println("data...");
  }
  
  // Bottom section - Last update time on left, SSID on right
  display.setTextSize(1);
  display.setCursor(0, 56);
  
  // Show time since last update
  if (lastUpdate > 0) {
    unsigned long secAgo = (millis() - lastUpdate) / 1000;
    unsigned long minAgo = secAgo / 60;
    unsigned long secRemainder = secAgo % 60;
    
    if (minAgo > 0) {
      display.printf("%lum %lus", minAgo, secRemainder);
    } else {
      display.printf("%lus ago", secAgo);
    }
  } else {
    display.print("--:--");
  }
  
  // Show SSID on bottom right
  String ssidName = WiFi.SSID();
  if (ssidName.length() > 0) {
    // Truncate SSID if too long (max ~8 chars to fit)
    if (ssidName.length() > 8) {
      ssidName = ssidName.substring(0, 8);
    }
    // Calculate position to right-align
    int textWidth = ssidName.length() * 6; // Approximate width
    display.setCursor(SCREEN_WIDTH - textWidth, 56);
    display.print(ssidName);
  }
  
  display.display();
}

void saveParamsCallback() {
  Serial.println("Saving parameters...");
  
  // Get parameters
  String newCurrency = custom_currency.getValue();
  String intervalStr = custom_interval.getValue();
  String newMetal = custom_metal.getValue();
  String timezoneStr = custom_timezone.getValue();
  
  // Validate and save currency
  if (newCurrency.length() > 0) {
    newCurrency.toUpperCase();
    currency = newCurrency;
    prefs.putString("currency", currency);
    Serial.println("Currency: " + currency);
  }
  
  // Validate and save metal symbol
  if (newMetal.length() > 0) {
    newMetal.toUpperCase();
    metalSymbol = newMetal;
    prefs.putString("metal", metalSymbol);
    Serial.println("Metal: " + metalSymbol);
  }
  
  // Validate and save interval (minimum 60 seconds)
  if (intervalStr.length() > 0) {
    unsigned long interval = intervalStr.toInt();
    if (interval >= 60) {
      updateInterval = interval * 1000;
      prefs.putULong("interval", interval);
      Serial.printf("Interval: %lu seconds\n", interval);
    }
  }
  
  // Save timezone offset
  if (timezoneStr.length() > 0) {
    int tzOffset = timezoneStr.toInt();
    timezoneOffset = tzOffset;
    prefs.putInt("timezone", tzOffset);
    Serial.printf("Timezone: %d hours\n", tzOffset);
    
    // Update time configuration with new timezone
    configTime(tzOffset * 3600, 0, "pool.ntp.org", "time.nist.gov");
  }
}