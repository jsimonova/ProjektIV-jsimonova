# Kontrola teploty, vlhkosti, stav okna /e-mailová informácia o stave okna
#include <DHT.h>  // kniznica potrebna pre DHT22 snimac
#include <DHT_U.h>
#include <WiFi.h> //zahrnie knižnicu na prácu s WiFi modulom
#include <ESP_Mail_Client.h> //zahrnie knižnicu na prácu s e-mailom
#include <Adafruit_Sensor.h>  // kniznica pre senzor 
#include <Wire.h>    //kniznica pre komunikaciu i2c komunikácia s displejom + DHT senzor
#include <Adafruit_GFX.h>  // kniznica pre displej 
#include <Adafruit_SSD1306.h>  // pozor kniznica pre MiniOled  Adafruit_SSD1306_Wemos_OLED nie klasicka Adafruit_SSD1306



//NASTAVENIE WIFI PRIPOJENIA
#define WIFI_NAZOV "xxxxxxxx" //definuje premennú WIFI_NAZOV 
#define WIFI_HESLO "xxxxxxxx" //definuje premennú WIFI_HESLO 

//NASTAVENIE SMTP PROTOKLU
//špecifikujeme Simple Mail Transfer Protocol
#define SMTP_HOST "mail.webhouse.sk" //definuje premennú SMTP_HOST (posielame z účtu Gmail)
#define SMTP_PORT 587 //definuje premennú SMTP_PORT (predvolený port serveru)

//špecifikujeme odosielateľa a príjemcu
#define ODOSIELATEL_EMAIL "xxxxxx@xxxxxx" //definuj e-mail odosielateľa
#define ODOSIELATEL_HESLO "xxxxxxxxx" //definuje heslo do e-mailu odosielateľa
#define PRIJEMCA_EMAIL "xxxxxx@xxxxxxx" //definuje e-mail príjemcu

SMTPSession smtp; //slúži na odosielanie e-mailov

//DEFIN reset displ none
#define OLED_RESET -1  // displej sa neda resetovať
Adafruit_SSD1306 display(OLED_RESET);
// uvodneho loga
#define NUMFLAKES 10
#define XPOS 0
#define YPOS 1
#define DELTAY 2
#define LOGO16_GLCD_HEIGHT 16
#define LOGO16_GLCD_WIDTH  16
static const unsigned char PROGMEM logo16_glcd_bmp[] =
{ B00000000, B11000000,
  B00000001, B11000000,
  B00000001, B11000000,
  B00000011, B11100000,
  B11110011, B11100000,
  B11111110, B11111000,
  B01111110, B11111111,
  B00110011, B10011111,
  B00011111, B11111100,
  B00001101, B01110000,
  B00011011, B10100000,
  B00111111, B11100000,
  B00111111, B11110000,
  B01111100, B11110000,
  B01110000, B01110000,
  B00000000, B00110000 };

#if (SSD1306_LCDHEIGHT != 48)
#error("Height incorrect, please fix Adafruit_SSD1306.h!");
#endif



// Uncomment the type of sensor in use:
//#define DHTTYPE    DHT11     // DHT 11
#define DHTTYPE    DHT22     // DHT 22 (AM2302)  - POUZITY SENZOR 
//#define DHTTYPE    DHT21     // DHT 21 (AM2301)
#define DHTPIN 4 // Digital pin connected to the DHT sensor  POUZITE GPIO 04 



DHT dht(DHTPIN, DHTTYPE);
//Def premenen - OKNO senzor otvorenia
int OKNO_PIN = 26;                          //OKNO kontakt  POUZITE GPIO 26  D26 
int OKNO_StavSucasny = LOW;               // premenna sucasny stav dverneho senzora
int OKNO_StavMinuly = LOW;                // premenna minuly stav dverneho senzora

