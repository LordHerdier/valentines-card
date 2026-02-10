<script lang="ts">
  import { onMount } from "svelte";

  let opened = false;
  let showNote = false;
  let showFlashback = false;
  
  function toggleEnvelope() {
    if (!opened) {
      opened = true;
      // Delay showing the note slightly for a better animation sequence
      setTimeout(() => {
        showNote = true;
        // Show flashback after 3 seconds
        setTimeout(() => {
          showFlashback = true;
        }, 3000);
      }, 300);
    } else {
      // Reverse the sequence when closing
      showFlashback = false;
      showNote = false;
      setTimeout(() => {
        opened = false;
      }, 400);
    }
  }

  // Canvas refs
  let canvasEl: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null = null;

  // Animation state
  let raf = 0;
  let lastT = 0;

  // Sizing
  let w = 0;
  let h = 0;
  let dpr = 1;

  // Performance knobs
  const BASE_HEARTS = 80;
  const MOBILE_MAX_DPR = 2;
  let heartCount = BASE_HEARTS;

  // Reduced motion handling
  let reduceMotion = false;
  let reduceMotionMQ: MediaQueryList | null = null;

  type Heart = {
    x: number;
    y: number;
    size: number;
    speed: number;
    drift: number;
    driftSpeed: number;
    phase: number;
    rot: number;
    rotSpeed: number;
    alpha: number;
    color: { r: number; g: number; b: number };
  };

  let hearts: Heart[] = [];

  const HEART_COLORS: Array<{ r: number; g: number; b: number }> = [
    { r: 255, g: 255, b: 255 },
    { r: 255, g: 182, b: 220 },
    { r: 210, g: 190, b: 255 },
    { r: 255, g: 230, b: 150 },
    { r: 255, g: 92, b: 108 },
  ];

  function rand(min: number, max: number) {
    return Math.random() * (max - min) + min;
  }

  function buildHeart(spawnAbove = true): Heart {
    const size = rand(14, 34);
    const startY = spawnAbove ? rand(-h, 0) : rand(0, h);
    const baseSpeed = rand(30, 85);
    return {
      x: rand(0, w),
      y: startY,
      size,
      speed: reduceMotion ? baseSpeed * 0.45 : baseSpeed,
      drift: rand(8, 55),
      driftSpeed: rand(0.7, 1.8),
      phase: rand(0, Math.PI * 2),
      rot: rand(-0.4, 0.4),
      rotSpeed: rand(-0.25, 0.25),
      alpha: rand(0.22, 0.65),
      color: HEART_COLORS[Math.floor(rand(0, HEART_COLORS.length))],
    };
  }

  function initHearts() {
    hearts = Array.from({ length: heartCount }, () => buildHeart(true));
  }

  function respawn(i: number) {
    hearts[i] = buildHeart(true);
    hearts[i].y = rand(-80, -10);
  }

  function resizeCanvas() {
    if (!canvasEl) return;

    const rect = canvasEl.getBoundingClientRect();
    w = Math.max(1, Math.floor(rect.width));
    h = Math.max(1, Math.floor(rect.height));

    dpr = Math.min(window.devicePixelRatio || 1, MOBILE_MAX_DPR);

    canvasEl.width = Math.floor(w * dpr);
    canvasEl.height = Math.floor(h * dpr);

    ctx = canvasEl.getContext("2d");
    if (!ctx) return;

    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }

  function setPerfProfile() {
    const isSmall = window.matchMedia("(max-width: 600px)").matches;
    const isCoarse = window.matchMedia("(pointer: coarse)").matches;

    const target = BASE_HEARTS;
    if (reduceMotion) heartCount = Math.max(40, Math.floor(target * 0.5));
    else if (isSmall || isCoarse) heartCount = target;
    else heartCount = target;

    initHearts();
  }

  function drawHeartPath(c: CanvasRenderingContext2D, size: number) {
    const s = size;
    c.beginPath();
    c.moveTo(0, -0.35 * s);
    c.bezierCurveTo(0.5 * s, -0.85 * s, 1.05 * s, -0.15 * s, 0, 0.55 * s);
    c.bezierCurveTo(-1.05 * s, -0.15 * s, -0.5 * s, -0.85 * s, 0, -0.35 * s);
    c.closePath();
  }

  function tick(t: number) {
    raf = requestAnimationFrame(tick);
    if (!ctx) return;

    const dt = Math.min(0.05, (t - lastT) / 1000 || 0);
    lastT = t;

    ctx.clearRect(0, 0, w, h);

    ctx.save();
    ctx.globalCompositeOperation = "source-over";

    for (let i = 0; i < hearts.length; i++) {
      const he = hearts[i];

      he.y += he.speed * dt;
      he.phase += he.driftSpeed * dt;
      he.rot += he.rotSpeed * dt;

      const x = he.x + Math.sin(he.phase) * he.drift;
      const y = he.y;

      if (y - he.size > h + 40) {
        respawn(i);
        continue;
      }

      ctx.save();
      ctx.translate(x, y);
      ctx.rotate(he.rot);

      ctx.shadowColor = "rgba(0,0,0,0.22)";
      ctx.shadowBlur = reduceMotion ? 6 : 10;
      ctx.shadowOffsetY = 6;

      ctx.fillStyle = `rgba(${he.color.r},${he.color.g},${he.color.b},${he.alpha})`;

      drawHeartPath(ctx, he.size);
      ctx.fill();

      if (!reduceMotion) {
        ctx.shadowColor = "transparent";
        ctx.globalAlpha = he.alpha * 0.35;
        ctx.scale(0.92, 0.92);
        ctx.fillStyle = "rgba(255,255,255,0.55)";
        drawHeartPath(ctx, he.size);
        ctx.fill();
      }

      ctx.restore();
    }

    ctx.restore();
  }

  onMount(() => {
    reduceMotionMQ = window.matchMedia("(prefers-reduced-motion: reduce)");
    reduceMotion = reduceMotionMQ.matches;

    const onRMChange = () => {
      reduceMotion = !!reduceMotionMQ?.matches;
      setPerfProfile();
    };

    if (reduceMotionMQ.addEventListener) reduceMotionMQ.addEventListener("change", onRMChange);
    else reduceMotionMQ.addListener(onRMChange);

    resizeCanvas();
    setPerfProfile();

    const onResize = () => {
      resizeCanvas();
      for (const he of hearts) he.x = Math.min(Math.max(0, he.x), w);
    };

    window.addEventListener("resize", onResize);

    lastT = performance.now();
    raf = requestAnimationFrame(tick);

    return () => {
      cancelAnimationFrame(raf);
      window.removeEventListener("resize", onResize);

      if (reduceMotionMQ) {
        if (reduceMotionMQ.removeEventListener) reduceMotionMQ.removeEventListener("change", onRMChange);
        else reduceMotionMQ.removeListener(onRMChange);
      }
    };
  });
