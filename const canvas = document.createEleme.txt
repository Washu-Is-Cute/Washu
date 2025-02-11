const canvas = document.createElement("canvas");
document.body.appendChild(canvas);
const ctx = canvas.getContext("2d");

canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

let spears = [];
let enemies = [];
let player = { x: canvas.width / 2, y: canvas.height - 50 };
let combo = 0;
let bossTimer = null;
let gameOver = false;
let spearDamage = 1;
let enemyHealth = 1;
let bossHealth = 100;
let bossActive = false;
let bossOpacity = 1;
let lastThrowTime = 0;
let throwCooldown = 100; // Default throw cooldown
let currentDifficulty = "easy"; // Default difficulty
let highScore = 0; // Track high score
let paused = false; // Flag to track if the game is paused
let enemySpeed = 1.5; // Default enemy speed

// Difficulty settings
const difficultySettings = {
    easy: { enemyHealth: 1, spawnInterval: 1500, throwCooldown: 100, enemySpeed: 1.5, mode: "tap" },
    hard: { enemyHealth: 3, spawnInterval: 1200, throwCooldown: 100, enemySpeed: 2, mode: "tap" },
    insane: { enemyHealth: 5, spawnInterval: 800, throwCooldown: 0, enemySpeed: 2.5, mode: "spear" }, // Spear throwing mode for insane
};

// Start screen state
let onStartScreen = true;

// Start screen rendering
function drawStartScreen() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    ctx.fillStyle = "black";
    ctx.font = "40px Arial";
    ctx.fillText("Press Enter to Start", canvas.width / 2 - 120, canvas.height / 2 - 40);
    ctx.font = "30px Arial";
    ctx.fillText(`High Score: ${highScore}`, canvas.width / 2 - 90, canvas.height / 2);
    ctx.fillText("Select Difficulty", canvas.width / 2 - 110, canvas.height / 2 + 50);
    ctx.fillText("Easy (1)", canvas.width / 2 - 60, canvas.height / 2 + 100);
    ctx.fillText("Hard (2)", canvas.width / 2 - 60, canvas.height / 2 + 150);
    ctx.fillText("Insane (3)", canvas.width / 2 - 60, canvas.height / 2 + 200);
    ctx.fillText("Press R to Reset", canvas.width / 2 - 100, canvas.height / 2 + 250);
}

// Initialize the game based on difficulty
function startGame() {
    onStartScreen = false;
    enemies = [];
    spears = [];
    player = { x: canvas.width / 2, y: canvas.height - 50 };
    combo = 0;
    gameOver = false;
    spearDamage = 1;
    bossActive = false;
    bossOpacity = 1;
    lastThrowTime = 0;
    
    // Apply the selected difficulty settings
    enemyHealth = difficultySettings[currentDifficulty].enemyHealth;
    throwCooldown = difficultySettings[currentDifficulty].throwCooldown;
    enemySpeed = difficultySettings[currentDifficulty].enemySpeed; // Ensure the speed is set correctly for the difficulty
    
    setInterval(spawnEnemy, difficultySettings[currentDifficulty].spawnInterval); // Adjust spawn interval based on difficulty
    gameLoop(); // Start the game loop
}

// Spawn enemies from the left or right
function spawnEnemy() {
    if (bossActive || paused) return; // Stop normal enemies from spawning when the game is paused
    
    let fromLeft = Math.random() < 0.5;
    let x = fromLeft ? 0 : canvas.width;
    let speed = fromLeft ? enemySpeed : -enemySpeed; // Use the speed from difficulty settings

    if (combo > 0 && combo % 50 === 0 && !bossActive) {
        let boss = { x: canvas.width / 2, y: 50, speed: 0, health: bossHealth, boss: true, size: 200 }; // 2.5x size for boss
        enemies = [boss]; // Clear existing enemies
        bossActive = true;
        startBossTimer();
    } else {
        enemies.push({ x, y: Math.random() * canvas.height, speed, health: enemyHealth, boss: false, size: 75 }); // 2.5x size for regular enemies
    }
}

// Start boss timer
function startBossTimer() {
    if (bossTimer) clearTimeout(bossTimer);
    bossTimer = setTimeout(() => {
        if (enemies.some(enemy => enemy.boss)) {
            gameOver = true;
            fadeOutBoss();
        }
    }, 20000); // 20 seconds to defeat boss
}

