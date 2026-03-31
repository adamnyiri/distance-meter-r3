# Távolságmérő
A projekt alapját a HC-SR04 típusú ultrahangos szenzor adja. Ez az eszköz a denevérek és delfinek által is használt echolokáció elvén működik.
<img width="963" height="629" alt="mentes" src="https://github.com/user-attachments/assets/81d0d467-9933-422f-a831-5ac5a60f28a3" />
## A mérés menete
A Szenzor két fő részből áll: egy adóból és egy vevőből. A mikrokontroller egy rövid impulzust küld az adó lábra, ekkor a szenzor egy 40 kHz-s ultrahangcsomagot bocsát ki. A hangullám elindul, visszaverődik a tárgyről, majda vevőegység érzékeli azt, ekkor a vevő lábon megjelnik egy impulzus, melynek hossza ponstosan megegyezik a hang oda-vissza útjának idejével.
## Hardveres felépítés és alkatrészlista
A projektnek egy Arduino Uno R3 mikrokonmtroller az alapja. Ez felel a szenzor vezérléséért, az adatok feldolzásáért és a perifériák működtetéséért. 
## Felhasznált komponensek

| Név |	Mennyiség |	Összetevő |
| :--- | :---: | ---: |
| U1 |	1 |	Arduino Uno R3 |
| Ulcd |	1 |	PCF8574-alapú, 32 (0x20) LCD 16 x 2 (I2C) |
| D1 | 1 | Zöld LED |
| D2 | 1 | Sárga LED |
| D3 | 1 |	Piros LED |
| PIEZO1 |	1 |	Piezo |
| R1, R2, R3 |	3 |	220 Ω Ellenállás |
|DISTultrasonic |	1 |	Ultrahangos távolságérzékelő (4 tűs) |

## Kapcsolási rajz
<img width="1084" height="828" alt="kapcsolasi" src="https://github.com/user-attachments/assets/c7fbdff0-7437-4f77-a5a9-e13a97e4dc5d" />

# Szoftveres architektúra és algoritmus-logika
A projekt szoftveres megvalósítása az Arduino keretrendszer két fő egységére, a setup() és a loop() függvényekre épül. A program alapvetően egy eseményvezérelt ciklus, amely folyamatosan mintavételezi a környezetet, majd az adatok alapján döntéseket hoz.
## Inicializálás
**I/O portok:** Meghatározzuk a bemeneti (Echo) és kimeneti (Trig, LED-ek, Buzzer) lábakat.  
**I2C kommunikáció:** Inicializáljuk az LCD kijelzőt a 32-es címen.  
**Egyedi karakterek:** A lcd.createChar(0, teliTegla) parancs segítségével feltöltjük a mikrokontroller memóriájába a vizuális sávhoz szükséges egyedi bitképet.  
## Mérési ciklus (loop())
**Triggerelés:** A trigPin rövid idejű magasba húzásával utasítjuk a szenzort a mérésre.  
**Időmérés:** A pulseIn() függvény segítségével nanoszekundumos pontossággal mérjük meg, mennyi ideig volt magas az echoPin  
**Konverzió:** A mért időt elosztjuk kettővel (mivel a hang oda-vissza utat tett meg), majd megszorozzuk a hangsebességgel  

## Döntési logika
| Zóna |	Feltétel |	LED | Hangjelzés |
| :--- | :---: | :---: | ---: |
| Biztonságos | d > 50 cm | Zöld LED | Nincs |
| Figyelmeztetés | 20 cm < d <= 50 cm | Sárga LED | Szaggatott (1000Hz) |
| Veszély | d <= 20 cm | Piros LED | Folyamatos (2000Hz) |

## Vizuláis megjelenítés
A rendszer egyik egyedi funkciója a kijelző alsó sorában látható grafikus skála. Az algoritmus a távolságot leképezi az LCD $16$ karakteres szélességére:  
A kockakSzama = 16 - (tav / 5) képlet biztosítja, hogy minél közelebb van a tárgy, annál több "teli tégla" jelenjen meg.  
Egy for ciklus segítségével a program végigszalad a 16 pozíción, és eldönti, hogy az adott helyre a saját készítésű karaktert vagy üres helyközt (szóközt) kell-e írni.

## Teljes forráskód:
```cpp
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(32, 16, 2); 

const int trigPin = 9;
const int echoPin = 10;
const int ledZold = 2, ledSarga = 3, ledPiros = 4, buzzer = 5;

byte teliTegla[8] = {
  B11111, B11111, B11111, B11111, B11111, B11111, B11111, B11111
};

void setup() {
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(ledZold, OUTPUT);
  pinMode(ledSarga, OUTPUT);
  pinMode(ledPiros, OUTPUT);
  pinMode(buzzer, OUTPUT);

  lcd.init();
  lcd.backlight();
  
  lcd.createChar(0, teliTegla); 
  
  lcd.print("Radar indul...");
  delay(1000);
  lcd.clear();
}

void loop() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(2);
  digitalWrite(trigPin, LOW);

  long ido = pulseIn(echoPin, HIGH);
  float tav = ido * 0.03436 / 2.0;

  lcd.setCursor(0, 0);
  lcd.print("Tav: ");
  lcd.print(tav, 1); 
  lcd.print("cm   ");

  lcd.setCursor(0, 1);
  
 
  int kockakSzama = 16 - (tav / 5); 
  if (kockakSzama < 0) kockakSzama = 0;
  if (kockakSzama > 14) kockakSzama = 16;

  for (int i = 0; i < 16; i++) {
    if (i < kockakSzama) {
      lcd.write(0); 
    } else {
      lcd.print(" ");
    }
  }

  if (tav > 50) {
    digitalWrite(ledZold, HIGH);
    digitalWrite(ledSarga, LOW);
    digitalWrite(ledPiros, LOW);
    noTone(buzzer);
  } 
  else if (tav <= 50 && tav > 20) {
    digitalWrite(ledZold, LOW);
    digitalWrite(ledSarga, HIGH);
    digitalWrite(ledPiros, LOW);
    tone(buzzer, 1000, 100);
  } 
  else {
    digitalWrite(ledZold, LOW);
    digitalWrite(ledSarga, LOW);
    digitalWrite(ledPiros, HIGH);
    tone(buzzer, 2000); 
  }

  delay(10);
}
