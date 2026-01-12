print("TRASH VISION ACTIVE: Raspberry Pi 5 Bridge Online")
import cv2
from roboflow import Roboflow
import threading
import time
import serial

API_KEY = "3955B9boHSHlorGo8uQT" 
WORKSPACE_ID = "sf-final"
PROJECT_ID = "waste-images-qb2ia"
VERSION_NUMBER = 3 

try:
    arduino = serial.Serial('/dev/ttyACM0', 9600, timeout=1)
    print("LOG: Arduino Bridge Connected via /dev/ttyACM0")
except:
    try:
        arduino = serial.Serial('/dev/ttyUSB0', 9600, timeout=1)
        print("LOG: Arduino Bridge Connected via /dev/ttyUSB0")
    except:
        arduino = None
        print("WARNING: No Arduino detected. System running in SIMULATION MODE.")

current_predictions = []
last_printed_category = None
running = True

rf = Roboflow(api_key=API_KEY)
project = rf.workspace(WORKSPACE_ID).project(PROJECT_ID)
model = project.version(VERSION_NUMBER).model

def fetch_predictions(cap):
    global current_predictions, running
    while running:
        ret, frame = cap.read()
        if ret:
            results = model.predict(frame, confidence=30).json()
            current_predictions = results.get('predictions', [])
        time.sleep(0.05)

# For Raspberry Pi, we remove CAP_DSHOW
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
cap.set(cv2.CAP_PROP_FPS, 30)

if not cap.isOpened():
    print("Error: Could not open webcam.")

thread = threading.Thread(target=fetch_predictions, args=(cap,), daemon=True)
thread.start()

while True:
    ret, frame = cap.read()
    if not ret: break

    found_trash_this_frame = None
    active_signal = '0'

    # Process predictions
    for prediction in current_predictions:
        label = prediction['class'].lower()
        conf = prediction['confidence']
        category = None 

        if any(word in label for word in ["orange", "banana", "peel", "food"]):
            category = "COMPOST"
            active_signal = 'C'
        elif any(word in label for word in ["paper", "cardboard", "box", "cup"]):
            category = "PAPER"
            active_signal = 'R'
        elif any(word in label for word in ["bottle", "plastic", "spoon"]):
            category = "PLASTIC"
            active_signal = 'P'
        elif any(word in label for word in ["can", "soda", "metal", "foil", "spoon"]):
            category = "METAL"
            active_signal = 'M'

        if category:
            found_trash_this_frame = category
            x, y = int(prediction['x']), int(prediction['y'])
            cv2.rectangle(frame, (x-60, y-60), (x+60, y+60), (0, 255, 0), 2)
            cv2.putText(frame, f"{category} ({conf*100:.0f}%)", (x-60, y-70), 
                        cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
            break # Priority: take the first valid detection

    # Send signal if a new category is detected
    if found_trash_this_frame and found_trash_this_frame != last_printed_category:
        print(f"ACTION: {found_trash_this_frame} detected. SENDING TO ARDUINO.")
        if arduino:
            arduino.write(active_signal.encode())
        last_printed_category = found_trash_this_frame
    elif not found_trash_this_frame:
        last_printed_category = None

    cv2.imshow("Trash Vision - Pi 5 Ready", frame)

    if cv2.waitKey(1) & 0xFF == ord('q'):
        running = False
        break

cap.release()
cv2.destroyAllWindows()
