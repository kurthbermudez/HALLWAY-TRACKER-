<!DOCTYPE html>
<html>
<head>
    <title>Hallway Tracker System</title>

    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <style>

        body {
            font-family: Arial;
            background-color: lavender;
            margin: 20px;
            text-align: center;
        }

        h1 {
            color: violet;
        }

        .container {
            background: white;
            padding: 20px;
            border-radius: 10px;
            max-width: 750px;
            margin: auto;
        }

        input {
            padding: 10px;
            margin: 5px;
            width: 85%;
            box-sizing: border-box;
        }

        button {
            padding: 10px 15px;
            margin: 5px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        .in {
            background: green;
            color: white;
        }

        .out {
            background: orange;
            color: white;
        }

        .scan {
            background: #286090;
            color: white;
        }

        .clear {
            background: red;
            color: white;
        }

        video {
            width: 280px;
            max-width: 100%;
            border-radius: 10px;
            background: black;
        }

        table {
            width: 100%;
            margin-top: 20px;
            border-collapse: collapse;
            font-size: 13px;
        }

        th, td {
            border: 1px solid gray;
            padding: 8px;
        }

        th {
            background: lavender;
            color: black;
        }

        #status {
            color: violet;
            font-size: 14px;
        }

    </style>
</head>

<body>

<div class="container">

    <h1>HALLWAY TRACKER</h1>

    <p>Student Time In and Time Out Monitoring</p>

    <hr>

    <h3>Barcode Scanner</h3>

    <video id="camera" autoplay playsinline></video>

    <br>

    <button class="scan" onclick="startCamera()">
        START CAMERA
    </button>

    <button onclick="stopCamera()">
        STOP CAMERA
    </button>

    <p id="status">Camera is not active.</p>

    <hr>

    <h3>Student Information</h3>

    <input
        type="text"
        id="studentLRN"
        placeholder="Enter LRN"
    >

    <br>

    <input
        type="text"
        id="studentName"
        placeholder="Enter Student Name"
    >

    <br><br>

    <button class="in" onclick="timeIn()">
        TIME IN
    </button>

    <button class="out" onclick="timeOut()">
        TIME OUT
    </button>

    <hr>

    <h3>Attendance Records</h3>

    <table>

        <thead>
            <tr>
                <th>Student LRN</th>
                <th>Name</th>
                <th>Time In</th>
                <th>Time Out</th>
            </tr>
        </thead>

        <tbody id="records"></tbody>

    </table>

    <br>

    <button class="clear" onclick="clearRecords()">
        CLEAR RECORDS
    </button>

</div>


<script>

// LOAD SAVED RECORDS

let attendance = JSON.parse(
    localStorage.getItem("attendance")
) || [];

let stream = null;
let detector = null;
let scanning = false;
let lastScanned = "";


// DISPLAY RECORDS

function displayRecords() {

    let table = document.getElementById("records");

    table.innerHTML = "";

    attendance.forEach(function(student) {

        let row = table.insertRow();

        row.insertCell(0).innerText = student.LRN;
        row.insertCell(1).innerText = student.name;
        row.insertCell(2).innerText = student.timeIn;
        row.insertCell(3).innerText =
            student.timeOut || "Not yet";

    });

}


// SAVE RECORDS

function saveRecords() {

    localStorage.setItem(
        "attendance",
        JSON.stringify(attendance)
    );

    displayRecords();

}


// TIME IN

function timeIn() {

    let LRN = document.getElementById("studentLRN").value.trim();

    let name = document.getElementById("studentName").value.trim();

    if (LRN === "") {

        alert("Please enter or scan Student LRN.");

        return;
    }

    let existing = attendance.find(function(student) {

        return student.LRN === LRN && student.timeOut === "";

    });

    if (existing) {

        alert("Student is already timed in.");

        return;
    }

    let now = new Date().toLocaleString();

    attendance.push({

        LRN: LRN,
        name: name || LRN,
        timeIn: now,
        timeOut: ""

    });

    saveRecords();

    alert("Time In Recorded!");

    document.getElementById("studentLRN").value = "";
    document.getElementById("studentName").value = "";

}


// TIME OUT

function timeOut() {

    let LRN = document.getElementById("studentLRN").value.trim();

    if (LRN === "") {

        alert("Please enter Student LRN.");

        return;
    }

    let student = [...attendance].reverse().find(function(item) {

        return item.LRN === LRN && item.timeOut === "";

    });

    if (!student) {

        alert("No active Time In record found.");

        return;
    }

    student.timeOut = new Date().toLocaleString();

    saveRecords();

    alert("Time Out Recorded!");

    document.getElementById("studentLRN").value = "";
    document.getElementById("studentName").value = "";

}


// START CAMERA

async function startCamera() {

    let status = document.getElementById("status");

    let video = document.getElementById("camera");

    if (!navigator.mediaDevices ||
        !navigator.mediaDevices.getUserMedia) {

        status.innerText =
            "Camera requires HTTPS or a supported browser.";

        return;
    }

    try {

        if (!("BarcodeDetector" in window)) {

            status.innerText =
                "Barcode scanning is not supported by this browser.";

            return;
        }

        detector = new BarcodeDetector({

            formats: [
                "qr_code",
                "code_128",
                "ean_13"
            ]

        });

        stream = await navigator.mediaDevices.getUserMedia({

            video: {
                facingMode: "environment"
            },

            audio: false

        });

        video.srcObject = stream;

        await video.play();

        scanning = true;

        lastScanned = "";

        status.innerText =
            "Camera is working! Scanning barcode...";

        scanBarcode();

    }

    catch(error) {

        status.innerText =
            "Camera Error: " + error.name;

        alert(error.name + ": " + error.message);

    }

}


// SCAN BARCODE

async function scanBarcode() {

    if (!scanning) return;

    try {

        let video = document.getElementById("camera");

        if (video.readyState >= 2) {

            let codes = await detector.detect(video);

            if (codes.length > 0) {

                let code = codes[0].rawValue;

                if (code !== lastScanned) {

                    document.getElementById("studentLRN").value = code;

                    document.getElementById("status").innerText =
                        "Barcode detected: " + code;

                    lastScanned = code;

                }

            }

        }

    }

    catch(error) {

        console.log(error);

    }

    if (scanning) {

        setTimeout(scanBarcode, 500);

    }

}


// STOP CAMERA

function stopCamera() {

    scanning = false;

    if (stream) {

        stream.getTracks().forEach(function(track) {

            track.stop();

        });

        stream = null;

    }

    document.getElementById("camera").srcObject = null;

    document.getElementById("status").innerText =
        "Camera stopped.";

    lastScanned = "";

}


// CLEAR RECORDS

function clearRecords() {

    let confirmClear = confirm(
        "Are you sure you want to clear all records?"
    );

    if (confirmClear) {

        attendance = [];

        localStorage.removeItem("attendance");

        displayRecords();

        alert("All records have been cleared!");

    }

}


// DISPLAY SAVED RECORDS

displayRecords();

</script>

</body>
</html>
