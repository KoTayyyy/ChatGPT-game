<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Geometric Runner Clone</title>
    <!-- Load Tailwind CSS for styling and Tone.js for sound -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.min.js"></script>
    <style>
        /* Custom styles for the game background and cursor */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #111827; /* Dark charcoal background */
        }
        #gameCanvas {
            background: linear-gradient(180deg, #1f2937 0%, #030712 100%);
            border: 4px solid #f97316; /* Neon orange border */
            box-shadow: 0 0 25px rgba(249, 115, 22, 0.8);
            cursor: pointer;
            touch-action: manipulation; /* Prevents default touch actions */
            transition: box-shadow 0.2s;
        }
        .ui-glow {
            text-shadow: 0 0 10px #f97316;
        }
    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">

    <div class="w-full max-w-4xl text-white">
        <header class="text-center mb-6">
            <h1 class="text-4xl md:text-6xl font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-teal-400 to-cyan-500 ui-glow">
                Geometric Dash Deluxe
            </h1>
            <p id="levelDisplay" class="text-xl font-semibold text-fuchsia-400 mt-2">Level: Loading...</p>
        </header>

        <div class="relative flex justify-center">
            <!-- Game Canvas -->
            <canvas id="gameCanvas" width="800" height="300" class="w-full rounded-xl"></canvas>

            <!-- Game Over/Start Overlay -->
            <div id="overlay" class="absolute inset-0 flex flex-col items-center justify-center bg-gray-900 bg-opacity-95 rounded-xl transition-opacity duration-300">
                <h2 id="overlay-text" class="text-3xl md:text-5xl font-extrabold mb-8 ui-glow text-cyan-400 text-center"></h2>
                <button id="startButton" class="px-8 py-4 bg-fuchsia-600 hover:bg-fuchsia-700 text-white text-xl font-bold rounded-full shadow-2xl transition duration-150 transform hover:scale-105">
                    Start Game
                </button>
                <p class="mt-6 text-gray-400 text-sm">Music by Tone.js</p>
            </div>
        </div>

        <!-- Score and Control Info -->
        <div class="mt-6 flex justify-between items-center text-xl font-medium px-4">
            <p>Score: <span id="scoreDisplay" class="text-teal-400 ui-glow">0</span></p>
            <p>High Score: <span id="highScoreDisplay" class="text-fuchsia-400 ui-glow">0</span></p>
        </div>
    </div>

    <script>
        // --- Game Setup ---
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('scoreDisplay');
        const highScoreDisplay = document.getElementById('highScoreDisplay');
        const levelDisplay = document.getElementById('levelDisplay');
        const overlay = document.getElementById('overlay');
        const overlayText = document.getElementById('overlay-text');
        const startButton = document.getElementById('startButton');

        // Fixed Game Constants
        const GROUND_Y = canvas.height - 40;
        const PLAYER_SIZE = 30;
        const JUMP_VELOCITY = -12; 
        
        // Game State Variables
        let isRunning = false;
        let score = 0;
        let highScore = localStorage.getItem('geometricDashHighScore') || 0;
        let frame = 0;
        let player, obstacles, animationFrameId;

        // Level State Variables
        let currentLevelIndex = 0;
        let obstacleIndex = 0;
        let framesUntilNextObstacle = 0;
        
        highScoreDisplay.textContent = highScore;
        
        // --- Level Definitions (Obstacle Timelines and Configuration) ---

        // Helper function to create a gap (in frames)
        const g = (frames) => ({ type: 'gap', frameGap: frames });
        
        // Obstacle types: 's' (spike, size 20), 'b' (block, size 40)
        // Obstacle format: {type: 's'|'b', frameGap: number}
        const LEVELS = [
            // Level 1: EASY - The Chill Start
            {
                name: "1: The Chill Start (Easy)",
                speed: 4.0,
                gravity: 0.4,
                bgmNotes: ["C4", null, "E4", null, "G4", null, "C5", null],
                bgmTempo: "4n", // Quarter notes, slow and spacious
                timeline: [
                    g(150), { type: 's', frameGap: 200 },
                    g(50), { type: 'b', frameGap: 240 },
                    g(70), { type: 's', frameGap: 180 },
                    g(40), { type: 's', frameGap: 180 },
                    g(20), { type: 'b', frameGap: 250 },
                    g(10), { type: 's', frameGap: 220 },
                    g(10), { type: 's', frameGap: 220 },
                    g(30), { type: 'b', frameGap: 300 },
                    g(150), // End gap
                ]
            },
            // Level 2: MEDIUM - The Rhythmic Grind
            {
                name: "2: The Rhythmic Grind (Medium)",
                speed: 5.5,
                gravity: 0.5,
                bgmNotes: ["C4", "E4", "G4", "A4", "G4", "E4", "D4", "F4"],
                bgmTempo: "8n", // Eighth notes, faster pace
                timeline: [
                    g(120), { type: 's', frameGap: 150 },
                    g(30), { type: 's', frameGap: 100 }, // Double spike
                    g(120), { type: 'b', frameGap: 180 },
                    g(10), { type: 's', frameGap: 100 }, // Block + Spike
                    g(130), { type: 'b', frameGap: 100 },
                    g(10), { type: 'b', frameGap: 120 }, // Double Block
                    g(150), { type: 's', frameGap: 100 },
                    g(30), { type: 's', frameGap: 100 },
                    g(30), { type: 's', frameGap: 200 }, // Triple spike
                    g(120), // End gap
                ]
            },
            // Level 3: HARD - The Neon Nightmare
            {
                name: "3: The Neon Nightmare (Hard)",
                speed: 7.0,
                gravity: 0.6,
                bgmNotes: ["C5", "C5", "D5", "D5", "E5", "E5", "D5", "D5", "C5", "G4", "A4", "B4"],
                bgmTempo: "16n", // Sixteenth notes, very fast and challenging
                timeline: [
                    g(100), { type: 's', frameGap: 90 },
                    g(10), { type: 's', frameGap: 90 },
                    g(10), { type: 's', frameGap: 150 }, // Triple
                    g(20), { type: 'b', frameGap: 80 },
                    g(10), { type: 's', frameGap: 80 },
                    g(10), { type: 's', frameGap: 100 },
                    g(100), { type: 'b', frameGap: 70 },
                    g(10), { type: 'b', frameGap: 100 },
                    g(20), { type: 's', frameGap: 70 },
                    g(5), { type: 's', frameGap: 70 },
                    g(5), { type: 's', frameGap: 120 },
                    g(150), // End gap
                ]
            }
        ];
        
        let currentLevel = LEVELS[currentLevelIndex]; // Start on the first level

        // --- Audio Setup (Tone.js) ---
        let deathSynth;
        let synth;
        let musicLoop;

        function setupAudio() {
            try {
                // Setup Death Sound (White noise burst)
                deathSynth = new Tone.NoiseSynth({
                    noise: { type: 'white' },
                    envelope: { attack: 0.005, decay: 0.2, sustain: 0, release: 0.5 },
                    volume: -5
                }).toDestination();
                
                // Setup BGM Synth
                synth = new Tone.Synth({
                    oscillator: { type: "square" },
                    envelope: { attack: 0.01, decay: 0.2, sustain: 0.1, release: 0.5 },
                    volume: -15 
                }).toDestination();

            } catch (e) {
                console.error("Tone.js setup failed:", e);
            }
        }

        function loadAndStartMusic() {
            // Stop any existing loop
            if (musicLoop) {
                musicLoop.dispose();
                musicLoop = null;
            }

            const { bgmNotes, bgmTempo } = currentLevel;
            let index = 0;
            
            // Create a new loop for the current level's rhythm
            musicLoop = new Tone.Loop(time => {
                const note = bgmNotes[index % bgmNotes.length];
                if (note) {
                    synth.triggerAttackRelease(note, bgmTempo, time);
                }
                index++;
            }, bgmTempo);
            
            // Ensure audio is running (requires user interaction)
            if (Tone.context.state !== 'running') {
                Tone.start().then(() => {
                    Tone.Transport.start();
                    musicLoop.start(0);
                });
            } else {
                Tone.Transport.start();
                musicLoop.start(0);
            }
        }

        function stopMusic() {
            if (musicLoop) {
                musicLoop.stop();
            }
            // Keep Tone.Transport running to avoid latency on next start
        }

        function playDeathSound() {
            deathSynth.triggerAttackRelease("4n", 0.5);
        }
        
        // --- Classes ---

        class Player {
            constructor() {
                this.x = 50;
                this.y = GROUND_Y - PLAYER_SIZE;
                this.vy = 0; 
                this.onGround = true;
                this.rotation = 0; 
                this.rotationSpeed = 0.15; 
            }

            jump() {
                if (this.onGround) {
                    this.vy = JUMP_VELOCITY;
                    this.onGround = false;
                }
            }

            update() {
                // Apply gravity based on current level settings
                this.vy += currentLevel.gravity; 
                this.y += this.vy;

                // Apply rotation only when in the air
                if (!this.onGround) {
                    this.rotation += this.rotationSpeed;
                } else {
                    this.rotation = Math.round(this.rotation / (Math.PI / 2)) * (Math.PI / 2);
                }

                // Check for ground collision
                if (this.y >= GROUND_Y - PLAYER_SIZE) {
                    this.y = GROUND_Y - PLAYER_SIZE;
                    this.vy = 0;
                    this.onGround = true;
                    this.rotation = 0; 
                }
            }

            draw() {
                ctx.save();
                
                const centerX = this.x + PLAYER_SIZE / 2;
                const centerY = this.y + PLAYER_SIZE / 2;
                
                ctx.translate(centerX, centerY);
                ctx.rotate(this.rotation);
                
                // Draw the cube centered at (0, 0)
                ctx.fillStyle = '#4ADEDE'; 
                ctx.shadowColor = '#4ADEDE';
                ctx.shadowBlur = 15;
                ctx.fillRect(-PLAYER_SIZE / 2, -PLAYER_SIZE / 2, PLAYER_SIZE, PLAYER_SIZE);
                ctx.shadowBlur = 0;

                ctx.restore();
            }
        }

        class Obstacle {
            constructor(x, type) {
                this.x = x;
                this.type = type; 
                this.size = (this.type === 'b') ? 40 : 20;
                this.y = GROUND_Y - this.size;
            }

            update() {
                this.x -= currentLevel.speed;
            }

            draw() {
                ctx.fillStyle = '#FF6EC7'; 
                ctx.shadowColor = '#FF6EC7';
                ctx.shadowBlur = 10;

                if (this.type === 's') {
                    // Draw a triangle (spike)
                    ctx.beginPath();
                    ctx.moveTo(this.x, this.y + this.size); 
                    ctx.lineTo(this.x + this.size / 2, this.y); 
                    ctx.lineTo(this.x + this.size, this.y + this.size); 
                    ctx.closePath();
                    ctx.fill();
                } else {
                    // Draw a block (square)
                    ctx.fillRect(this.x, this.y, this.size, this.size);
                }

                ctx.shadowBlur = 0;
            }
        }

        // --- Game Logic ---

        function initLevel() {
            currentLevel = LEVELS[currentLevelIndex];
            
            player = new Player();
            obstacles = [];
            
            frame = 0;
            obstacleIndex = 0;
            framesUntilNextObstacle = 0;

            levelDisplay.textContent = `Level: ${currentLevel.name}`;
            
            isRunning = true;
            overlay.style.opacity = 0;
            overlay.style.pointerEvents = 'none';

            loadAndStartMusic();
        }

        function initGame(levelIndex = 0) {
            currentLevelIndex = levelIndex;
            score = 0; // Reset score for a new game
            scoreDisplay.textContent = score;
            initLevel();
        }

        function checkCollision() {
            const px = player.x;
            const py = player.y;
            const ps = PLAYER_SIZE;

            for (const obs of obstacles) {
                const ox = obs.x;
                const oy = obs.y;
                const os = obs.size;

                // AABB Collision detection
                if (px < ox + os &&
                    px + ps > ox &&
                    py < oy + os &&
                    py + ps > oy) {
                    return true; // Collision detected
                }
            }
            return false;
        }

        function spawnObstacleFromTimeline() {
            if (obstacleIndex < currentLevel.timeline.length) {
                const nextItem = currentLevel.timeline[obstacleIndex];
                
                if (nextItem.type === 'gap') {
                    // It's a gap, set the counter and move to the next item
                    framesUntilNextObstacle = nextItem.frameGap;
                    obstacleIndex++;
                } else {
                    // It's an obstacle, spawn it, set the gap counter, and move to the next item
                    obstacles.push(new Obstacle(canvas.width, nextItem.type));
                    framesUntilNextObstacle = nextItem.frameGap;
                    obstacleIndex++;
                }
            }
        }

        function handleInput() {
            if (!isRunning) return;
            player.jump();
        }

        function gameLoop() {
            // 1. Update
            if (isRunning) {
                // Clear the canvas
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                // Player Update
                player.update();

                // Obstacle Spawning Logic (Reads from fixed timeline)
                if (framesUntilNextObstacle <= 0) {
                    spawnObstacleFromTimeline();
                } else {
                    framesUntilNextObstacle--;
                }

                // Obstacle Update
                obstacles.forEach(obs => obs.update());

                // Check for level complete (no more obstacles to spawn and all existing obstacles have left the screen)
                if (obstacleIndex >= currentLevel.timeline.length && obstacles.length === 0) {
                    levelComplete();
                    return;
                }
                
                // Remove off-screen obstacles
                obstacles = obstacles.filter(obs => obs.x + obs.size > 0);
                
                // Score update (every 10 frames)
                if (frame % 10 === 0) {
                    score++;
                    scoreDisplay.textContent = score;
                }

                // Collision Check
                if (checkCollision()) {
                    gameOver();
                    return;
                }
                
                frame++;
            }
            
            // 2. Draw
            
            // Draw Ground
            ctx.fillStyle = '#374151'; 
            ctx.fillRect(0, GROUND_Y, canvas.width, canvas.height - GROUND_Y);

            // Draw Obstacles
            obstacles.forEach(obs => obs.draw());

            // Draw Player 
            player.draw();


            animationFrameId = requestAnimationFrame(gameLoop);
        }

        function levelComplete() {
            isRunning = false;
            cancelAnimationFrame(animationFrameId);
            stopMusic();

            currentLevelIndex++;

            if (currentLevelIndex < LEVELS.length) {
                // Transition to next level
                const nextLevel = LEVELS[currentLevelIndex];
                overlayText.innerHTML = `Level ${currentLevelIndex} Cleared!<br><span class="text-teal-400">NEXT: ${nextLevel.name}</span>`;
                startButton.textContent = 'Continue';
                overlay.style.opacity = 1;
                overlay.style.pointerEvents = 'auto';
            } else {
                // Game Finished
                updateHighScore();
                overlayText.innerHTML = `<span class="text-green-500 ui-glow">ALL LEVELS CLEARED!</span><br>Final Score: ${score}<br>High Score: ${highScore}`;
                startButton.textContent = 'Play Again';
                overlay.style.opacity = 1;
                overlay.style.pointerEvents = 'auto';
                currentLevelIndex = 0; // Reset for next game start
            }
        }

        function gameOver() {
            isRunning = false;
            cancelAnimationFrame(animationFrameId);
            
            stopMusic(); 
            playDeathSound(); 

            updateHighScore();

            // Show Game Over Screen
            overlayText.innerHTML = `<span class="text-red-500 ui-glow">GAME OVER!</span><br>Score: ${score}<br>High Score: ${highScore}`;
            startButton.textContent = 'Restart Game';
            overlay.style.opacity = 1;
            overlay.style.pointerEvents = 'auto';
            currentLevelIndex = 0; // Reset to level 1 on failure
        }

        function updateHighScore() {
            if (score > highScore) {
                highScore = score;
                localStorage.setItem('geometricDashHighScore', highScore);
                highScoreDisplay.textContent = highScore;
            }
        }

        // --- Event Listeners ---

        function startGameHandler() {
            if (!isRunning) {
                // Check if we are starting a new game (index 0) or continuing to the next level
                if (currentLevelIndex === 0 && score === 0) {
                    initGame(0); // Start from level 1
                } else {
                    initLevel(); // Continue to the next initialized level
                }
                gameLoop();
            }
        }

        // Mouse/Touch Jump
        canvas.addEventListener('click', handleInput);
        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault(); 
            handleInput();
        });

        // Keyboard Jump
        document.addEventListener('keydown', (e) => {
            if (e.code === 'Space' || e.key === ' ') {
                e.preventDefault();
                handleInput();
            }
        });

        // Start Button
        startButton.addEventListener('click', startGameHandler);
        
        // Handle keyboard event on overlay/start screen
        document.addEventListener('keydown', (e) => {
            if (!isRunning && (e.code === 'Space' || e.key === ' ')) {
                e.preventDefault();
                startGameHandler();
            }
        });

        // Initialize the game state when the window loads
        window.onload = () => {
             setupAudio(); 
             // Show Title Screen
             levelDisplay.textContent = `Geometric Dash Deluxe`;
             overlayText.innerHTML = `Welcome to Geometric Dash Deluxe!`;
             startButton.textContent = 'Start Game';
             overlay.style.opacity = 1;
             overlay.style.pointerEvents = 'auto';
        };

    </script>

</body>
</html>
