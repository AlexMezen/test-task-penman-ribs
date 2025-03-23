<template>
  <div class="container">
    <!-- <div class="trail">
      <div
        v-for="(point, index) in trailPoints"
        :key="index"
        class="trail-dot"
        :style="{
          left: `calc(50% + ${point.x}px)`,
          top: `calc(50% + ${point.y}px)`
        }"
      ></div>
    </div> -->
    <div class="airplane" :style="airplaneStyle">✈️</div>
    <button class="control-button" @click="toggleAnimation">
      {{ isPlaying ? "Stop" : "Start" }}
    </button>
  </div>
</template>

<script>
import flightData from "../src/assets/flightData.json";
export default {
  data() {
    return {
      isPlaying: false,
      progress: 0,
      points: [],
      scale: 1,
      animationId: null,
      currentPosition: { x: 0, y: 0 },
      currentDirection: 0,
      smoothDirection: 0,
      trailPoints: [],
    };
  },
  computed: {
    airplaneStyle() {
      return {
        transform: `translate(
    calc(-50% + ${this.currentPosition.x}px),
    calc(-50% + ${this.currentPosition.y}px)
  ) rotate(${this.smoothDirection + -45}deg)`,
      };
    },
  },
  mounted() {
    this.calculateTrajectory();
  },
  methods: {
    calculateTrajectory() {
      let points = [{ x: 0, y: 0, direction: flightData[0].direction }];
      let currentX = 0;
      let currentY = 0;

      flightData.forEach((segment) => {
        const speed = parseFloat(segment.speed);
        const direction = parseFloat(segment.direction);
        const angle = ((90 - direction) * Math.PI) / 180;
        const hours = 60 / 3600;
        const distance = speed * hours;

        const dx = distance * Math.cos(angle);
        const dy = -distance * Math.sin(angle);

        currentY += -dx;
        currentX += dy;

        const adjustedDirection = (direction - 90) % 360;
        points.push({ x: currentX, y: currentY, direction: adjustedDirection });
      });

      let minX = 0,
        maxX = 0,
        minY = 0,
        maxY = 0;
      points.forEach((point) => {
        minX = Math.min(minX, point.x);
        maxX = Math.max(maxX, point.x);
        minY = Math.min(minY, point.y);
        maxY = Math.max(maxY, point.y);
      });

      const width = maxX - minX;
      const height = maxY - minY;
      const maxDimension = Math.max(width, height);
      const screenSize = Math.min(window.innerWidth, window.innerHeight);
      const scale = (0.45 * screenSize) / maxDimension;

      this.points = points.map((point) => ({
        x: point.x * scale,
        y: point.y * scale,
        direction: point.direction,
      }));
    },

    calculatePosition(progress) {
      const totalSegments = this.points.length - 1;
      if (totalSegments === 0) return { x: 0, y: 0, direction: 0 };

      const totalTime = 20;
      const timePerSegment = totalTime / totalSegments;
      const currentTime = progress * totalTime;
      const segmentIndex = Math.floor(currentTime / timePerSegment);

      if (segmentIndex >= totalSegments) {
        return this.points[this.points.length - 1];
      }

      const segmentProgress = (currentTime % timePerSegment) / timePerSegment;
      const start = this.points[segmentIndex];
      const end = this.points[segmentIndex + 1];

      const dx = end.x - start.x;
      const dy = end.y - start.y;
      let angle = (Math.atan2(dy, dx) * 180) / Math.PI;
      angle = 80 + angle; 

      while (angle < 0) angle += 360;
      while (angle >= 360) angle -= 360;

      return {
        x: start.x + (end.x - start.x) * segmentProgress,
        y: start.y + (end.y - start.y) * segmentProgress,
        direction: angle,
      };
    },

    smoothRotation(targetAngle, currentAngle) {
      let diff = targetAngle - currentAngle;

      while (diff > 180) diff -= 360;
      while (diff < -180) diff += 360;

      const smoothFactor = 0.05;
      return currentAngle + diff * smoothFactor;
    },

    getShortestRotation(from, to) {
      const diff = ((((to - from) % 360) + 540) % 360) - 180;
      return from + diff;
    },

    toggleAnimation() {
      if (this.isPlaying) {
        this.stopAnimation();
      } else {
        this.startAnimation();
      }
    },

    // addTrailDot() {
    //   this.trailPoints.push({...this.currentPosition})

    //   if (this.trailPoints.length > 75) {
    //     this.trailPoints.shift()
    //   }
    // },

    returnToStart() {
      const startTime = Date.now();
      const startPos = { ...this.currentPosition };
      const duration = 1000; 

      const animateReturn = () => {
        const elapsed = Date.now() - startTime;
        const progress = Math.min(elapsed / duration, 1);

        const easeOut = (t) => 1 - Math.pow(1 - t, 2);
        const easedProgress = easeOut(progress);

        this.currentPosition = {
          x: startPos.x * (1 - easedProgress),
          y: startPos.y * (1 - easedProgress),
        };

        this.currentDirection = this.currentDirection * (1 - easedProgress);
        this.smoothDirection = this.smoothDirection * (1 - easedProgress);

        // this.trailPoints = this.trailPoints.map(point => ({
        //   ...point,
        //   opacity: 1 - easedProgress
        // }))

        if (progress < 1) {
          requestAnimationFrame(animateReturn);
        } else {
          // this.trailPoints = []
          this.currentPosition = { x: 0, y: 0 };
          this.currentDirection = 0;
          this.smoothDirection = 0;
        }
      };

      requestAnimationFrame(animateReturn);
    },

    startAnimation() {
      if (this.isPlaying) return;
      this.isPlaying = true;
      const startTime = Date.now();
      // this.trailPoints = []
      this.smoothDirection = 0; 

      // let lastDotTime = 0
      // const dotInterval = 250

      const animate = () => {
        if (!this.isPlaying) return;

        const now = Date.now();
        const elapsed = now - startTime;
        const progress = Math.min(elapsed / 20000, 1);
        this.progress = progress;

        const position = this.calculatePosition(progress);
        this.currentPosition = { x: position.x, y: position.y };
        this.currentDirection = position.direction;

        this.smoothDirection = this.smoothRotation(
          this.currentDirection,
          this.smoothDirection
        );

        // if (now - lastDotTime > dotInterval) {
        //   this.addTrailDot()
        //   lastDotTime = now
        // }

        if (progress < 1) {
          this.animationId = requestAnimationFrame(animate);
        } else {
          this.isPlaying = false;
          this.returnToStart();
        }
      };

      this.animationId = requestAnimationFrame(animate);
    },

    stopAnimation() {
      this.isPlaying = false;
      if (this.animationId) {
        cancelAnimationFrame(this.animationId);
      }
      this.returnToStart();
    },
  },
};
</script>

<style>
.container {
  position: relative;
  height: 100vh;
  overflow: hidden;
  background: url("../src/assets/map.png") no-repeat center center;
  background-size: cover;
}

.airplane {
  position: absolute;
  left: 50%;
  top: 50%;
  font-size: 24px;
  will-change: transform;
  z-index: 990;
  transition: transform 0.3s ease-out; /* Smooth transition for rotation */
  padding: 40px;
}

/* .trail {
  position: absolute;
  width: 100%;
  height: 100%;
} */

/* .trail-dot {
  position: absolute;
  width: 3px;
  height: 3px;
  background-color: #4287f5;
  border-radius: 50%;
  opacity: 0.4; 
} */

.control-button {
  position: absolute;
  left: 50%;
  bottom: 20%;
  transform: translate(-50%, 50px);
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  background-color: #ead854;
  color: black;
  border: none;
  border-radius: 5px;
  z-index: 10;
}

.control-button:hover {
  background-color: #ffe945;
}
</style>
