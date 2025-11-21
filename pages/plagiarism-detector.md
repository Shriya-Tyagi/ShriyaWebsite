# 🔍 Plagiarism Detector

![plagiarism detector showing results for ketchup](/media/plag-ketchup.jpg)

**Technologies:** Python, Scikit-learn, NLTK, Requests, BeautifulSoup  

**Date:** Aug–Sept 2025  

Developed a TF-IDF + cosine similarity text similarity system. Used NLTK pipelines for tokenization, stopword removal, and keyword extraction. Built for scalable detection of academic and technical plagiarism.

![plagiarism detector analysing tomato sauce paragraph](/media/plag-tomato.jpg)

_Watch it correctly identify the source of a tomato sauce article!_
import time
import board
import adafruit_dht
import digitalio
import busio

# -----------------------------
# LIBRARY CHECK
# -----------------------------
# There is no standard "adafruit_max30102" library. 
# We wrap this import to prevent immediate crashing if you don't have it.
try:
    import adafruit_max30102
    MAX_LIB_AVAILABLE = True
except ImportError:
    print("⚠️ Warning: 'adafruit_max30102' library not found.")
    print("   Using Dummy Data for Heart Rate. Install the library to fix.")
    MAX_LIB_AVAILABLE = False

# -----------------------------
# SENSOR INITIALIZATION
# -----------------------------

# 1. DHT Sensor
# WE USE GPIO 27 (Pin 13) because GPIO 4 is used for DS18B20
DHT_PIN = board.D27 
try:
    dht = adafruit_dht.DHT11(DHT_PIN)
except RuntimeError as e:
    print(f"Error initializing DHT: {e}")

# 2. Sound Sensor (Digital)
sound = digitalio.DigitalInOut(board.D17)
sound.direction = digitalio.Direction.INPUT

# 3. Heart Rate (MAX30102)
max_sensor = None
if MAX_LIB_AVAILABLE:
    try:
        i2c = board.I2C()  # uses board.SCL and board.SDA
        max_sensor = adafruit_max30102.MAX30102(i2c)
    except ValueError:
        print("⚠️ Error: MAX30102 not found on I2C. Check wiring (SDA/SCL).")

# -----------------------------
# SAFE OPERATING LIMITS
# -----------------------------
TEMP_MIN, TEMP_MAX = 18, 26
HUM_MIN, HUM_MAX = 30, 60
HR_MIN, HR_MAX = 60, 100
SPO2_MIN = 94

print("\n--- Starting Monitoring ---")
print("Press CTRL+C to stop")

# -----------------------------
# MAIN LOOP
# -----------------------------
while True:
    try:
        # --- READ DHT (TEMP/HUMIDITY) ---
        try:
            t = dht.temperature
            h = dht.humidity
        except RuntimeError as error:
            # DHT11 is frequent to fail. We catch it, print, and move on.
            print(f"DHT Read Fail (Retrying): {error.args[0]}")
            time.sleep(2.0)
            continue
        except Exception as error:
            dht.exit()
            raise error

        # If DHT returns None (happens occasionally)
        if t is None or h is None:
            time.sleep(2.0)
            continue

        # --- READ HEART SENSOR ---
        hr = 0
        spo2 = 0
        
        if max_sensor:
            # This relies on your specific library supporting 'read_sequential'
            try:
                red, ir = max_sensor.read_sequential(50)
                if len(ir) > 0 and len(red) > 0:
                    # Your approximation logic
                    hr = int(sum(ir) / len(ir)) % 100 + 50
                    spo2 = 95 + (sum(red) % 5)
            except Exception as e:
                print(f"MAX30102 Read Error: {e}")

        # --- READ SOUND SENSOR ---
        # Note: On some sound modules, LOW is noise, on others HIGH is noise.
        # Adjust 'not sound.value' if it's detecting silence as noise.
        noise_detected = not sound.value 

        # --- DISPLAY ---
        print("-" * 40)
        print(f"Temp: {t:.1f}°C   | Humidity: {h:.1f}%")
        print(f"HR:   {hr} bpm    | SpO2:     {spo2}%")
        print(f"Noise Detected: {noise_detected}")

        # --- ALERTS ---
        if not (TEMP_MIN <= t <= TEMP_MAX):
            print(f"🚨 ALERT: Temp out of range! ({t:.1f}°C)")
        
        if noise_detected:
            print("🚨 ALERT: Loud noise detected!")

    except KeyboardInterrupt:
        print("\nExiting...")
        dht.exit()
        break
    
    except Exception as e:
        print(f"Critical Main Loop Error: {e}")
        dht.exit()
        time.sleep(2)

    # DHT11 needs at least 2 seconds between reads to recharge
    time.sleep(2)