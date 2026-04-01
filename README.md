# RFID Attendance System

## 📌 Overview
This project is a simple RFID-based attendance system that records user data using an RFID card. When a card is scanned, the system logs the attendance with date and time.

## ⚙️ Features
- RFID card scanning
- Automatic attendance marking
- Data stored in Google Sheets
- Real-time logging

## 🛠️ Technologies Used
- Arduino
- RFID Module (RC522)
- Google Apps Script
- WiFi Module (ESP8266 / ESP32)

## 📂 Project Structure
rfid-attendance-system/
│
├── rfid_attendance.ino        # Arduino Code
├── rfid_script.js             # Google Apps Script
├── README.md                  # Project Documentation

## 🚀 How It Works
1. RFID card is scanned using the reader
2. UID is captured by Arduino/ESP module
3. Data is sent to Google Apps Script
4. Attendance is stored in Google Sheets

## 📸 Output
Attendance is logged in a Google Sheet with:
- Name
- UID
- Date
- Time

## 💡 Future Improvements
- Add web dashboard
- Add face recognition
- Mobile app integration

## 👨‍💻 Author
sahil