</script>

<svelte:head>
  <title>for my fayrene💌</title>
  <meta name="description" content="a tiny virtual valentines card" />
  <link
    href="https://fonts.googleapis.com/css2?family=Fraunces:wght@600;700&family=Space+Grotesk:wght@400;600&family=Caveat:wght@400;600;700&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<main class="wrap">
  <div class="bg-shade" aria-hidden="true"></div>

  <!-- Canvas hearts layer UNDER the envelope -->
  <canvas class="hearts-canvas" bind:this={canvasEl} aria-hidden="true"></canvas>

  <!-- Note that appears above envelope -->
  {#if showNote}
    <div class="note">
      <div class="note-content">
        <p class="handwritten">My dearest Fayrene,</p>
        <p class="handwritten">
          Every moment with you feels like a dream. Your smile lights up my world
          and your laughter is my favorite melody.
        </p>
        <p class="handwritten">
          Thank you for being you – for your kindness, your warmth, and the way
          you make every day brighter just by being in it.
        </p>
        <p class="handwritten signature">
          Forever yours,<br/>
          ♥
        </p>
      </div>
    </div>
  {/if}

  <!-- Flashback section with polaroids -->
  {#if showFlashback}
    <div class="flashback-section">
      <h2 class="flashback-title">Our Story ♥</h2>
      
      <div class="polaroids">
        <div class="polaroid polaroid-1">
          <div class="polaroid-photo">
            <div class="photo-placeholder">
              <span class="photo-icon">📸</span>
              <span class="photo-text">First Meeting</span>
            </div>
          </div>
          <p class="polaroid-caption">our first meeting ♡</p>
          <div class="polaroid-note">
            <p class="note-text">
              The party where it all began...
              I couldn't stop smiling!
            </p>
          </div>
        </div>

        <div class="polaroid polaroid-2">
          <div class="polaroid-photo">
            <div class="photo-placeholder">
              <span class="photo-icon">☕</span>
              <span class="photo-text">First Date</span>
            </div>
          </div>
          <p class="polaroid-caption">coffee & conversations ♡</p>
          <div class="polaroid-note">
            <p class="note-text">
              Hours flew by like minutes.
              I knew you were special.
            </p>
          </div>
        </div>

        <div class="polaroid polaroid-3">
          <div class="polaroid-photo">
            <div class="photo-placeholder">
              <span class="photo-icon">🌟</span>
              <span class="photo-text">Today</span>
            </div>
          </div>
          <p class="polaroid-caption">still falling for you ♡</p>
          <div class="polaroid-note">
            <p class="note-text">
              Every day with you is
              my new favorite memory.
            </p>
          </div>
        </div>
      </div>
    </div>
  {/if}

  <section class="envelope" class:opened={opened} class:fallen={showNote}>
    <button class="envelope-btn" type="button" on:click={toggleEnvelope} aria-pressed={opened}>
      <img class="img sealed" src="/envelope_sealed.png" alt="Sealed envelope" />
      <img class="img unsealed" src="/envelope_unsealed.png" alt="Unsealed envelope" />
      <span class="tap">{opened ? "click to reseal" : "tap to open"}</span>
    </button>
  </section>
</main>

<style>
  :global(html, body) {
    height: 100%;
    width: 100%;
    overflow: hidden;
  }

  :global(body) {
    margin: 0;
    font-family: "Space Grotesk", "Segoe UI", sans-serif;
    color: rgba(255, 255, 255, 0.92);
    background:
      radial-gradient(circle at 20% 20%, rgba(255, 182, 220, 0.6), transparent 50%),
      radial-gradient(circle at 80% 80%, rgba(255, 140, 200, 0.5), transparent 50%),
      radial-gradient(circle at 50% 50%, rgba(255, 160, 210, 0.4), transparent 60%),
      linear-gradient(135deg, #FFB3D9 0%, #FF8CC5 25%, #FF6FB4 50%, #FF52A3 75%, #FF3B92 100%);
  }

  .wrap {
    min-height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    padding: 16px;
    position: relative;
    overflow-x: hidden;
    overflow-y: auto;
  }

  .bg-shade {
    position: absolute;
    inset: -40px;
    pointer-events: none;
    background:
      radial-gradient(600px 420px at 18% 72%, rgba(255, 255, 255, 0.18), transparent 62%),
      radial-gradient(520px 380px at 82% 68%, rgba(255, 255, 255, 0.12), transparent 66%),
      radial-gradient(900px 700px at 50% 110%, rgba(0, 0, 0, 0.18), transparent 60%);
    mix-blend-mode: soft-light;
    opacity: 0.9;
  }

  .hearts-canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
    pointer-events: none;
  }

  .note {
    width: min(420px, 85vw);
    margin-top: 15vh;
    margin-bottom: 60px;
    z-index: 10;
    opacity: 0;
    transform: translateY(50px) rotate(2deg);
    animation: notePopUp 1.4s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
    animation-delay: 0.2s;
  }

  @keyframes notePopUp {
    0% {
      transform: translateY(50px) rotate(2deg);
      opacity: 0;
    }
    100% {
      transform: translateY(0) rotate(-1deg);
      opacity: 1;
    }
  }

  .note-content {
    background: 
      linear-gradient(to bottom, 
        #fdfbf7 0%, 
        #faf8f3 50%, 
        #f8f6f1 100%
      );
    padding: 40px 35px 35px;
    border-radius: 3px;
    box-shadow: 
      0 1px 3px rgba(0, 0, 0, 0.12),
      0 8px 24px rgba(0, 0, 0, 0.15),
      0 16px 48px rgba(0, 0, 0, 0.1),
      inset 0 0 0 1px rgba(139, 69, 19, 0.08);
    position: relative;
  }

  /* Paper texture overlay */
  .note-content::before {
    content: '';
    position: absolute;
    inset: 0;
    background-image: 
      repeating-linear-gradient(
        0deg,
        transparent,
        transparent 30px,
        rgba(139, 69, 19, 0.03) 30px,
        rgba(139, 69, 19, 0.03) 31px
      );
    pointer-events: none;
    border-radius: 3px;
  }

  /* Subtle paper grain */
  .note-content::after {
    content: '';
    position: absolute;
    inset: 0;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 400 400' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)'/%3E%3C/svg%3E");
    opacity: 0.025;
    pointer-events: none;
    border-radius: 3px;
  }

  .handwritten {
    font-family: 'Caveat', cursive;
    font-size: 24px;
    line-height: 1.6;
    color: #2c1810;
    margin: 0 0 20px 0;
    font-weight: 600;
    position: relative;
    z-index: 1;
  }

  .handwritten:first-child {
    font-size: 26px;
    font-weight: 700;
    margin-bottom: 24px;
  }

  .signature {
    margin-top: 32px;
    margin-bottom: 0;
    font-size: 28px;
    font-weight: 700;
    text-align: right;
  }

  .envelope {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: min(400px, 72vw);
    display: grid;
    gap: 18px;
    place-items: center;
    z-index: 1;
    transition: all 1s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .envelope.fallen {
    transform: translate(-50%, 150vh) rotate(12deg);
    opacity: 0;
  }

  .envelope-btn {
    position: relative;
    width: min(400px, 72vw);
    aspect-ratio: 4 / 3;
    border: none;
    background: transparent;
    cursor: pointer;
    padding: 0;
    filter: drop-shadow(0 24px 50px rgba(0, 0, 0, 0.35));
    transition: filter 0.3s ease;
  }

  .envelope.fallen .envelope-btn {
    pointer-events: none;
  }

  .img {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: contain;
    transition: opacity 400ms ease, transform 400ms ease;
  }

  .sealed {
    opacity: 1;
    transform: translateY(0) scale(1);
  }

  .unsealed {
    opacity: 0;
    transform: translateY(8px) scale(0.98);
  }

  .opened .sealed {
    opacity: 0;
    transform: translateY(0) scale(0.98);
  }

  .opened .unsealed {
    opacity: 1;
    transform: translateY(0) scale(1);
  }

  .tap {
    position: absolute;
    left: 50%;
    bottom: -12px;
    transform: translateX(-50%);
    background: rgba(255, 255, 255, 0.18);
    border: 1px solid rgba(255, 255, 255, 0.28);
    color: rgba(255, 255, 255, 0.92);
    padding: 6px 12px;
    border-radius: 999px;
    font-size: 13px;
    letter-spacing: 0.01em;
    backdrop-filter: blur(10px);
    transition: opacity 0.3s ease;
  }

  .envelope.fallen .tap {
    opacity: 0;
  }

  @media (max-width: 600px) {
    .tap { 
      bottom: -10px; 
    }
    
    .note {
      width: min(380px, 90vw);
    }
    
    .note-content {
      padding: 32px 28px 28px;
    }
    
    .handwritten {
      font-size: 22px;
    }
    
    .handwritten:first-child {
      font-size: 24px;
    }
    
    .signature {
      font-size: 26px;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .note {
      animation-duration: 0.4s;
    }
    
    .envelope {
      transition-duration: 0.3s;
    }

    .flashback-section {
      animation-duration: 0.6s;
    }
  }

  /* Flashback Section Styles */
  .flashback-section {
    width: 100%;
    padding: 40px 20px 60px;
    z-index: 5;
    opacity: 0;
    animation: fadeInUp 1.2s ease-out forwards;
  }

  @keyframes fadeInUp {
    0% {
      opacity: 0;
      transform: translateY(30px);
    }
    100% {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .flashback-title {
    font-family: 'Caveat', cursive;
    font-size: 32px;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.95);
    text-align: center;
    margin: 0 0 28px 0;
    text-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  }

  .polaroids {
    display: flex;
    gap: 20px;
    justify-content: center;
    align-items: flex-start;
    flex-wrap: wrap;
    padding: 0 10px;
  }

  .polaroid {
    background: #fdfbf7;
    padding: 16px 16px 20px;
    box-shadow: 
      0 4px 12px rgba(0, 0, 0, 0.2),
      0 8px 24px rgba(0, 0, 0, 0.15);
    border-radius: 2px;
    max-width: 200px;
    position: relative;
    animation: polaroidFloat 0.6s ease-out forwards;
    opacity: 0;
  }

  .polaroid-1 {
    transform: rotate(-4deg);
    animation-delay: 0.1s;
  }

  .polaroid-2 {
    transform: rotate(2deg);
    animation-delay: 0.3s;
  }

  .polaroid-3 {
    transform: rotate(-2deg);
    animation-delay: 0.5s;
  }

  @keyframes polaroidFloat {
    0% {
      opacity: 0;
      transform: translateY(20px) rotate(0deg);
    }
    100% {
      opacity: 1;
      transform: translateY(0) rotate(var(--rotate, 0deg));
    }
  }

  .polaroid-1 {
    --rotate: -4deg;
  }

  .polaroid-2 {
    --rotate: 2deg;
  }

  .polaroid-3 {
    --rotate: -2deg;
  }

  .polaroid-photo {
    width: 100%;
    aspect-ratio: 1;
    background: linear-gradient(135deg, #f0f0f0 0%, #e8e8e8 100%);
    border: 1px solid rgba(0, 0, 0, 0.08);
    margin-bottom: 12px;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    overflow: hidden;
  }

  .photo-placeholder {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    color: #999;
  }

  .photo-icon {
    font-size: 48px;
    opacity: 0.5;
  }

  .photo-text {
    font-family: 'Caveat', cursive;
    font-size: 18px;
    font-weight: 600;
    color: #666;
  }

  .polaroid-caption {
    font-family: 'Caveat', cursive;
    font-size: 20px;
    font-weight: 600;
    color: #2c1810;
    text-align: center;
    margin: 0 0 10px 0;
    line-height: 1.2;
  }

  .polaroid-note {
    background: rgba(255, 255, 220, 0.6);
    border-left: 3px solid rgba(255, 200, 100, 0.8);
    padding: 10px 12px;
    margin-top: 8px;
    border-radius: 2px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.08);
  }

  .note-text {
    font-family: 'Caveat', cursive;
    font-size: 16px;
    font-weight: 600;
    color: #3d2817;
    margin: 0;
    line-height: 1.4;
  }

  @media (max-width: 768px) {
    .flashback-section {
      padding: 30px 15px 50px;
    }

    .flashback-title {
      font-size: 28px;
      margin-bottom: 20px;
    }

    .polaroids {
      gap: 16px;
    }

    .polaroid {
      max-width: 160px;
      padding: 12px 12px 16px;
    }

    .photo-icon {
      font-size: 36px;
    }

    .photo-text {
      font-size: 16px;
    }

    .polaroid-caption {
      font-size: 18px;
    }

    .note-text {
      font-size: 14px;
    }
  }

  @media (max-width: 600px) {
    .polaroid {
      max-width: 140px;
    }
  }
</style>
