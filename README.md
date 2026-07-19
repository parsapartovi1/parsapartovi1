<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Chrome Dino Backend Developer Journey</title>
  <style>
    :root {
      --background: #000000;
      --dino-fill: #535353;
      --dino-outline: #f7f7f7;
      --line: #535353;
      --step-fill: #080808;
      --step-line: #747474;
      --step-active: #f1f1f1;
      --text: #d8d8d8;
      --blue-dark: #0b2c5f;
      --blue-light: #174b91;
      --gold-dark: #b88b20;
      --gold-light: #f1cb58;
    }

    * {
      box-sizing: border-box;
    }

    html,
    body {
      width: 100%;
      height: 100%;
      margin: 0;
      overflow: hidden;
      background: var(--background);
    }

    body {
      font-family: "Courier New", Courier, monospace;
    }

    .game-frame {
      position: relative;
      width: 100%;
      height: 100%;
      background: var(--background);
    }

    canvas {
      display: block;
      width: 100%;
      height: 100%;
      background: var(--background);
      image-rendering: pixelated;
      image-rendering: crisp-edges;
    }

    #replay {
      position: absolute;
      left: 10px;
      top: 10px;
      z-index: 2;
      padding: 5px 9px;
      border: 1px solid #454545;
      background: #050505;
      color: #888888;
      font: 700 10px "Courier New", Courier, monospace;
      cursor: pointer;
      opacity: 0.72;
    }

    #replay:hover,
    #replay:focus-visible {
      color: #eeeeee;
      border-color: #eeeeee;
      outline: none;
      opacity: 1;
    }
  </style>
