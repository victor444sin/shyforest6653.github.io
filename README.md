<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Teachable Machine HTTP Trigger</title>
<style>
body{font-family:sans-serif;text-align:center;margin-top:30px}
#webcam-container canvas{border-radius:12px}
#status{margin-top:20px;font-size:18px;color:#333}
.hit{color:green;font-weight:bold}
#scores{margin-top:10px;font-size:14px;color:#555}
</style>
</head>
<body>
<h2>Teachable Machine Detection &rarr; HTTP Send</h2>
<button onclick="init()">Start Camera &amp; Detection</button>
<div id="webcam-container"></div>
<div id="status">Waiting to start...</div>
<div id="scores"></div>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest/dist/teachablemachine-image.min.js"></script>
<script>
const MODEL_URL="https://teachablemachine.withgoogle.com/models/goyru8LbI/";
const TARGET_CLASS="Class 1";
const POST_URL="https://httpbin.org/post";
const MIN_CONFIDENCE=0.90;
const STABLE_FRAMES=5;
let model,webcam,maxPredictions,stableCount=0,hasSent=false;
async function init(){
const modelURL=MODEL_URL+"model.json";
const metadataURL=MODEL_URL+"metadata.json";
try{model=await tmImage.load(modelURL,metadataURL);maxPredictions=model.getTotalClasses();}
catch(e){document.getElementById("status").innerText="Failed to load model: "+e.message;return;}
const flip=true;
webcam=new tmImage.Webcam(300,300,flip);
await webcam.setup();
await webcam.play();
document.getElementById("webcam-container").appendChild(webcam.canvas);
document.getElementById("status").innerText="Detecting... show your object to the camera.";
window.requestAnimationFrame(loop);
}
async function loop(){webcam.update();await predictAndSend();window.requestAnimationFrame(loop);}
async function predictAndSend(){
const prediction=await model.predict(webcam.canvas);
let scoresHTML="";
for(let i=0;i<maxPredictions;i++){scoresHTML+=prediction[i].className+": "+(prediction[i].probability*100).toFixed(0)+"% &nbsp; ";}
document.getElementById("scores").innerHTML=scoresHTML;
let bestClass="",bestProb=0;
for(let i=0;i<maxPredictions;i++){if(prediction[i].probability>bestProb){bestProb=prediction[i].probability;bestClass=prediction[i].className;}}
const statusEl=document.getElementById("status");
if(bestClass!==TARGET_CLASS||bestProb<MIN_CONFIDENCE){stableCount=0;hasSent=false;statusEl.innerHTML="Current: "+bestClass+" ("+(bestProb*100).toFixed(0)+"%) &mdash; no trigger";return;}
stableCount++;
statusEl.innerHTML="Target hit! Confidence "+(bestProb*100).toFixed(0)+"%, stable "+stableCount+"/"+STABLE_FRAMES;
if(stableCount<STABLE_FRAMES)return;
if(hasSent){statusEl.innerHTML+=' <span class="hit">(already sent, waiting for reset)</span>';return;}
try{
const response=await fetch(POST_URL,{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify({trigger:"buzzer",className:bestClass,confidence:parseFloat(bestProb.toFixed(3)),timestamp:new Date().toISOString()})});
hasSent=true;
if(response.ok){statusEl.innerHTML="&#9989; Sent! HTTP "+response.status;}
else{statusEl.innerHTML="&#9888; Request sent, but server returned "+response.status;}
}catch(err){statusEl.innerHTML="&#10060; Send failed: "+err.message;hasSent=false;}
}
</script>
</body>
</html>
