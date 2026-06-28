<script lang="ts">
  import { onMount } from 'svelte';

  export type GraphicMode = 'loading' | 'speaking' | 'thinking' | 'waiting';

  let { mode = 'waiting' }: { mode?: GraphicMode } = $props();

  let canvas: HTMLCanvasElement | undefined = $state();
  const modeRef: { current: GraphicMode } = { current: 'waiting' };

  $effect(() => {
    modeRef.current = mode;
  });

  const SPEEDS: Record<GraphicMode, number> = {
    loading: 0.55,
    waiting: 0.32,
    thinking: 1.5,
    speaking: 5.4,
  };

  onMount(() => {
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const size = 300;
    const dpr = window.devicePixelRatio || 1;
    canvas.width = size * dpr;
    canvas.height = size * dpr;
    canvas.style.width = `${size}px`;
    canvas.style.height = `${size}px`;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

    const particles = Array.from({ length: 14 }, (_, index) => ({
      offset: (index / 14) * Math.PI * 2,
      radius: 64 + (index % 4) * 16,
      size: 2.5 + (index % 3) * 1.2,
    }));

    let angle = 0;
    let speed = SPEEDS.waiting;
    let breathe = 0;
    let last = performance.now();
    let frameId = 0;

    const tick = (now: number) => {
      const dt = Math.min((now - last) / 1000, 0.05);
      last = now;

      const target = SPEEDS[modeRef.current];
      speed += (target - speed) * Math.min(1, dt * 3.5);
      angle += speed * dt;
      breathe += dt * (0.7 + speed * 0.12);

      const cx = size / 2;
      const cy = size / 2;

      ctx.clearRect(0, 0, size, size);

      const pulse = 0.58 + Math.sin(breathe) * 0.1;
      const glow = ctx.createRadialGradient(cx, cy, 8, cx, cy, 98 * pulse);
      glow.addColorStop(0, `rgba(16, 163, 127, ${0.28 + Math.min(speed, 3) * 0.06})`);
      glow.addColorStop(1, 'rgba(16, 163, 127, 0)');
      ctx.fillStyle = glow;
      ctx.beginPath();
      ctx.arc(cx, cy, 100, 0, Math.PI * 2);
      ctx.fill();

      ctx.strokeStyle = `rgba(16, 163, 127, ${0.2 + Math.min(speed, 3) * 0.06})`;
      ctx.lineWidth = 1.5;
      ctx.beginPath();
      ctx.arc(cx, cy, 66 + Math.sin(breathe * 1.15) * (3 + speed * 0.4), 0, Math.PI * 2);
      ctx.stroke();

      for (const particle of particles) {
        const orbit =
          particle.radius + Math.sin(breathe * 1.8 + particle.offset) * (4 + speed * 0.35);
        const a = angle + particle.offset + Math.sin(breathe + particle.offset) * 0.06 * speed;
        const x = cx + Math.cos(a) * orbit;
        const y = cy + Math.sin(a) * orbit;

        ctx.fillStyle = `rgba(236, 236, 236, ${0.45 + Math.min(speed, 4) * 0.08})`;
        ctx.beginPath();
        ctx.arc(x, y, particle.size + speed * 0.12, 0, Math.PI * 2);
        ctx.fill();
      }

      ctx.fillStyle = '#10a37f';
      ctx.beginPath();
      ctx.arc(cx, cy, 9 + Math.min(speed, 4) * 0.75, 0, Math.PI * 2);
      ctx.fill();

      frameId = requestAnimationFrame(tick);
    };

    frameId = requestAnimationFrame(tick);

    return () => cancelAnimationFrame(frameId);
  });
</script>

<canvas bind:this={canvas} class="voice-graphic" aria-hidden="true"></canvas>

<style>
  .voice-graphic {
    display: block;
    margin: 0 auto;
  }
</style>