void setup() {
 Serial.begin(9600);
 Serial.println();
 Serial.print("Pripájam k sieti...");
 WiFi.begin(WIFI_NAZOV, WIFI_HESLO); //spustí WiFi modul
 while (WiFi.status()!= WL_CONNECTED){ //pokiaľ nie je WiFi pripojená, vypisuje bodky, toto je podmienka aby to vedelo odslat email, ak nie je pripojen k wifi ani nezacne zobrazovat udaje
    Serial.print(".");
     delay(2000);

  }

  Serial.println("");
  Serial.print("Pripojené na WiFi sieť ");
  Serial.println(WIFI_NAZOV);
  Serial.print("IP adresa mikrokontroléra je: ");
  Serial.println(WiFi.localIP()); //vypíše IP adresu ESP32
  Serial.println();

ESP_Mail_Session session;

  session.server.host_name = SMTP_HOST ;
  session.server.port = SMTP_PORT;
  session.login.email = ODOSIELATEL_EMAIL;
  session.login.password = ODOSIELATEL_HESLO;
  session.login.user_domain = "";
  
 SMTP_Message message;

  message.sender.name = "ESP32";
  message.sender.email = ODOSIELATEL_EMAIL;
  message.subject = "ESP32 is UP";
  message.addRecipient("TESTER",PRIJEMCA_EMAIL);  

 //Send HTML message
  String htmlMsg = "<div style=\"color:#FF0000;\"><h1>ESP sa ZOBUDILO</h1><p>Sent by ESP</p></div>";
  message.html.content = htmlMsg.c_str();
  message.html.content = htmlMsg.c_str();
  message.text.charSet = "us-ascii";
  message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit; 

 if (!smtp.connect(&session)) //ak sa neporadí spojiť so serverom, 
    return; //vráti sa späť na začiatok

 else
 Serial.println("SMTP-pripojene"); 

if (!MailClient.sendMail(&smtp, &message))
  Serial.println("Error sending Email, " + smtp.errorReason());
else
 Serial.println("Odoslane info o pripojeni"); 
 



  pinMode(OKNO_PIN, INPUT_PULLUP);  //nastavenie rezimu INPUT pre OKNO_PIN - treba nsatavit rezim PULLUP lebo ma to nejaky odpor aj ked je zopnute

  dht.begin();
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);  // initialize with the I2C addr 0x3C (for the 64x48)
  // Show image buffer on the display hardware.
  // Since the buffer is intialized with an Adafruit splashscreen
  // internally, this will display the splashscreen.
  //zobrazenie uvodneho  logo
  display.display();
  delay(2000);

  // Clear the buffer.
  display.clearDisplay();
  }









void zobraz_na_disp_stav_okna() {
   // display OKNO_STAV
  display.setTextSize(1); 
  display.setCursor(0, 25);
  display.print("OKNO: ");
  display.setTextSize(1,2); 
  display.setCursor(10,35);
  
  if (OKNO_StavSucasny == LOW)
     {display.print("ZATVORENE");
     }
   else
    {display.print("OTVORENE");
    }
  display.display(); //zobrazim 
  delay(2000); //pockam
  display.clearDisplay(); // vymazem
}