// Fade out boss on failure
function fadeOutBoss() {
    let fadeInterval = setInterval(() => {
        bossOpacity -= 0.05;
        if (bossOpacity <= 0) {
            clearInterval(fadeInterval);
            enemies = [];
            bossActive = false; // After boss fades out, allow enemies to spawn again
        }
    }, 100);
}

// Handle mouse clicks to hit blocks (tapping on the screen)
canvas.addEventListener("click", (event) => {
    if (gameOver || paused) return;
    
    const x = event.clientX;
    const y = event.clientY;

    if (difficultySettings[currentDifficulty].mode === "tap") {
        // Check if the click is inside any enemy block
        enemies.forEach((enemy, index) => {
            const dx = x - enemy.x;
            const dy = y - enemy.y;
            if (Math.sqrt(dx * dx + dy * dy) < enemy.size / 2) {
                // If the click hits an enemy, reduce its health
                enemy.health -= spearDamage;
                // If the enemy is dead, remove it and update combo
                if (enemy.health <= 0) {
                    enemies.splice(index, 1);
                    combo++;
                    if (enemy.boss) {
                        clearTimeout(bossTimer);
                        bossActive = false;
                        spearDamage++;
                        enemyHealth += 2;
                        bossHealth += 50;
                    }
                }
            }
        });
    }
});

// Handle spear throwing (only for insane difficulty)
canvas.addEventListener("click", (event) => {
    if (gameOver || paused) return;

    if (difficultySettings[currentDifficulty].mode === "spear") {
        const currentTime = Date.now();
        if (currentTime - lastThrowTime > throwCooldown) {
            lastThrowTime = currentTime;

            let spear = { x: player.x, y: player.y, speed: 5, damage: spearDamage };
            spears.push(spear);
        }
    }
});

// Move spears and check for collisions
function moveSpears() {
    spears.forEach((spear, index) => {
        spear.y -= spear.speed; // Move spear upward

        // Check for collisions with enemies
        enemies.forEach((enemy, eIndex) => {
            const dx = spear.x - enemy.x;
            const dy = spear.y - enemy.y;
            if (Math.sqrt(dx * dx + dy * dy) < enemy.size / 2) {
                enemy.health -= spear.damage;

                // If the enemy is dead, remove it and update combo
                if (enemy.health <= 0) {
                    enemies.splice(eIndex, 1);
                    combo++;
                    if (enemy.boss) {
                        clearTimeout(bossTimer);
                        bossActive = false;
                        spearDamage++;
                        enemyHealth += 2;
                        bossHealth += 50;
                    }
                }

                // Remove the spear after collision
                spears.splice(index, 1);
            }
        });

        // Remove spear if it goes off screen
        if (spear.y < 0) spears.splice(index, 1);
    });
}

// Pause the game if "P" is pressed
document.addEventListener("keydown", (event) => {
    if (event.key === "p" || event.key === "P") {
        paused = !paused; // Toggle pause state
    }

    // Reset the game if "R" is pressed
    if (event.key === "r" || event.key === "R") {
        resetGame();
    }
});

// Reset the game
function resetGame() {
    highScore = Math.max(highScore, combo); // Save high score
    combo = 0;
    onStartScreen = true;
    enemies = [];
    spears = [];
    bossActive = false;
    gameOver = false;
    paused = false;
    lastThrowTime = 0;
    bossOpacity = 1;
    bossHealth = 100;
    enemySpeed = difficultySettings[currentDifficulty].enemySpeed; // Ensure speed resets correctly
    drawStartScreen();
}

