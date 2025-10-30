<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Adaptive Running Planner</title>
<style>
  body {
    font-family: "Inter", Arial, sans-serif;
    margin: 0;
    background-color: #f8fbfb;
    color: #222;
  }
  header {
    background: linear-gradient(90deg, #3bb3bd, #1e90a2);
    color: white;
    padding: 1rem 2rem;
    text-align: center;
    font-size: 1.8rem;
    font-weight: 600;
  }
  .container {
    max-width: 900px;
    margin: 2rem auto;
    background: white;
    border-radius: 12px;
    box-shadow: 0 4px 10px rgba(0,0,0,0.1);
    padding: 2rem;
  }
  label { font-weight: 600; display:block; margin-top: 1rem;}
  input, select {
    width: 100%;
    padding: 0.6rem;
    border: 1px solid #ccc;
    border-radius: 8px;
    margin-top: 0.3rem;
  }
  button {
    background-color: #1e90a2;
    color: white;
    border: none;
    border-radius: 8px;
    padding: 0.8rem 1.2rem;
    margin-top: 1rem;
    cursor: pointer;
    font-size: 1rem;
  }
  button:hover { background-color: #167986; }
  #calendar {
    display: grid;
    grid-template-columns: repeat(7, 1fr);
    gap: 0.6rem;
    margin-top: 2rem;
  }
  .day {
    background-color: #f0f8f9;
    border-radius: 10px;
    padding: 0.5rem;
    font-size: 0.9rem;
    min-height: 90px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    overflow: hidden;
  }
  .day strong { display: block; font-weight: 700; color:#1e90a2; }
  .run { background-color: #e0f7f7; }
  .lift { background-color: #f6f0ff; }
  .cross { background-color: #fff9e0; }
  .rest { background-color: #f0f0f0; }
  .feedback {
    margin-top: 2rem;
    background: #f9f9ff;
    padding: 1rem;
    border-radius: 10px;
  }
</style>
</head>
<body>
<header>Adaptive Running Planner 🏃‍♀️</header>

<div class="container" id="setup">
  <h2>Build Your Plan</h2>
  <label>Preferred Units:</label>
  <select id="unit">
    <option value="miles" selected>Miles (Imperial)</option>
    <option value="km">Kilometers (Metric)</option>
  </select>

  <label>Current Easy Run Pace (min/mile or min/km):</label>
  <input type="text" id="currentPace" placeholder="e.g. 7:30" />

  <label>Race Distance:</label>
  <input type="text" id="raceDistance" placeholder="e.g. 5K, 10K, 6K, 1600m" />

  <label>Race Goal Time:</label>
  <input type="text" id="goalTime" placeholder="e.g. 19:30" />

  <label>Race Date:</label>
  <input type="date" id="raceDate" />

  <label>Start Date:</label>
  <input type="date" id="startDate" />

  <label>Weekly Training Days:</label>
  <input type="number" id="weeklyDays" min="3" max="7" value="5" />

  <label>Preferred Hard Days (comma separated, e.g. Tue,Thu):</label>
  <input type="text" id="hardDays" placeholder="e.g. Tue,Thu" />

  <label>Lift Days (comma separated, e.g. Mon,Fri):</label>
  <input type="text" id="liftDays" placeholder="e.g. Mon,Fri" />

  <label>Cross-Training Days (comma separated, e.g. Wed):</label>
  <input type="text" id="crossDays" placeholder="e.g. Wed" />

  <label>Rest Days (comma separated, e.g. Sun):</label>
  <input type="text" id="restDays" placeholder="e.g. Sun" />

  <button onclick="generatePlan()">Generate Plan</button>
</div>

<div class="container" id="plan" style="display:none;">
  <h2>Your Monthly Plan</h2>
  <div id="calendar"></div>
  <div class="feedback">
    <h3>Weekly Check-In</h3>
    <p>How did your week go?</p>
    <textarea id="feedbackInput" rows="3" style="width:100%;"></textarea>
    <button onclick="adjustPlan()">Submit Feedback</button>
  </div>
</div>

<script>
function parsePace(p) {
  const [min, sec] = p.split(":").map(Number);
  return min + sec/60;
}

function formatPace(mins) {
  const m = Math.floor(mins);
  const s = Math.round((mins - m)*60).toString().padStart(2,'0');
  return `${m}:${s}`;
}

function getDates(start, end) {
  const arr = [];
  let dt = new Date(start);
  while (dt <= end) {
    arr.push(new Date(dt));
    dt.setDate(dt.getDate()+1);
  }
  return arr;
}

function generatePlan() {
  const unit = document.getElementById("unit").value;
  const currentPace = parsePace(document.getElementById("currentPace").value);
  const startDate = new Date(document.getElementById("startDate").value);
  const raceDate = new Date(document.getElementById("raceDate").value);
  const hardDays = document.getElementById("hardDays").value.split(",").map(d=>d.trim());
  const liftDays = document.getElementById("liftDays").value.split(",").map(d=>d.trim());
  const crossDays = document.getElementById("crossDays").value.split(",").map(d=>d.trim());
  const restDays = document.getElementById("restDays").value.split(",").map(d=>d.trim());

  const allDates = getDates(startDate, raceDate);
  const calendar = document.getElementById("calendar");
  calendar.innerHTML = "";

  allDates.forEach((d, i)=>{
    const dayName = d.toLocaleDateString('en-US',{weekday:'short'});
    const div = document.createElement("div");
    div.classList.add("day");
    let type="run";
    if (restDays.includes(dayName)) type="rest";
    else if (liftDays.includes(dayName)) type="lift";
    else if (crossDays.includes(dayName)) type="cross";
    div.classList.add(type);
    div.innerHTML = `<strong>${dayName}</strong><small>${d.toLocaleDateString()}</small><br>${generateWorkout(type, currentPace, unit, i)}`;
    calendar.appendChild(div);
  });

  document.getElementById("setup").style.display="none";
  document.getElementById("plan").style.display="block";
}

function generateWorkout(type, pace, unit, dayIndex){
  if (type==="rest") return "Rest or light mobility work";
  if (type==="lift") return "Strength training (lower + core focus)";
  if (type==="cross") return "Cross-train: bike/elliptical/stairmaster 45–60 min";
  // run logic
  const easy = formatPace(pace+0.5);
  const tempo = formatPace(pace-0.3);
  const interval = formatPace(pace-0.8);
  const miles = unit==="miles" ? "mi" : "km";
  if (dayIndex%7===0) return `Long run: ${unit==="miles"?8:13} ${miles} @ ${easy}/mi`;
  else if (dayIndex%7===2) return `Intervals: 5×800m @ ${interval}/mi`;
  else if (dayIndex%7===4) return `Tempo: 4 ${miles} @ ${tempo}/mi`;
  return `Easy run: ${unit==="miles"?5:8} ${miles} @ ${easy}/mi`;
}

function adjustPlan(){
  const feedback = document.getElementById("feedbackInput").value.toLowerCase();
  if(feedback.includes("tired")||feedback.includes("hard")){
    alert("We'll reduce mileage slightly for next week to aid recovery.");
  } else if(feedback.includes("easy")||feedback.includes("great")){
    alert("Nice! We'll increase intensity slightly next week.");
  } else {
    alert("Thanks for your feedback — adjustments saved!");
  }
  document.getElementById("feedbackInput").value="";
}
</script>

</body>
</html>