void loop() {
  delay(1000); 

  OKNO_StavMinuly = OKNO_StavSucasny;                   // ulozime stary stav dverneho senzora
  OKNO_StavSucasny = digitalRead(OKNO_PIN);            // nacitame novy stav

 
 
 if (OKNO_StavMinuly == LOW && OKNO_StavSucasny == HIGH)            // zmena: LOW -> HIGH
  {  Serial.println("OKNO OTVORENE!");
     zobraz_na_disp_stav_okna();
          // do tejto časti je možné pridať alarm, notifikaciu... 
ESP_Mail_Session session;

  session.server.host_name = SMTP_HOST ;
  session.server.port = SMTP_PORT;
  session.login.email = ODOSIELATEL_EMAIL;
  session.login.password = ODOSIELATEL_HESLO;
  session.login.user_domain = "";
  
 SMTP_Message message;

  message.sender.name = "ESP32";
  message.sender.email = ODOSIELATEL_EMAIL;
  message.subject = "ESP32 is UP";  
  message.addRecipient("TESTER",PRIJEMCA_EMAIL);  


           String htmlMsg = "<div style=\"color:#FF0000;\"><h1>OKNO OTVORENE</p></div>";
      message.html.content = htmlMsg.c_str();
      message.html.content = htmlMsg.c_str();
      message.text.charSet = "us-ascii";
      message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit; 
     
 if (!smtp.connect(&session)) //ak sa neporadí spojiť so serverom, 
    return; //vráti sa späť na začiatok

 else
 Serial.println("SMTP-pripojene"); 

if (!MailClient.sendMail(&smtp, &message))
  Serial.println("Error sending Email, " + smtp.errorReason());
else
 Serial.println("Odoslane info o pripojeni"); 

  } 
 else 
   if (OKNO_StavMinuly == HIGH && OKNO_StavSucasny == LOW)          // zmena: HIGH -> LOW
     {
        Serial.println("OKNO ZATVORENE!");
        zobraz_na_disp_stav_okna();

ESP_Mail_Session session;

  session.server.host_name = SMTP_HOST ;
  session.server.port = SMTP_PORT;
  session.login.email = ODOSIELATEL_EMAIL;
  session.login.password = ODOSIELATEL_HESLO;
  session.login.user_domain = "";
  
 SMTP_Message message;

  message.sender.name = "ESP32";
  message.sender.email = ODOSIELATEL_EMAIL;
  message.subject = "ESP32 is UP";
  message.addRecipient("TESTER",PRIJEMCA_EMAIL);  


         String htmlMsg = "<div style=\"color:#FF0000;\"><h1>OKNO ZATVORENE</p></div>";
         message.html.content = htmlMsg.c_str();
        message.html.content = htmlMsg.c_str();
        message.text.charSet = "us-ascii";
       message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit; 
       
 if (!smtp.connect(&session)) //ak sa neporadí spojiť so serverom, 
    return; //vráti sa späť na začiatok

 else
 Serial.println("SMTP-pripojene"); 

if (!MailClient.sendMail(&smtp, &message))
  Serial.println("Error sending Email, " + smtp.errorReason());
else
 Serial.println("Odoslane info o pripojeni"); 
        
        }



  //Teraz citame teplotu a vlhkost 
  float t = dht.readTemperature();   // do float premenej t - nacitavam teplotu
  float h = dht.readHumidity();  // h - vlhkost
  // kontrola ci vieme citat data z snimaca DHT22 ak nie vypise hlasku
  if (isnan(h) || isnan(t)) {
    Serial.println("Chyba citania z teplotneho senzora!"); 
  }
  // clear display
  display.clearDisplay();
  // Print to serial monitor  -  pre testovanie DHT22 vypis na ser.konzolu  
 
  Serial.print(F("Vlhkost: ")); // vypisuje na konzolu
  Serial.print(h);
  Serial.print(F(" %"));
  Serial.println(F(" "));
  Serial.print(F("Teplota: "));
  Serial.print(t);
  Serial.print(F(" °C"));
  Serial.println(F(" "));
  Serial.print(F("STAV OKNA: "));
  Serial.println(OKNO_StavSucasny);
  //Serial.println();
  //Serial.println(F("Nacitavam ..... "));

  // display temperature  na dsiplej
  display.setTextSize(1);  // velkost text
  display.setTextColor(WHITE);
  display.setCursor(0,0); // pozicia 
  display.print("Teplota: ");
  display.setTextSize(1,2); 
  display.setCursor(10,10);
  display.print(t);
   display.print(" C");
  // display humidity
  display.setTextSize(1); 
  display.setCursor(0, 25);
  display.print("Vlhkost: ");
  display.setTextSize(1,2); 
  display.setCursor(10,35);
  display.print(h);
  display.print(" %");
  display.display(); //zobrazim 
  delay(2000); //pockam
  display.clearDisplay(); // vymazem
  
 
   //
   display.setTextSize(1);
   display.setCursor(0, 0);
  display.print("Nacitavam...........");
  display.display(); //zobrazim

  display.display(); 
}