// Mobile Touch Events for Difficulty Selection
canvas.addEventListener("touchstart", (event) => {
    event.preventDefault(); // Prevent default touch behavior

    const touch = event.touches[0];
    const x = touch.clientX;
    const y = touch.clientY;

    if (onStartScreen) {
        if (y > canvas.height / 2 + 100 && y < canvas.height / 2 + 140) {
            currentDifficulty = "easy";
            startGame();
        } else if (y > canvas.height / 2 + 150 && y < canvas.height / 2 + 190) {
            currentDifficulty = "hard";
            startGame();
        } else if (y > canvas.height / 2 + 200 && y < canvas.height / 2 + 240) {
            currentDifficulty = "insane";
            startGame();
        }
    }

    // Pause/Unpause if touched on the pause button area
    const pauseButtonArea = { x: canvas.width - 100, y: 20, width: 80, height: 40 };
    if (
        x > pauseButtonArea.x && x < pauseButtonArea.x + pauseButtonArea.width &&
        y > pauseButtonArea.y && y < pauseButtonArea.y + pauseButtonArea.height
    ) {
        paused = !paused;
    } else {
        // Handle tapping blocks for damage on mobile
        if (difficultySettings[currentDifficulty].mode === "tap") {
            enemies.forEach((enemy, index) => {
                const dx = x - enemy.x;
                const dy = y - enemy.y;
                if (Math.sqrt(dx * dx + dy * dy) < enemy.size / 2) {
                    enemy.health -= spearDamage;
                    if (enemy.health <= 0) {
                        enemies.splice(index, 1);
                        combo++;
                        if (enemy.boss) {
                            clearTimeout(bossTimer);
                            bossActive = false;
                            spearDamage++;
                            enemyHealth += 2;
                            bossHealth += 50;
                        }
                    }
                }
            });
        }
    }

    // Mobile Reset Button
    const resetButtonArea = { x: canvas.width / 2 - 100, y: canvas.height / 2 + 270, width: 200, height: 40 };
    if (
        x > resetButtonArea.x && x < resetButtonArea.x + resetButtonArea.width &&
        y > resetButtonArea.y && y < resetButtonArea.y + resetButtonArea.height
    ) {
        resetGame();
    }
});

// Draw the game state
function draw() {
    if (onStartScreen) {
        drawStartScreen();
        return;
    }

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    if (gameOver) {
        ctx.fillStyle = "red";
        ctx.font = "40px Arial";
        ctx.fillText("Game Over", canvas.width / 2 - 100, canvas.height / 2);
        ctx.fillText("Press Enter to Restart", canvas.width / 2 - 140, canvas.height / 2 + 40);
        return;
    }

    // Draw player
    ctx.fillStyle = "blue";
    ctx.fillRect(player.x - 15, player.y - 15, 30, 30);

    // Draw enemies
    enemies.forEach(enemy => {
        ctx.fillStyle = enemy.boss ? `rgba(128, 0, 128, ${bossOpacity})` : "red";
        ctx.fillRect(enemy.x - enemy.size / 2, enemy.y - enemy.size / 2, enemy.size, enemy.size);
        
        // Draw boss health bar
        if (enemy.boss) {
            let healthBarWidth = (enemy.health / bossHealth) * 100; // Scale health bar width
            ctx.fillStyle = "green";
            ctx.fillRect(enemy.x - enemy.size / 2, enemy.y - enemy.size / 2 - 10, healthBarWidth, 5);
        }
    });

    // Draw combo counter
    ctx.fillStyle = "black";
    ctx.font = "20px Arial";
    ctx.fillText("Combo: " + combo, 10, 30);

    // Draw pause text if the game is paused
    if (paused) {
        ctx.fillStyle = "black";
        ctx.font = "40px Arial";
        ctx.fillText("PAUSED", canvas.width / 2 - 80, canvas.height / 2);
        ctx.fillText("Press P to Resume", canvas.width / 2 - 120, canvas.height / 2 + 40);
    }

    // Draw reset button for mobile
    ctx.fillStyle = "gray";
    ctx.fillRect(canvas.width / 2 - 100, canvas.height / 2 + 270, 200, 40);
    ctx.fillStyle = "white";
    ctx.font = "20px Arial";
    ctx.fillText("Reset Game", canvas.width / 2 - 80, canvas.height / 2 + 300);
}

// Update game objects
function update() {
    if (gameOver || paused) {
        return; // Skip updates if game is over or paused
    }

    // Move spears if in spear mode
    if (difficultySettings[currentDifficulty].mode === "spear") {
        moveSpears();
    }

    // Update enemies
    enemies.forEach((enemy, eIndex) => {
        // Move enemies across the screen
        enemy.x += enemy.speed;

        // If the enemy moves off-screen, remove it
        if (enemy.x < 0 || enemy.x > canvas.width) {
            enemies.splice(eIndex, 1);
        }
    });

    draw();
    requestAnimationFrame(update);
}

startGame();
