<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>iPad Arcade Volleyball - Local Wireless</title>
    <!-- PeerJS Library for serverless device-to-device multiplayer -->
    <script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
    <style>
        body { margin: 0; background: #1a1a1a; color: white; font-family: monospace; display: flex; flex-direction: column; align-items: center; height: 100vh; overflow: hidden; touch-action: none; -webkit-user-select: none; user-select: none; }
        #netBar { background: #222; width: 95%; padding: 8px; display: flex; justify-content: center; gap: 10px; align-items: center; border-bottom: 2px solid #444; box-sizing: border-box; font-size: 13px; }
        #roomInput { background: #333; color: white; border: 1px solid #555; padding: 6px; border-radius: 6px; width: 140px; font-family: monospace; text-align: center; }
        .net-btn { border: none; padding: 6px 12px; font-weight: bold; border-radius: 6px; color: white; cursor: pointer; }
        #btnHost { background: #2e7d32; } #btnJoin { background: #0288d1; }
        #hud { display: flex; justify-content: space-between; align-items: center; width: 95%; padding: 10px; background: #2a2a2a; border-bottom: 3px solid #444; box-sizing: border-box; }
        #coinCount { color: #ffd700; font-size: 20px; font-weight: bold; }
        #scoreBoard { color: #fff; font-size: 20px; font-weight: bold; }
        #rollAlert { color: #ffeb3b; font-size: 14px; text-align: center; flex-grow: 1; }
        .gacha-btn { padding: 10px 15px; font-size: 14px; font-weight: bold; border-radius: 8px; border: none; color: white; margin-left: 10px; cursor: pointer; }
        #normalSpinBtn { background: #2196f3; } #luckySpinBtn { background: #e91e63; }
        #gameArea { flex-grow: 1; display: flex; justify-content: center; align-items: center; width: 100%; position: relative; }
        canvas { background: linear-gradient(to bottom, #0f385a, #15578a); border-left: 4px solid white; border-right: 4px solid white; max-width: 100%; height: 100%; object-fit: contain; }
        #controllerArea { height: 160px; width: 100%; background: #222; display: flex; justify-content: space-between; align-items: center; padding: 0 20px; box-sizing: border-box; border-top: 4px solid #444; }
        .d-pad { display: grid; grid-template-columns: repeat(3, 65px); grid-template-rows: repeat(2, 65px); gap: 8px; transition: opacity 0.3s ease; }
        .touch-btn { background: #333; color: white; border: 3px solid #666; border-radius: 12px; font-weight: bold; font-size: 16px; display: flex; align-items: center; justify-content: center; box-shadow: 0 5px 0 #111; }
        .touch-btn:active { background: #555; transform: translateY(5px); box-shadow: none; }
        .p1-btn { border-color: #ea4335; color: #ea4335; } .p2-btn { border-color: #34a853; color: #34a853; }
    </style>
</head>
<body>
    <div id="netBar">
        <input id="roomInput" placeholder="Room Code (e.g., 555)">
        <button id="btnHost" class="net-btn">HOST (P1)</button>
        <button id="btnJoin" class="net-btn">JOIN (P2)</button>
        <span id="netStatus" style="color: #bbb; margin-left: 10px;">Mode: Single Device Shared Screen</span>
    </div>

    <div id="hud">
        <div>🪙 <span id="coinCount">100</span></div>
        <div id="rollAlert">Tap spins to roll characters!</div>
        <div id="scoreBoard">P1: 0 | P2: 0</div>
        <div>
            <button id="normalSpinBtn" class="gacha-btn">🌀 SPIN (100)</button>
            <button id="luckySpinBtn" class="gacha-btn">🔥 LUCKY (200)</button>
        </div>
    </div>
    <div id="gameArea"><canvas id="gameCanvas" width="900" height="400"></canvas></div>
    <div id="controllerArea">
        <div class="d-pad" id="p1-controls">
            <div class="touch-btn p1-btn" id="btn-p1-jump" style="grid-column: 2; grid-row: 1;">JMP</div>
            <div class="touch-btn p1-btn" id="btn-p1-left" style="grid-column: 1; grid-row: 2;">◀</div>
            <div class="touch-btn p1-btn" id="btn-p1-bump" style="grid-column: 2; grid-row: 2;">HIT</div>
            <div class="touch-btn p1-btn" id="btn-p1-right" style="grid-column: 3; grid-row: 2;">▶</div>
            <div class="touch-btn p1-btn" id="btn-p1-dive" style="grid-column: 3; grid-row: 1; font-size:11px;">DIVE</div>
        </div>
        <div class="d-pad" id="p2-controls">
            <div class="touch-btn p2-btn" id="btn-p2-jump" style="grid-column: 2; grid-row: 1;">JMP</div>
            <div class="touch-btn p2-btn" id="btn-p2-left" style="grid-column: 1; grid-row: 2;">◀</div>
            <div class="touch-btn p2-btn" id="btn-p2-bump" style="grid-column: 2; grid-row: 2;">HIT</div>
            <div class="touch-btn p2-btn" id="btn-p2-right" style="grid-column: 3; grid-row: 2;">▶</div>
            <div class="touch-btn p2-btn" id="btn-p2-dive" style="grid-column: 1; grid-row: 1; font-size:11px;">DIVE</div>
        </div>
    </div>
<script>
document.addEventListener('contextmenu', e => e.preventDefault());
const canvas = document.getElementById("gameCanvas"); const ctx = canvas.getContext("2d");
const scoreBoard = document.getElementById("scoreBoard"); const coinCountDisplay = document.getElementById("coinCount"); const rollAlert = document.getElementById("rollAlert");

// Networking Setup States
let role = "local"; // "local", "host", or "client"
let peer = null; let conn = null;
const netStatus = document.getElementById("netStatus");
const roomInput = document.getElementById("roomInput");

let userCoins = 100;
const characterDatabase = {
    "Rare": { color: "#9e9e9e", pool: ["Nishinoya", "Tanaka", "Kuroo"] },
    "Legendary": { color: "#00e676", pool: ["Bokuto", "Oikawa", "Kageyama"] },
    "Mythical": { color: "#29b6f6", pool: ["Atsumu", "Hoshiumi", "TS Hinata"] },
    "Godly": { color: "#ff9100", pool: ["Ushijima", "Sakusa"] },
    "Secret": { color: "#d500f9", pool: ["Jackals Hinata", "Adlers Kageyama"] },
    "OG": { color: "#ffea00", pool: ["Classic Hinata", "Classic Kuroo"] }
};
let p1ActiveSkin = { name: "Default Left", color: "#ea4335" }; let p2ActiveSkin = { name: "Default Right", color: "#34a853" };
const gravity = 0.6; const friction = 0.85; let score1 = 0; let score2 = 0; let p1Touches = 0; let p2Touches = 0; let lastTouchPlayer = null; let gameOver = false; let winner = ""; let shakeIntensity = 0; let shakeDuration = 0;
const net = { x: canvas.width / 2 - 5, y: canvas.height - 140, width: 10, height: 140 };
const touchStates = { p1Left: false, p1Right: false, p2Left: false, p2Right: false };

class Player {
    constructor(x, y, isPlayerLeft) { this.startX = x; this.startY = y; this.x = x; this.y = y; this.radius = 40; this.speed = 7; this.vx = 0; this.vy = 0; this.isLeft = isPlayerLeft; this.isJumping = false; this.isDiving = false; this.diveTimer = 0; this.actionCooldown = 0; this.activeSkill = null; }
    update() {
        if (this.isDiving) { this.diveTimer--; this.vx = this.isLeft ? 15 : -15; this.vy = 0; if (this.diveTimer <= 0) { this.isDiving = false; this.vx = 0; } } else { this.vy += gravity; this.vx *= friction; }
        this.x += this.vx; this.y += this.vy; if (this.actionCooldown > 0) this.actionCooldown--;
        if (this.y >= canvas.height - this.radius) { this.y = canvas.height - this.radius; this.vy = 0; this.isJumping = false; }
        if (this.isLeft) { if (this.x - this.radius < 0) { this.x = this.radius; this.vx = 0; } if (this.x + this.radius > net.x) { this.x = net.x - this.radius; this.vx = 0; } } 
        else { if (this.x - this.radius < net.x + net.width) { this.x = net.x + net.width + this.radius; this.vx = 0; } if (this.x + this.radius > canvas.width) { this.x = canvas.width - this.radius; this.vx = 0; } }
    }
    draw() {
        let currentSkin = this.isLeft ? p1ActiveSkin : p2ActiveSkin; ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, Math.PI, 0, false); ctx.fillStyle = currentSkin.color; ctx.fill(); ctx.closePath();
        ctx.beginPath(); ctx.arc(this.isLeft ? this.x + 12 : this.x - 12, this.y - 15, 6, 0, Math.PI * 2); ctx.fillStyle = "white"; ctx.fill(); ctx.closePath();
        ctx.font = "bold 14px sans-serif"; ctx.fillStyle = "#fff"; ctx.textAlign = "center"; ctx.fillText(currentSkin.name.toUpperCase(), this.x, this.y - this.radius - 10);
    }
    reset() { this.x = this.startX; this.y = this.startY; this.vx = 0; this.vy = 0; this.isJumping = false; this.isDiving = false; this.activeSkill = null; }
}

const ball = {
    x: 225, y: 100, radius: 14, vx: 3, vy: 0, color: "#ffffff",
    update() {
        this.vy += gravity * 0.62; this.x += this.vx; this.y += this.vy;
        if (this.x - this.radius < 0 || this.x + this.radius > canvas.width) { if (lastTouchPlayer) awardPoint(lastTouchPlayer === 1 ? 2 : 1); else awardPoint(this.x < canvas.width / 2 ? 2 : 1); return; }
        if (this.x + this.radius > net.x && this.x - this.radius < net.x + net.width && this.y + this.radius > net.y) { if (this.vx > 0 && this.x < net.x) { this.x = net.x - this.radius; this.vx = -this.vx * 0.8; } else if (this.vx < 0 && this.x > net.x + net.width) { this.x = net.x + net.width + this.radius; this.vx = -this.vx * 0.8; } else if (this.vy > 0 && this.y < net.y) { this.y = net.y - this.radius; this.vy = -this.vy * 0.8; } }
        if (this.y >= canvas.height - this.radius) { if (this.x < canvas.width / 2) awardPoint(2); else awardPoint(1); }
    },
    draw() { ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2); ctx.fillStyle = this.color; ctx.fill(); ctx.strokeStyle = "#000"; ctx.lineWidth = 2; ctx.stroke(); ctx.closePath(); },
    reset(servingPlayer) { this.x = servingPlayer === 1 ? 225 : 675; this.y = 100; this.vx = servingPlayer === 1 ? 3 : -3; this.vy = -5; p1Touches = 0; p2Touches = 0; lastTouchPlayer = null; }
};

const p1 = new Player(225, canvas.height, true); const p2 = new Player(675, canvas.height, false);
function triggerScreenShake(intensity, duration) { shakeIntensity = intensity; shakeDuration = duration; }

function triggerSkillAction(playerId, actionType) {
    if (role === "client") {
        if (conn && conn.open) conn.send({ type: "action", action: actionType });
        return;
    }
    if (gameOver) { if (actionType === "jump") { score1 = 0; score2 = 0; gameOver = false; scoreBoard.textContent = `P1: 0 | P2: 0`; p1.reset(); p2.reset(); ball.reset(1); } return; }
    let player = (playerId === 1) ? p1 : p2;
    if (!player.isDiving && player.actionCooldown === 0) { if (!player.isJumping) { if (actionType === "jump") { player.vy = -14; player.isJumping = true; } if (actionType === "bump") player.activeSkill = "bump"; if (actionType === "dive") player.activeSkill = "dive"; } else { if (actionType === "bump") player.activeSkill = "spike"; } }
}

function processPlayerSkills(player) {
    if (!player.activeSkill) return;
    if (player.activeSkill === "dive") { player.isDiving = true; player.diveTimer = 16; player.actionCooldown = 30; player.activeSkill = null; return; }
    let dx = ball.x - player.x; let dy = ball.y - player.y; let dist = Math.sqrt(dx * dx + dy * dy); let hitRange = player.radius + ball.radius + 40;
    if (dist < hitRange) { player.actionCooldown = 20; if (registerTouch(player)) { if (player.activeSkill === "bump") { ball.vx = player.isLeft ? 4 : -4; ball.vy = -15; triggerScreenShake(3, 8); } else if (player.activeSkill === "spike") { ball.vx = player.isLeft ? 16 : -16; ball.vy = 8; triggerScreenShake(8, 15); } } }
    player.activeSkill = null;
}

function handleMovement() { if (gameOver) return; if (!p1.isDiving) { if (touchStates.p1Left) p1.vx = -p1.speed; if (touchStates.p1Right) p1.vx = p1.speed; } if (!p2.isDiving) { if (touchStates.p2Left) p2.vx = -p2.speed; if (touchStates.p2Right) p2.vx = p2.speed; } }
function checkPhysicalCollisions(player) { let dx = ball.x - player.x; let dy = ball.y - player.y; let distance = Math.sqrt(dx * dx + dy * dy); if (distance < player.radius + ball.radius && ball.y <= player.y + 10) { let angle = Math.atan2(dy, dx); ball.x = player.x + Math.cos(angle) * (player.radius + ball.radius); ball.y = player.y + Math.sin(angle) * (player.radius + ball.radius); if (registerTouch(player)) { ball.vx = Math.cos(angle) * 10 + player.vx * 0.3; ball.vy = Math.sin(angle) * 10; } } }
function registerTouch(player) { const currentPlayerId = player.isLeft ? 1 : 2; if (lastTouchPlayer === currentPlayerId) { awardPoint(player.isLeft ? 2 : 1); return false; } lastTouchPlayer = currentPlayerId; if (player.isLeft) { p2Touches = 0; p1Touches++; if (p1Touches > 3) { awardPoint(2); return false; } } else { p1Touches = 0; p2Touches++; if (p2Touches > 3) { awardPoint(1); return false; } } return true; }
function awardPoint(winnerSide) { if (winnerSide === 1) score1++; else score2++; userCoins += 50; coinCountDisplay.textContent = userCoins; scoreBoard.textContent = `P1: ${score1} | P2: ${score2}`; if (score1 >= 15) { gameOver = true; winner = "PLAYER 1 WINS! TAP JMP TO RESTART"; } else if (score2 >= 15) { gameOver = true; winner = "PLAYER 2 WINS! TAP JMP TO RESTART"; } else { p1.reset(); p2.reset(); ball.reset(winnerSide === 1 ? 2 : 1); } }

function bindTouchBtn(elementId, pressCallback, releaseCallback) { const el = document.getElementById(elementId); el.addEventListener("touchstart", (e) => { e.preventDefault(); pressCallback(); }, {passive: false}); if (releaseCallback) { el.addEventListener("touchend", (e) => { e.preventDefault(); releaseCallback(); }, {passive: false}); el.addEventListener("touchcancel", (e) => { e.preventDefault(); releaseCallback(); }, {passive: false}); } }

function transmitClientInputs() { if (role === "client" && conn && conn.open) { conn.send({ type: "input", p2Left: touchStates.p2Left, p2Right: touchStates.p2Right }); } }

bindTouchBtn("btn-p1-left", () => touchStates.p1Left = true, () => touchStates.p1Left = false); bindTouchBtn("btn-p1-right", () => touchStates.p1Right = true, () => touchStates.p1Right = false); bindTouchBtn("btn-p1-jump", () => triggerSkillAction(1, "jump")); bindTouchBtn("btn-p1-bump", () => triggerSkillAction(1, "bump")); bindTouchBtn("btn-p1-dive", () => triggerSkillAction(1, "dive"));
bindTouchBtn("btn-p2-left", () => { touchStates.p2Left = true; transmitClientInputs(); }, () => { touchStates.p2Left = false; transmitClientInputs(); }); bindTouchBtn("btn-p2-right", () => { touchStates.p2Right = true; transmitClientInputs(); }, () => { touchStates.p2Right = false; transmitClientInputs(); }); bindTouchBtn("btn-p2-jump", () => triggerSkillAction(2, "jump")); bindTouchBtn("btn-p2-bump", () => triggerSkillAction(2, "bump")); bindTouchBtn("btn-p2-dive", () => triggerSkillAction(2, "dive"));

function handleSpin(type, cost) {
    if (role === "client") { if (conn && conn.open) conn.send({ type: "spin", spinType: type, cost: cost }); return; }
    if (userCoins < cost) { rollAlert.innerHTML = `❌ <span style="color:red">No coins!</span>`; return; } userCoins -= cost; coinCountDisplay.textContent = userCoins; let rng = Math.random() * 100; let selectedTier = "Rare"; if (type === "normal") { if (rng < 70) selectedTier = "Rare"; else if (rng < 95) selectedTier = "Legendary"; else selectedTier = "Mythical"; } else { if (rng < 75) selectedTier = "Godly"; else if (rng < 95) selectedTier = "Secret"; else selectedTier = "OG"; } const poolData = characterDatabase[selectedTier]; const characterName = poolData.pool[Math.floor(Math.random() * poolData.pool.length)]; p1ActiveSkin = { name: characterName, color: poolData.color }; p2ActiveSkin = { name: characterName, color: poolData.color }; rollAlert.innerHTML = `🎉 <span style="color: ${poolData.color}">${characterName} (${selectedTier})</span>!`;
}
bindTouchBtn("normalSpinBtn", () => handleSpin("normal", 100)); bindTouchBtn("luckySpinBtn", () => handleSpin("lucky", 200));

document.getElementById("btnHost").addEventListener("click", () => {
    const code = roomInput.value.trim(); if (!code) { alert("Enter a room number code first!"); return; }
    netStatus.textContent = "Setting up server link...";
    peer = new Peer('vball-room-' + code);
    peer.on('open', () => {
        role = "host"; netStatus.textContent = `Hosting Room: ${code}. Get P2 to join!`;
        document.getElementById("p2-controls").style.opacity = "0";
        document.getElementById("p2-controls").style.pointerEvents = "none";
    });
    peer.on('connection', (c) => {
        conn = c; netStatus.textContent = `Player 2 Connected! Game active.`;
        conn.on('data', (data) => {
            if (data.type === "input") { touchStates.p2Left = data.p2Left; touchStates.p2Right = data.p2Right; }
            if (data.type === "action") { triggerSkillAction(2, data.action); }
            if (data.type === "spin") { handleSpin(data.spinType, data.cost); }
        });
    });
    peer.on('error', (err) => { alert("Room code taken! Try a different number code."); netStatus.textContent = "Mode: Single Device"; });
});

document.getElementById("btnJoin").addEventListener("click", () => {
    const code = roomInput.value.trim(); if (!code) { alert("Enter matching room number code first!"); return; }
    netStatus.textContent = "Joining host game...";
    peer = new Peer();
    peer.on('open', () => {
        conn = peer.connect('vball-room-' + code);
        conn.on('open', () => {
            role = "client"; netStatus.textContent = `Connected to Room: ${code}!`;
            document.getElementById("p1-controls").style.opacity = "0";
            document.getElementById("p1-controls").style.pointerEvents = "none";
        });
        conn.on('data', (data) => {
            if (data.type === "state") {
                ball.x = data.ball.x; ball.y = data.ball.y; ball.color = data.ball.color;
                p1.x = data.p1.x; p1.y = data.p1.y; p1.isJumping = data.p1.isJumping; p1.isDiving = data.p1.isDiving;
                p2.x = data.p2.x; p2.y = data.p2.y; p2.isJumping = data.p2.isJumping; p2.isDiving = data.p2.isDiving;
                score1 = data.score1; score2 = data.score2; p1ActiveSkin = data.p1Skin; p2ActiveSkin = data.p2Skin;
                userCoins = data.coins; gameOver = data.gameOver; winner = data.winner;
                shakeDuration = data.shakeD; shakeIntensity = data.shakeI;
                scoreBoard.textContent = `P1: ${score1} | P2: ${score2}`; coinCountDisplay.textContent = userCoins;
                rollAlert.innerHTML = data.alertHtml;
            }
        });
    });
    peer.on('error', (err) => { alert("Failed to find room. Check number code."); netStatus.textContent = "Mode: Single Device"; });
});

function gameLoop() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.save();
    if (shakeDuration > 0) { ctx.translate((Math.random() - 0.5) * shakeIntensity, (Math.random() - 0.5) * shakeIntensity); shakeDuration--; }
    ctx.fillStyle = "#dcdcdc"; ctx.fillRect(net.x, net.y, net.width, net.height);
    ctx.fillStyle = "#ff1744"; ctx.fillRect(net.x, net.y, net.width, 10);

    if (role !== "client") {
        handleMovement(); p1.update(); p2.update();
        processPlayerSkills(p1); processPlayerSkills(p2);
        checkPhysicalCollisions(p1); checkPhysicalCollisions(p2);
        ball.update();

        if (role === "host" && conn && conn.open) {
            conn.send({
                type: "state",
                ball: { x: ball.x, y: ball.y, color: ball.color },
                p1: { x: p1.x, y: p1.y, isJumping: p1.isJumping, isDiving: p1.isDiving },
                p2: { x: p2.x, y: p2.y, isJumping: p2.isJumping, isDiving: p2.isDiving },
                score1: score1, score2: score2, p1Skin: p1ActiveSkin, p2Skin: p2ActiveSkin,
                coins: userCoins, gameOver: gameOver, winner: winner, shakeD: shakeDuration, shakeI: shakeIntensity,
                alertHtml: rollAlert.innerHTML
            });
        }
    }

    p1.draw(); p2.draw(); ball.draw();
    ctx.restore();
    if (gameOver) {
        ctx.fillStyle = "rgba(0, 0, 0, 0.85)"; ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "#ffd700"; ctx.font = "bold 26px sans-serif"; ctx.textAlign = "center";
        ctx.fillText(winner, canvas.width / 2, canvas.height / 2);
    }
    requestAnimationFrame(gameLoop);
}
gameLoop();
</script>
</body>
</html>
