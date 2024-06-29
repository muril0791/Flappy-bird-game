<template>
    <div class="game-container">
        <div v-if="!gameRunning" class="start-screen">
            <button @click="startGame">Start Game</button>
        </div>
        <div v-if="gameRunning" class="score">{{ score }}</div>
        <div ref="gameContainer" class="canvas-container"></div>
    </div>
</template>

<script>
import * as PIXI from 'pixi.js';

export default {
    name: 'Game',
    data() {
        return {
            app: null,
            bird: null,
            pipes: [],
            gravity: 0.33, // Modifique aqui para ajustar a gravidade que puxa o pássaro
            jumpPower: -10,
            birdVelocity: 0,
            pipeGap: 200,
            pipeSpeed: 4,
            pipeInterval: 2000,
            gameRunning: false,
            score: 0,
            pipeSpawner: null,
            backgroundDay: null,
            backgroundNight: null,
            base: [],
            dayCycle: 30000, // 30 seconds for day/night cycle
            birdFrames: [],
            currentBirdFrame: 0,
        };
    },
    mounted() {
        this.setup();
        window.addEventListener('resize', this.resize);
    },
    beforeDestroy() {
        window.removeEventListener('resize', this.resize);
    },
    methods: {
        setup() {
            // Create PIXI Application
            this.app = new PIXI.Application({
                width: 375,
                height: 667,
                backgroundColor: 0x87ceeb, // sky blue background
                resolution: window.devicePixelRatio || 1,
                autoDensity: true,
            });
            this.$refs.gameContainer.appendChild(this.app.view);

            // Load assets
            PIXI.Loader.shared
                .add('bird-downflap', '/src/assets/yellowbird-downflap.png')
                .add('bird-midflap', '/src/assets/yellowbird-midflap.png')
                .add('bird-upflap', '/src/assets/yellowbird-upflap.png')
                .add('pipe', '/src/assets/pipe-green.png')
                .add('background-day', '/src/assets/background-day.png')
                .add('background-night', '/src/assets/background-night.png')
                .add('base', '/src/assets/base.png')
                .load(this.onAssetsLoaded);

            // Resize the app when the window is resized
            this.resize();
        },
        onAssetsLoaded() {
            // Create background sprites
            this.backgroundDay = new PIXI.Sprite(PIXI.Loader.shared.resources['background-day'].texture);
            this.backgroundNight = new PIXI.Sprite(PIXI.Loader.shared.resources['background-night'].texture);

            this.backgroundDay.width = this.app.screen.width;
            this.backgroundDay.height = this.app.screen.height;
            this.backgroundNight.width = this.app.screen.width;
            this.backgroundNight.height = this.app.screen.height;

            this.app.stage.addChild(this.backgroundDay);

            // Base frames for infinite scrolling
            for (let i = 0; i < 2; i++) {
                const base = new PIXI.Sprite(PIXI.Loader.shared.resources['base'].texture);
                base.width = this.app.screen.width;
                base.height = 112; // Height of the base
                base.y = this.app.screen.height - base.height;
                base.x = i * this.app.screen.width;
                this.base.push(base);
                this.app.stage.addChild(base);
            }

            // Bird frames for animation
            this.birdFrames = [
                PIXI.Loader.shared.resources['bird-downflap'].texture,
                PIXI.Loader.shared.resources['bird-midflap'].texture,
                PIXI.Loader.shared.resources['bird-upflap'].texture,
            ];

            // Create bird sprite
            this.bird = new PIXI.Sprite(this.birdFrames[this.currentBirdFrame]);
            this.bird.anchor.set(0.5);
            this.bird.width = 34; // Set bird width similar to original
            this.bird.height = 24; // Set bird height similar to original
            this.bird.x = this.app.screen.width / 4;
            this.bird.y = this.app.screen.height / 2;
            this.app.stage.addChild(this.bird);

            // Add event listeners
            window.addEventListener('keydown', this.onKeyDown);
            this.app.ticker.add(this.gameLoop);

            // Day/night cycle
            setInterval(this.toggleDayNight, this.dayCycle);
        },
        startGame() {
            this.gameRunning = true;
            this.score = 0;
            this.bird.y = this.app.screen.height / 2;
            this.birdVelocity = 0;
            this.pipes.forEach(pipe => this.app.stage.removeChild(pipe));
            this.pipes = [];
            this.pipeSpawner = setInterval(this.spawnPipe, this.pipeInterval);
        },
        onKeyDown(e) {
            if (e.code === 'Space' && this.gameRunning) {
                this.birdVelocity = this.jumpPower;
            }
        },
        gameLoop(delta) {
            if (!this.gameRunning) return;

            // Update bird position
            this.birdVelocity += this.gravity;
            this.bird.y += this.birdVelocity;

            // Bird animation
            this.currentBirdFrame = (this.currentBirdFrame + 1) % this.birdFrames.length;
            this.bird.texture = this.birdFrames[this.currentBirdFrame];

            // Check for collisions
            if (this.bird.y > this.app.screen.height - this.base[0].height || this.bird.y < 0) {
                this.endGame();
            }

            this.pipes.forEach(pipe => {
                pipe.x -= this.pipeSpeed * delta;

                // Check collision with pipes
                if (this.checkCollision(this.bird, pipe)) {
                    this.endGame();
                }

                // Increase score
                if (pipe.x + pipe.width < this.bird.x && !pipe.scored) {
                    this.score++;
                    pipe.scored = true;
                }
            });

            // Remove off-screen pipes
            this.pipes = this.pipes.filter(pipe => pipe.x + pipe.width > 0);

            // Scroll base
            this.base.forEach(base => {
                base.x -= this.pipeSpeed * delta;
                if (base.x <= -this.app.screen.width) {
                    base.x = this.app.screen.width;
                }
            });
        },
        spawnPipe() {
            const pipeHeight = 320;
            const gapStart = Math.random() * (this.app.screen.height - this.pipeGap - this.base[0].height - 100) + 50;

            const pipeTop = new PIXI.Sprite(PIXI.Loader.shared.resources['pipe'].texture);
            pipeTop.anchor.set(0.5, 1);
            pipeTop.width = 52; // Set pipe width similar to original
            pipeTop.height = pipeHeight; // Set pipe height similar to original
            pipeTop.x = this.app.screen.width;
            pipeTop.y = gapStart;

            const pipeBottom = new PIXI.Sprite(PIXI.Loader.shared.resources['pipe'].texture);
            pipeBottom.anchor.set(0.5, 0);
            pipeBottom.width = 52; // Set pipe width similar to original
            pipeBottom.height = pipeHeight; // Set pipe height similar to original
            pipeBottom.x = this.app.screen.width;
            pipeBottom.y = gapStart + this.pipeGap;

            pipeTop.scored = false;
            pipeBottom.scored = false;

            // Add pipes behind the base
            this.app.stage.addChildAt(pipeTop, this.app.stage.getChildIndex(this.base[0]));
            this.app.stage.addChildAt(pipeBottom, this.app.stage.getChildIndex(this.base[0]));

            this.pipes.push(pipeTop);
            this.pipes.push(pipeBottom);
        },
        checkCollision(bird, pipe) {
            const birdBounds = bird.getBounds();
            const pipeBounds = pipe.getBounds();
            return (
                birdBounds.x < pipeBounds.x + pipeBounds.width &&
                birdBounds.x + birdBounds.width > pipeBounds.x &&
                birdBounds.y < pipeBounds.y + pipeBounds.height &&
                birdBounds.y + birdBounds.height > pipeBounds.y
            );
        },
        endGame() {
            this.gameRunning = false;
            clearInterval(this.pipeSpawner);
            alert(`Game Over! Your score: ${this.score}`);
        },
        toggleDayNight() {
            if (this.app.stage.children.includes(this.backgroundDay)) {
                this.app.stage.removeChild(this.backgroundDay);
                this.app.stage.addChildAt(this.backgroundNight, 0);
            } else {
                this.app.stage.removeChild(this.backgroundNight);
                this.app.stage.addChildAt(this.backgroundDay, 0);
            }
        },
        resize() {
            this.app.renderer.resize(375, 667);
            if (this.backgroundDay) {
                this.backgroundDay.width = this.app.screen.width;
                this.backgroundDay.height = this.app.screen.height;
            }
            if (this.backgroundNight) {
                this.backgroundNight.width = this.app.screen.width;
                this.backgroundNight.height = this.app.screen.height;
            }
            this.base.forEach(base => {
                base.width = this.app.screen.width;
                base.y = this.app.screen.height - base.height;
            });
            if (this.bird) {
                this.bird.x = this.app.screen.width / 4;
                this.bird.y = this.app.screen.height / 2;
            }
        },
    },
};
</script>

<style scoped>
.game-container {
    width: 375px;
    height: 667px;
    overflow: hidden;
    margin: 0;
    padding: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
}

.start-screen {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 10;
}

.start-screen button {
    font-size: 2rem;
    padding: 10px 20px;
}

.score {
    position: absolute;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 2rem;
    color: white;
    z-index: 10;
}
</style>
