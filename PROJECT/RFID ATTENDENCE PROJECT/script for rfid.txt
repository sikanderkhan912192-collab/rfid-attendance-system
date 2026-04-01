script for rfid 



// Google Apps Script for Simple Scan Logger
// Extensions -> Apps Script -> paste this -> Deploy as Web App (Anyone)

function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();

  var uid   = e.parameter.uid || "";
  var name  = e.parameter.name || "";
  var date  = e.parameter.date || "";
  var time  = e.parameter.intime || ""; // ESP हर बार "intime" में time भेजेगा

  // हर स्कैन पर नया row: Name | UID | Date | Time
  sheet.appendRow([name, uid, date, time]);

  return ContentService.createTextOutput("OK");
}
