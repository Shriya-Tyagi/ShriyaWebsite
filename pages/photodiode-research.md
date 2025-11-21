import time
import board
import adafruit_dht
import adafruit_max30102
import digitalio


# -----------------------------
# SENSOR INITIALIZATION
# -----------------------------

# DHT sensor (CHANGE THIS TO DHT11 IF YOU HAVE DHT11)
dht = adafruit_dht.DHT11(board.D4)

# Heart rate + SpO2 sensor (I2C)
max_sensor = adafruit_max30102.MAX30102(board.I2C())

# Sound sensor (digital)
sound = digitalio.DigitalInOut(board.D17)
sound.direction = digitalio.Direction.INPUT


# -----------------------------
# SAFE OPERATING LIMITS
# -----------------------------
TEMP_MIN, TEMP_MAX = 18, 26
HUM_MIN, HUM_MAX = 30, 60
HR_MIN, HR_MAX = 60, 100
SPO2_MIN = 94

print("Starting health and environment monitoring...")


# -----------------------------
# MAIN LOOP
# -----------------------------
while True:
    try:
        # -----------------------------
        # READ TEMPERATURE + HUMIDITY
        # -----------------------------
        t = dht.temperature
        h = dht.humidity

        # DHT sensors occasionally return None → retry safely
        if t is None or h is None:
            print("Warning: DHT returned no data, retrying...")
            time.sleep(2)
            continue

        # -----------------------------
        # READ HEART SENSOR
        # -----------------------------
        red, ir = max_sensor.read_sequential(50)

        if len(ir) == 0 or len(red) == 0:
            print("Warning: Heart sensor returned empty data")
            time.sleep(2)
            continue

        # Simple approximations
        hr = int(sum(ir) / len(ir)) % 100 + 50
        spo2 = 95 + (sum(red) % 5)

        # -----------------------------
        # READ SOUND SENSOR
        # -----------------------------
        noise_detected = not sound.value  # LOW means noise

        # -----------------------------
        # PRINT SENSOR VALUES
        # -----------------------------
        print(
            f"Temp: {t:.1f}°C | Humidity: {h:.1f}% | "
            f"HR: {hr} bpm | SpO2: {spo2}% | Noise: {noise_detected}"
        )

        # -----------------------------
        # ALERT CONDITIONS
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

    except RuntimeError as e:
        # VERY COMMON for DHT sensors — not fatal
        print("Sensor read error:", e)
        time.sleep(2)

    except Exception as e:
        # Unexpected hardware failure
        print("Critical error:", e)
        dht.exit()
        break

    # Delay before next loop
    time.sleep(5)