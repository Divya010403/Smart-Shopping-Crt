# Smart-Shopping-Crt
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <SoftwareSerial.h>

// Define the pin connections
#define SS_PIN 10
#define RST_PIN 9
#define BUTTON_DELETE_MODE 2
#define BUTTON_FINALIZE 3
#define SERVO_PIN 4

// Create an instance of the RFID library
MFRC522 rfid(SS_PIN, RST_PIN);

// Create an instance of the LCD (address might be 0x27 or 0x3F based on your LCD module)
LiquidCrystal_I2C lcd(0x3F, 16, 2);

// Create an instance of the Servo
Servo myServo;

// Create a software serial connection for SIM808 on alternative pins
SoftwareSerial sim808(8, 7); // RX, TX

// Define a structure to hold item details
struct Item {
  String uid;
  String name;
  float price;
};

// Create a list of known items with their UIDs, names, and prices
Item items[] = {
  {"1311A8D9", "Tea", 50.0},   // Replace with actual UID
  {"93E869E2", "Milk", 40.0},  // Replace with actual UID
  {"2317F3E1", "Juice", 30.0}, // Replace with actual UID
  {"638243E2", "Coffee", 60.0} // Replace with actual UID
};

float total = 0.0;
bool deleteMode = false; // To track if delete mode is active

void setup() {
  // Initialize serial communication
  Serial.begin(9600);
  
  // Initialize SPI bus for RFID
  SPI.begin();
  rfid.PCD_Init();

  // Initialize the LCD
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Smart Cart");
  delay(2000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Total: 0 Rs");
  
  // Set up the buttons
  pinMode(BUTTON_DELETE_MODE, INPUT_PULLUP); // Use internal pull-up resistor
  pinMode(BUTTON_FINALIZE, INPUT_PULLUP); // Use internal pull-up resistor

  // Initialize the servo
  myServo.attach(SERVO_PIN);
  myServo.write(0); // Set initial position (closed)

  // Initialize SIM808 on alternative pins
  sim808.begin(9600);
  delay(1000);
  sendCommand("AT", 1000); // Check if SIM808 is responding
  sendCommand("AT+CMGF=1", 1000); // Set SMS mode to text
}

void loop() {
  // Check if the Delete Mode button is pressed
  if (digitalRead(BUTTON_DELETE_MODE) == LOW) {
    toggleDeleteMode();
    delay(500); // Debounce delay
  }

  // Check if the Finalize button is pressed
  if (digitalRead(BUTTON_FINALIZE) == LOW) {
    finalizeBill();
    delay(500); // Debounce delay
  }

  // Look for a new card
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    // Get UID string
    String uidString = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      uidString += String(rfid.uid.uidByte[i], HEX);
    }

    if (deleteMode) {
      deleteItem(uidString);
    } else {
      addItem(uidString);
    }

    // Halt the card
    rfid.PICC_HaltA();
  }
}

void toggleDeleteMode() {
  deleteMode = !deleteMode; // Toggle delete mode

  lcd.clear();
  if (deleteMode) {
    lcd.setCursor(0, 0);
    lcd.print("Delete Mode ON");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Delete Mode OFF");
  }
  delay(2000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Total: " + String(total) + " Rs");
}

void addItem(String uid) {
  // Check if the UID matches any of the known items
  bool itemFound = false;
  for (int i = 0; i < sizeof(items) / sizeof(items[0]); i++) {
    if (uid.equalsIgnoreCase(items[i].uid)) {
      // Add the item price to the total
      total += items[i].price;

      // Activate servo to indicate action
      activateServo();

      // Display item name and updated total on the LCD
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Added: " + items[i].name);
      lcd.setCursor(0, 1);
      lcd.print("Price: " + String(items[i].price) + " Rs");
      delay(3000);

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Total: " + String(total) + " Rs");

      itemFound = true;
      break;
    }
  }

  if (!itemFound) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Unknown Item");
    delay(2000);
  }
}

void deleteItem(String uid) {
  // Check if the UID matches any of the known items
  bool itemFound = false;
  for (int i = 0; i < sizeof(items) / sizeof(items[0]); i++) {
    if (uid.equalsIgnoreCase(items[i].uid)) {
      // Subtract the item price from the total
      total -= items[i].price;

      // Activate servo to indicate action
      activateServo();

      // Display item name and updated total on the LCD
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Deleted: " + items[i].name);
      lcd.setCursor(0, 1);
      lcd.print("Price: -" + String(items[i].price) + " Rs");
      delay(3000);

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Total: " + String(total) + " Rs");

      itemFound = true;
      break;
    }
  }

  if (!itemFound) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Unknown Item");
    delay(2000);
  }
}

void activateServo() {
  myServo.write(90); // Open the servo (90 degrees)
  delay(1000);       // Keep it open for a second
  myServo.write(0);  // Return to closed position
}

void finalizeBill() {
  // Display total and end the transaction
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Final Total:");
  lcd.setCursor(0, 1);
  lcd.print(String(total) + " Rs");
  delay(5000);

  // Send SMS with total amount
  sendSMS("+1234567890", "Your final bill is: " + String(total) + " Rs"); // Replace with the target phone number

  // Reset total for next transaction
  total = 0;
  deleteMode = false;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Total: 0 Rs");
}

void sendSMS(String phoneNumber, String message) {
  sendCommand("AT+CMGS=\"" + phoneNumber + "\"", 2000); // Start sending SMS
  sim808.print(message);
  sim808.write(26); // ASCII code for CTRL+Z to send the message
  delay(5000);
}

void sendCommand(String command, int timeout) {
  sim808.println(command);
  delay(timeout);
  while (sim808.available()) {
    Serial.write(sim808.read());
  }
}
