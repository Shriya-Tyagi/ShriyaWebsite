import time
import board
import adafruit_dht
import adafruit_max30102
import digitalio


# -----------------------------
# SENSOR INITIALIZATION
# -----------------------------

# DHT sensor (change DHT22 → DHT11 if needed)
dht = adafruit_dht.DHT22(board.D4)

# Heart rate + SpO2 sensor (MAX30102 over I2C)
max_sensor = adafruit_max30102.MAX30102(board.I2C())

# Digital sound sensor (HIGH = quiet, LOW = noise detected)
sound = digitalio.DigitalInOut(board.D17)
sound.direction = digitalio.Direction.INPUT


# -----------------------------
# SAFE OPERATING RANGES
# -----------------------------
TEMP_MIN, TEMP_MAX = 18, 26     # °C
HUM_MIN, HUM_MAX = 30, 60       # %
HR_MIN, HR_MAX = 60, 100        # bpm
SPO2_MIN = 94                   # % oxygen


print("Starting health and environment monitoring...")


# -----------------------------
# MAIN LOOP
# -----------------------------
while True:
    try:
        # -----------------------------
        # READ EACH SENSOR
        # -----------------------------

        t = dht.temperature      # may return None if sensor is unstable
        h = dht.humidity

        # Handle None values early
        if t is None or h is None:
            print("Warning: DHT sensor returned no data. Retrying...")
            time.sleep(2)
            continue

        # Read 50 samples from MAX30102
        red, ir = max_sensor.read_sequential(50)

        # Sanity check for empty readings (rare)
        if len(ir) == 0 or len(red) == 0:
            print("Warning: Heart sensor (MAX30102) returned no data.")
            time.sleep(2)
            continue

        # Simple heart rate approximation (not medically accurate)
        hr = int(sum(ir) / len(ir)) % 100 + 50

        # Simulated SpO2 approximation
        spo2 = 95 + (sum(red) % 5)

        # Noise sensor → LOW = noise detected
        noise_detected = not sound.value

        # -----------------------------
        # PRINT LIVE DATA
        # -----------------------------
        print(f"Temperature: {t:.1f}°C | Humidity: {h:.1f}% | Heart Rate: {hr} bpm | "
              f"SpO2: {spo2}% | Noise Detected: {noise_detected}")

        # -----------------------------
        # ALERT CHECKS
        # -----------------------------
        if not (TEMP_MIN <= t <= TEMP_MAX):
            print(f"Alert: Temperature out of range ({t:.1f}°C)")

        if not (HUM_MIN <= h <= HUM_MAX):
            print(f"Alert: Humidity out of range ({h:.1f}%)")

        if not (HR_MIN <= hr <= HR_MAX):
            print(f"Alert: Abnormal heart rate ({hr} bpm)")

        if spo2 < SPO2_MIN:
            print(f"Alert: Low oxygen level ({spo2}% SpO2)")

        if noise_detected:
            print("Alert: High noise detected")

    # -----------------------------
    # ERROR HANDLING
    # -----------------------------
    except RuntimeError:
        # Common DHT errors: checksum error / bad temp read
        print("Sensor read error, retrying...")

    except Exception as e:
        # Any other sensor or system error
        print("Critical Error:", e)
        dht.exit()
        break

    # Wait before next reading
    time.sleep(10)