</head>
<body>
  <main class="game-frame">
    <canvas id="game" aria-label="Chrome Dino climbing backend development learning stairs"></canvas>
    <button id="replay" type="button">REPLAY</button>
  </main>

  <script>
    (() => {
      "use strict";

      const canvas = document.getElementById("game");
      const context = canvas.getContext("2d");
      const replayButton = document.getElementById("replay");

      context.imageSmoothingEnabled = false;

      const dinoImage = new Image();
      dinoImage.src = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACwAAAAvCAYAAACYECivAAABB0lEQVR42u2YMQ7DIAxFTdSF+zHmeIzcj7GdGGrFBcemkTFsURKLPP34fxNg0qq1vjXrxRgDAMABxtZrNtnzPEX1cs5f1+YIh1mEpWQp0n8jXEqBUoq4zv7pqJVSUqljjvDe8N7w7hJKhrAs4UCFlhbnng49eB/2NIzDCkdTVyF9lCxXu366hIQqZxRal3Cj1RsipVTx+6OaXkfDVNfQ0mury623XpfQHibdZYnH0xo3s9gj3L5w1gGIeUlQsZG9YSuk1zkM1CaN++1dSax33ColrUXWLOGDQ+qX7/fuu+0SXePA/blnBJTmpdr1l9akFrsJj04iTbMU6dHn/Doddry73cSt030Ar6iLc49s17oAAAAASUVORK5CYII=";

      const milestones = [
        { full: "Self-learning →", lines: ["Self-", "learning →"] },
        { full: "Python →", lines: ["Python →"] },
        { full: "Bootcamp →", lines: ["Bootcamp →"] },
        { full: "Django →", lines: ["Django →"] },
        { full: "DRF →", lines: ["DRF →"] },
        { full: "REST APIs →", lines: ["REST", "APIs →"] },
        {
          full: "Databases (PostgreSQL/SQLite) →",
          lines: ["Databases", "(PostgreSQL", "/ SQLite) →"]
        },
        { full: "Docker & Git →", lines: ["Docker &", "Git →"] },
        { full: "Real Projects →", lines: ["Real", "Projects →"] },
        { full: "Backend Developer", lines: ["Backend", "Developer"] }
      ];

      const state = {
        width: 0,
        height: 0,
        dpr: 1,
        groundY: 0,
        startedAt: 0,
        finished: false,
        score: 0,
        stairs: [],
        path: [],
        particles: [],
        lastFrameTime: 0
      };

      const COLORS = {
        background: "#000000",
        ground: "#535353",
        stepFill: "#080808",
        stepLine: "#747474",
        stepActive: "#f1f1f1",
        text: "#d8d8d8",
        muted: "#535353",
        blueDark: "#0b2c5f",
        blueLight: "#174b91",
        goldDark: "#b88b20",
        goldLight: "#f1cb58"
      };

      function clamp(value, minimum, maximum) {
        return Math.max(minimum, Math.min(maximum, value));
      }

      function resizeCanvas() {
        const bounds = canvas.getBoundingClientRect();
        const dpr = clamp(window.devicePixelRatio || 1, 1, 2);

        canvas.width = Math.round(bounds.width * dpr);
        canvas.height = Math.round(bounds.height * dpr);
        context.setTransform(dpr, 0, 0, dpr, 0, 0);
        context.imageSmoothingEnabled = false;

        state.width = bounds.width;
        state.height = bounds.height;
        state.dpr = dpr;

        buildLayout();
      }

      function buildLayout() {
        const width = state.width;
        const height = state.height;

        state.groundY = Math.floor(height * 0.875);

        const dinoScale = clamp(height / 185, 0.78, 1.48);
        const leftSpace = clamp(64 * dinoScale, 60, width * 0.18);
        const rightMargin = clamp(width * 0.025, 12, 36);
        const availableWidth = Math.max(300, width - leftSpace - rightMargin);
        const stepWidth = availableWidth / milestones.length;

        const topMargin = clamp(height * 0.09, 20, 70);
        const baseHeight = clamp(height * 0.135, 22, 54);
        const maximumRise = clamp(height * 0.038, 5, 21);
        const riseToFit =
          (state.groundY - topMargin - baseHeight) /
          Math.max(1, milestones.length - 1);
        const rise = clamp(
          Math.min(maximumRise, riseToFit),
          5,
          maximumRise
        );

        state.stairs = milestones.map((milestone, index) => {
          const x = leftSpace + index * stepWidth;
          const heightFromGround = baseHeight + index * rise;
          const y = state.groundY - heightFromGround;

          return {
            ...milestone,
            index,
            x,
            y,
            width: stepWidth + 0.5,
            height: heightFromGround
          };
        });

        const dinoWidth = dinoImage.naturalWidth || 44;
        const startX = Math.max(0, 2 - dinoWidth * 0.04);

        state.path = [
          {
            x: startX,
            footY: state.groundY,
            scale: dinoScale
          },
          ...state.stairs.map((step) => ({
            x:
              step.x +
              step.width * 0.5 -
              (dinoWidth * dinoScale) * 0.5,
            footY: step.y,
            scale: dinoScale
          }))
        ];

        state.particles = [];
        state.finished = false;
      }

      function pixelRect(x, y, width, height, color) {
        context.fillStyle = color;
        context.fillRect(
          Math.round(x),
          Math.round(y),
          Math.round(width),
          Math.round(height)
        );
      }

      function horizontalLine(x1, x2, y, color, thickness = 1) {
        pixelRect(x1, y, x2 - x1, thickness, color);
      }

      function verticalLine(x, y1, y2, color, thickness = 1) {
        pixelRect(x, y1, thickness, y2 - y1, color);
      }

      function drawGround() {
        horizontalLine(0, state.width, state.groundY, COLORS.ground, 1);

        for (let x = 7; x < state.width; x += 29) {
          pixelRect(x, state.groundY + 4, 2, 1, COLORS.ground);

          if (Math.floor(x / 29) % 3 === 0) {
            pixelRect(x + 9, state.groundY + 7, 1, 1, COLORS.ground);
          }

          if (Math.floor(x / 29) % 5 === 0) {
            pixelRect(x + 17, state.groundY + 5, 2, 1, COLORS.ground);
          }
        }
      }

      function findFittingFont(lines, maximumWidth, maximumHeight) {
        let fontSize = clamp(maximumWidth * 0.18, 7, 15);

        while (fontSize > 5) {
          context.font = `700 ${fontSize}px "Courier New", Courier, monospace`;

          const widestLine = Math.max(
            ...lines.map((line) => context.measureText(line).width)
          );
          const lineHeight = fontSize * 1.05;
          const totalHeight = lines.length * lineHeight;

          if (
            widestLine <= maximumWidth &&
            totalHeight <= maximumHeight
          ) {
            return {
              fontSize,
              lineHeight
            };
          }

          fontSize -= 0.25;
        }

        return {
          fontSize: 5,
          lineHeight: 5.3
        };
      }

      function drawCenteredLabel(step, color) {
        const horizontalPadding = clamp(step.width * 0.06, 3, 8);
        const verticalPadding = 3;
        const labelWidth = Math.max(10, step.width - horizontalPadding * 2);

        // Labels stay in the upper part of each stair face.
        const labelHeight = Math.max(
          12,
          Math.min(step.height - verticalPadding * 2, 49)
        );

        const fitted = findFittingFont(
          step.lines,
          labelWidth,
          labelHeight
        );

        context.font =
          `700 ${fitted.fontSize}px "Courier New", Courier, monospace`;
        context.textAlign = "center";
        context.textBaseline = "middle";
        context.fillStyle = color;

        const totalTextHeight = fitted.lineHeight * step.lines.length;
        const centerY =
          step.y +
          verticalPadding +
          labelHeight * 0.5;
        const firstLineY =
          centerY -
          totalTextHeight * 0.5 +
          fitted.lineHeight * 0.5;

        step.lines.forEach((line, lineIndex) => {
          context.fillText(
            line,
            step.x + step.width * 0.5,
            firstLineY + lineIndex * fitted.lineHeight,
            labelWidth
          );
        });
      }

      function drawStairs(activeIndex) {
        state.stairs.forEach((step, index) => {
          const active = index === activeIndex;
          const outline = active ? COLORS.stepActive : COLORS.stepLine;
          const thickness = active ? 2 : 1;

          pixelRect(
            step.x,
            step.y,
            step.width,
            step.height,
            COLORS.stepFill
          );

          horizontalLine(
            step.x,
            step.x + step.width,
            step.y,
            outline,
            thickness
          );
          verticalLine(
            step.x,
            step.y,
            state.groundY,
            outline,
            thickness
          );
          verticalLine(
            step.x + step.width - thickness,
            step.y,
            state.groundY,
            outline,
            thickness
          );

          // Small Chrome-Dino-style ground details on each tread.
          const detailColor = active ? "#bdbdbd" : "#4e4e4e";
          for (
            let detailX = step.x + 7;
            detailX < step.x + step.width - 6;
            detailX += 15
          ) {
            pixelRect(detailX, step.y + 4, 2, 1, detailColor);
          }

          drawCenteredLabel(
            step,
            active ? COLORS.stepActive : COLORS.text
          );
        });
      }


      function drawProfileTitle() {
        const titleX = clamp(state.width * 0.29, 115, state.width * 0.42);
        const titleY = clamp(state.height * 0.16, 34, 105);

        const titleSize = clamp(
          Math.min(state.width * 0.026, state.height * 0.065),
          15,
          30
        );
        const subtitleSize = clamp(titleSize * 0.43, 8, 13);

        context.save();
        context.textAlign = "center";
        context.textBaseline = "middle";

        // Pixel-style main name, matching the Dino score typography.
        context.font =
          `700 ${titleSize}px "Courier New", Courier, monospace`;
        context.fillStyle = "#d8d8d8";
        context.fillText("Parsa Partovi", titleX, titleY);

        // Small role line under the name.
        context.font =
          `700 ${subtitleSize}px "Courier New", Courier, monospace`;
        context.fillStyle = "#777777";
        context.fillText(
          "django-python back-end developer",
          titleX,
          titleY + titleSize * 0.82
        );

        context.restore();
      }

      function drawScore() {
        context.fillStyle = COLORS.muted;
        context.textAlign = "right";
        context.textBaseline = "top";
        context.font =
          `700 ${clamp(state.height * 0.075, 11, 18)}px "Courier New", Courier, monospace`;

        context.fillText(
          String(state.score).padStart(5, "0"),
          state.width - 12,
          8
        );
      }

      function drawDino(position, elapsed, standing) {
        if (!dinoImage.complete || !dinoImage.naturalWidth) {
          return;
        }

        const width = dinoImage.naturalWidth * position.scale;
        const height = dinoImage.naturalHeight * position.scale;

        const runningBob = standing
          ? 0
          : Math.floor(elapsed * 12) % 2 === 0
            ? 0
            : 1.5 * position.scale;

        context.save();
        context.imageSmoothingEnabled = false;
        context.drawImage(
          dinoImage,
          Math.round(position.x),
          Math.round(position.footY - height + runningBob),
          Math.round(width),
          Math.round(height)
        );
        context.restore();
      }

      function createConfetti(originX, originY) {
        const colors = [
          COLORS.blueDark,
          COLORS.blueLight,
          COLORS.goldDark,
          COLORS.goldLight
        ];

        const count = clamp(
          Math.round(state.width / 5),
          90,
          220
        );

        state.particles = Array.from({ length: count }, () => {
          const angle =
            -Math.PI / 2 +
            (Math.random() - 0.5) * 2.2;
          const speed = 85 + Math.random() * 250;

          return {
            x: originX,
            y: originY,
            velocityX:
              Math.cos(angle) * speed +
              (Math.random() - 0.5) * 90,
            velocityY: Math.sin(angle) * speed,
            gravity: 180 + Math.random() * 120,
            age: 0,
            life: 1.5 + Math.random() * 1.35,
            width: 3 + Math.random() * 6,
            height: 2 + Math.random() * 4,
            rotation: Math.random() * Math.PI * 2,
            spin: (Math.random() - 0.5) * 15,
            color:
              colors[Math.floor(Math.random() * colors.length)]
          };
        });
      }

      function updateConfetti(deltaTime) {
        state.particles.forEach((particle) => {
          particle.age += deltaTime;
          particle.x += particle.velocityX * deltaTime;
          particle.y += particle.velocityY * deltaTime;
          particle.velocityY += particle.gravity * deltaTime;
          particle.rotation += particle.spin * deltaTime;
        });

        state.particles = state.particles.filter(
          (particle) =>
            particle.age < particle.life &&
            particle.y < state.height + 30
        );
      }

      function drawConfetti() {
        state.particles.forEach((particle) => {
          const opacity = Math.max(
            0,
            1 - particle.age / particle.life
          );

          context.save();
          context.globalAlpha = opacity;
          context.translate(particle.x, particle.y);
          context.rotate(particle.rotation);
          context.fillStyle = particle.color;
          context.fillRect(
            -particle.width * 0.5,
            -particle.height * 0.5,
            particle.width,
            particle.height
          );
          context.restore();
        });

        context.globalAlpha = 1;
      }

      function currentPosition(elapsed) {
        const initialPause = 0.45;
        const jumpDuration = 0.83;
        const landingPause = 0.12;

        if (elapsed < initialPause) {
          return {
            ...state.path[0],
            activeIndex: -1,
            standing: true,
            done: false
          };
        }

        let remaining = elapsed - initialPause;

        for (
          let index = 0;
          index < state.path.length - 1;
          index += 1
        ) {
          if (remaining <= jumpDuration) {
            const start = state.path[index];
            const end = state.path[index + 1];
            const progress = clamp(remaining / jumpDuration, 0, 1);
            const easedProgress =
              1 - Math.pow(1 - progress, 3);
            const jumpHeight = clamp(
              state.height * 0.12,
              18,
              92
            );

            return {
              x:
                start.x +
                (end.x - start.x) * easedProgress,
              footY:
                start.footY +
                (end.footY - start.footY) * easedProgress -
                Math.sin(Math.PI * progress) * jumpHeight,
              scale: end.scale,
              activeIndex: index,
              standing: false,
              done: false
            };
          }

          remaining -= jumpDuration;

          if (remaining <= landingPause) {
            return {
              ...state.path[index + 1],
              activeIndex: index,
              standing: true,
              done: false
            };
          }

          remaining -= landingPause;
        }

        return {
          ...state.path[state.path.length - 1],
          activeIndex: state.stairs.length - 1,
          standing: true,
          done: true
        };
      }

      function render(timestamp) {
        if (!state.startedAt) {
          state.startedAt = timestamp;
        }

        const elapsed = (timestamp - state.startedAt) / 1000;
        const deltaTime = state.lastFrameTime
          ? Math.min((timestamp - state.lastFrameTime) / 1000, 0.04)
          : 0;
        state.lastFrameTime = timestamp;

        const position = currentPosition(elapsed);

        if (position.done && !state.finished) {
          state.finished = true;
          const finalStep = state.stairs[state.stairs.length - 1];
          createConfetti(
            finalStep.x + finalStep.width * 0.55,
            finalStep.y - 6
          );
        }

        state.score = Math.min(
          99999,
          Math.floor(elapsed * 7) + (state.finished ? 33 : 0)
        );

        context.clearRect(0, 0, state.width, state.height);
        pixelRect(
          0,
          0,
          state.width,
          state.height,
          COLORS.background
        );

        drawProfileTitle();
        drawScore();
        drawStairs(position.activeIndex);
        drawGround();
        drawDino(position, elapsed, position.standing);

        updateConfetti(deltaTime);
        drawConfetti();

        requestAnimationFrame(render);
      }

      function restart() {
        buildLayout();
        state.startedAt = performance.now();
        state.lastFrameTime = 0;
        state.finished = false;
        state.particles = [];
      }

      replayButton.addEventListener("click", restart);

      let resizeTimer = 0;
      window.addEventListener("resize", () => {
        window.clearTimeout(resizeTimer);
        resizeTimer = window.setTimeout(() => {
          resizeCanvas();
          restart();
        }, 100);
      });

      dinoImage.addEventListener("load", () => {
        resizeCanvas();
        restart();
      });

      resizeCanvas();
      requestAnimationFrame(render);
    })();
  </script>
