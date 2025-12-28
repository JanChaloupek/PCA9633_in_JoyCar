# JoyCar – ovládání motorů

Pro ovládání motorů je na JoyCaru použitý obvod **PCA9633**. Je to LED driver pro 4 (skupiny) LED, ale na JoyCaru jsou na jeho výstupy připojené motory. Obvod tedy nedává „PWM pro LED“, ale **napětí pro jednotlivé směry motorů**.  
**Datasheet je k dispozici [zde](datasheet/PCA9633.pdf).**

---

## Adresování PCA9633

Tento budič může na sběrnici I²C vystupovat až na pěti různých adresách.

### HW a SW adresy

- **HW adresa:** 0x62 (na starší verzi JoyCaru: 0x60).  
  8pinová verze PCA9633 má pevnou adresu 0x62.  
  10pinová verze může mít adresu 0x60 až 0x63 podle toho, jak jsou připojené piny A0 a A1.  
  Existuje i 16pinová varianta, která má vyvedeny všechny adresové bity (A0–A6), takže by bylo možné nastavit libovolnou I²C adresu.

- **SW adresy:**  
  Až čtyři softwarové adresy nastavitelné pomocí registrů **SUBADR1**, **SUBADR2**, **SUBADR3** a **ALLCALLADR**.  
  Všechny čtyři lze vypnout — včetně adresy **0x70** (viz registr **MODE1**).

---

## Struktura registrů

### Control registr

První byte, který zapíšeme na I²C adresu budiče, se uloží do **control registru** (strana 10 datasheetu).  
Spodní nibble určuje, **který registr chceme číst nebo zapisovat** (seznam je na straně 11).  
Horní nibble nastavuje automatické inkrementování adresy registrů — tuto funkci ale nevyužíváme.  
V praxi tedy vždy zapisujeme jen **dva byty**:  
1. control registr  
2. hodnota cílového registru

![ControlRegistr](img/PCA9633%20-%201%20Control%20register.png)  
![RegisterList](img/PCA9633%20-%202%20Register%20definition.png)

---

### Registr MODE1

Při inicializaci motorů se v návodech JoyCaru používají dvojice bytů:

- `[0x00, 0x01]` – zápis do registru 0 (**MODE1**)  
  - vypne sleep  
  - vypne subadresy **SUBADR1**, **SUBADR2**, **SUBADR3**  
  - **ALLCALLADR** zůstává aktivní  
  - probuzení ze sleepu trvá cca 500 µs (datasheet str. 12), takže je vhodné po inicializaci udělat krátké `sleep(2)`

![Register MODE1](img/PCA9633%20-%204%20Register%201%20-%20MODE1.png)

---

### Registr LEDOUT

Druhá dvojice bytů `[0xE8, 0xAA]` zapisuje do registru 8 (**LEDOUT**).  
Hodnota `0xAA` říká, že **všechny čtyři výstupy** budou řízeny přes registry **PWMx** (datasheet str. 14).

![Register LEDOUT](img/PCA9633%20-%203%20Register%208%20-%20LEDOUT.png)

---

### Registry PWMx

Adresy 2–5 odpovídají registrům **PWM0** až **PWM3** (datasheet str. 13).  
Hodnoty lze nejen zapisovat, ale i číst.  
V praxi je ale jednodušší si hodnoty PWM uchovávat v instanci motoru, než je pokaždé vyčítat přes I²C.

![Register PWMx](img/PCA9633%20-%205%20Register%202-5%20-%20PWMx.png)

---

## Inicializace při použití adresy 0x62

```python
i2c.write(0x62, bytes([0, 0x00]))  # MODE1
i2c.write(0x62, bytes([8, 0xAA]))  # LEDOUT
```

---

# Objektová verze ovládání motorů

Níže je jednoduchá třída `MotorDriver`, která obsluhuje PCA9633 na adrese **0x62**.  
Každý motor má dva PWM kanály – jeden pro směr dopředu, druhý pro směr dozadu.

Kód není psaný pro konkrétní implementaci MicroPythonu. Není tedy přímo použitelný ani na Micro:bitu, ani na Pico:edu. Slouží pouze jako ukázka principu objektové implementace.

### Mapa kanálů
- Motor A: PWM0 (dopředu), PWM1 (dozadu)  
- Motor B: PWM2 (dopředu), PWM3 (dozadu)

---

## Třída MotorDriver

```python
from machine import I2C, Pin
from time import sleep

class MotorDriver:
    def __init__(self, i2c, address=0x62):
        self.i2c = i2c
        self.address = address

        # Inicializace PCA9633
        self.i2c.write(self.address, bytes([0, 0x00]))  # MODE1 – probuzení
        sleep(0.002)
        self.i2c.write(self.address, bytes([8, 0xAA]))  # LEDOUT – PWM režim

        # Uložené hodnoty PWM (není nutné, ale praktické)
        self.pwm = [0, 0, 0, 0]

    def set_pwm(self, channel, value):
        """Nastaví PWM kanál (0–3) na hodnotu 0–255."""
        value = max(0, min(255, value))
        self.pwm[channel] = value
        self.i2c.write(self.address, bytes([2 + channel, value]))

    def motorA(self, speed):
        """Řízení motoru A: speed -255 až +255."""
        if speed > 0:
            self.set_pwm(0, speed)
            self.set_pwm(1, 0)
        elif speed < 0:
            self.set_pwm(0, 0)
            self.set_pwm(1, -speed)
        else:
            self.set_pwm(0, 0)
            self.set_pwm(1, 0)

    def motorB(self, speed):
        """Řízení motoru B: speed -255 až +255."""
        if speed > 0:
            self.set_pwm(2, speed)
            self.set_pwm(3, 0)
        elif speed < 0:
            self.set_pwm(2, 0)
            self.set_pwm(3, -speed)
        else:
            self.set_pwm(2, 0)
            self.set_pwm(3, 0)

    def stop(self):
        """Zastaví oba motory."""
        self.motorA(0)
        self.motorB(0)
```

---

## Ukázka použití

```python
from machine import Pin, I2C
from time import sleep
from motor import MotorDriver

# Inicializace I2C
i2c = I2C(0, scl=Pin(17), sda=Pin(16))

# Vytvoření instance driveru
motors = MotorDriver(i2c)

# Jízda dopředu
motors.motorA(150)
motors.motorB(150)
sleep(2)

# Zatáčka doprava
motors.motorA(50)
motors.motorB(200)
sleep(1)

# Zpátečka
motors.motorA(-180)
motors.motorB(-180)
sleep(1)

# Stop
motors.stop()
```
