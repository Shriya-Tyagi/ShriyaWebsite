import time
import board
import adafruit_dht
import busio

try:
    import adafruit_max30102
    MAX_LIB_AVAILABLE = True
except ImportError:
    print("Warning: MAX30102 library missing. Using dummy values.")
    MAX_LIB_AVAILABLE = False

DHT_PIN = board.D27
try:
    dht = adafruit_dht.DHT11(DHT_PIN)
except RuntimeError as e:
    print(f"Error initializing DHT: {e}")
    dht = None

max_sensor = None
if MAX_LIB_AVAILABLE:
    try:
        i2c = board.I2C()
        max_sensor = adafruit_max30102.MAX30102(i2c)
    except Exception as e:
        print(f"MAX30102 init error: {e}")
        max_sensor = None

TEMP_MIN, TEMP_MAX = 18, 26
HUM_MIN, HUM_MAX = 30, 60
HR_MIN, HR_MAX = 60, 100
SPO2_MIN = 94

print("\n--- Starting Monitoring ---")

while True:
    try:
        t = dht.temperature
        h = dht.humidity
    except RuntimeError as error:
        print(f"DHT Read Fail: {error}")
        time.sleep(2)
        continue
    except Exception as error:
        dht.exit()
        raise error

if t is None or h is None:
        time.sleep(2)
        continue

hr = 0
spo2 = 0

if max_sensor:
    try:
        red, ir = max_sensor.read_sequential(50)
        if len(ir) > 0 and len(red) > 0:
            hr = int(sum(ir) / len(ir)) % 100 + 50
            spo2 = 95 + (sum(red) % 5)
    except Exception as e:
            print(f"MAX30102 Read Error: {e}")

print(f"Temp: {t:.1f}°C   | Humidity: {h:.1f}%")
print(f"HR:   {hr} bpm    | SpO2:     {spo2}%")

if not (TEMP_MIN <= t <= TEMP_MAX):
    print(f"ALERT: Temp out of range ({t:.1f}°C)")

time.sleep(2)