</body>
</html>



## 🌐 Socials:
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%230077B5.svg?logo=linkedin&logoColor=white)](https://linkedin.com/in/parsa-partovi1) [![email](https://img.shields.io/badge/Email-D14836?logo=gmail&logoColor=white)](mailto:pparsapartovi13@gmail.com) 

# 💻 Tech Stack:
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi) ![Django](https://img.shields.io/badge/django-%23092E20.svg?style=for-the-badge&logo=django&logoColor=white) ![DjangoREST](https://img.shields.io/badge/DJANGO-REST-ff1709?style=for-the-badge&logo=django&logoColor=white&color=ff1709&labelColor=gray) ![Flask](https://img.shields.io/badge/flask-%23000.svg?style=for-the-badge&logo=flask&logoColor=white) ![Celery](https://img.shields.io/badge/celery-%23a9cc54.svg?style=for-the-badge&logo=celery&logoColor=ddf4a4) ![Postgres](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white) ![MongoDB](https://img.shields.io/badge/MongoDB-%234ea94b.svg?style=for-the-badge&logo=mongodb&logoColor=white) ![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white) ![SQLite](https://img.shields.io/badge/sqlite-%2307405e.svg?style=for-the-badge&logo=sqlite&logoColor=white) ![GIT](https://img.shields.io/badge/Git-fc6d26?style=for-the-badge&logo=git&logoColor=white) ![Jasmine](https://img.shields.io/badge/-Jasmine-%238A4182?style=for-the-badge&logo=Jasmine&logoColor=white) ![GitHub](https://img.shields.io/badge/GitHub-%23121011.svg?style=for-the-badge&logo=github&logoColor=white) ![GitLab](https://img.shields.io/badge/gitlab-%23181717.svg?style=for-the-badge&logo=gitlab&logoColor=white) ![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white) ![GitLab CI](https://img.shields.io/badge/gitlab%20CI-%23181717.svg?style=for-the-badge&logo=gitlab&logoColor=white) ![LINUX](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black) ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white) ![JWT](https://img.shields.io/badge/JWT-black?style=for-the-badge&logo=JSON%20web%20tokens)![Swagger](https://img.shields.io/badge/-Swagger-%23Clojure?style=for-the-badge&logo=swagger&logoColor=white) ![Postman](https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white) ![Insomnia](https://img.shields.io/badge/Insomnia-black?style=for-the-badge&logo=insomnia&logoColor=5849BE) ![Bruno](https://img.shields.io/badge/bruno-%23F4AA41.svg?style=for-the-badge&logo=bruno&logoColor=white) ![YAML](https://img.shields.io/badge/yaml-%23ffffff.svg?style=for-the-badge&logo=yaml&logoColor=151515) ![HTML5](https://img.shields.io/badge/html5-%23E34F26.svg?style=for-the-badge&logo=html5&logoColor=white) ![CSS3](https://img.shields.io/badge/css3-%231572B6.svg?style=for-the-badge&logo=css3&logoColor=white) ![Bootstrap](https://img.shields.io/badge/bootstrap-%23563D7C.svg?style=for-the-badge&logo=bootstrap&logoColor=white) ![jQuery](https://img.shields.io/badge/jquery-%230769AD.svg?style=for-the-badge&logo=jquery&logoColor=white)

