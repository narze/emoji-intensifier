<script>
  import { onMount } from 'svelte';
  import { GIFEncoder, quantize, applyPalette } from 'gifenc';

  let canvas, fileInput;
  let emoji = '😂', image = null, fileName = '', intensity = 5, speed = 50, blur = 3;
  let busy = false, message = '', dragging = false;
  let cropZoom = 1, cropX = 0, cropY = 0;
  let background = 'transparent';
  let outputZoom = 1, outputX = 0, outputY = 0;
  const size = 128, frames = 12;
  // A deterministic loop keeps the live preview and exported GIF in sync.
  const motion = [[-1,-.3, -3],[.8,.8, 3],[-.6,1,-2],[1,-.8,3],[-1,.4,-3],[.7,-1,2],[-.8,-.7,-2],[1,.4,3],[-.5,1,-3],[.8,-.5,2],[-1,.2,-2],[.5,-1,3]];
  let emojiLayoutKey = '', emojiLayout = { x: 0, y: 0, scale: 1 };
  function getEmojiLayout() {
    const key = `${emoji}|${intensity}`;
    if (key === emojiLayoutKey) return emojiLayout;
    emojiLayoutKey = key;
    const measure = document.createElement('canvas');
    measure.width = measure.height = 256;
    const context = measure.getContext('2d', { willReadFrequently: true });
    context.textAlign = 'center'; context.textBaseline = 'middle';
    context.font = '90px "Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji", sans-serif';
    context.fillText(emoji || '😂', 128, 133, 108);
    const pixels = context.getImageData(0, 0, 256, 256).data;
    let left = 256, right = -1, top = 256, bottom = -1;
    for (let y = 0; y < 256; y++) for (let x = 0; x < 256; x++) {
      if (pixels[(y * 256 + x) * 4 + 3] > 0) {
        left = Math.min(left, x); right = Math.max(right, x);
        top = Math.min(top, y); bottom = Math.max(bottom, y);
      }
    }
    if (right < left) return emojiLayout = { x: 0, y: 0, scale: 1 };
    const centerX = (left + right + 1) / 2 - 128;
    const centerY = (top + bottom + 1) / 2 - 128;
    const halfWidth = (right - left + 1) / 2, halfHeight = (bottom - top + 1) / 2;
    let scale = Infinity;
    for (const [x, y, angle] of motion) {
      const radians = angle * Number(intensity) / 5 * Math.PI / 180;
      const c = Math.abs(Math.cos(radians)), sin = Math.abs(Math.sin(radians));
      const extentX = halfWidth * c + halfHeight * sin;
      const extentY = halfWidth * sin + halfHeight * c;
      scale = Math.min(scale, (63 - Math.abs(x * intensity)) / extentX, (63 - Math.abs(y * intensity)) / extentY);
    }
    return emojiLayout = { x: centerX, y: centerY, scale: Math.max(0.1, scale) };
  }
  function draw(ctx, frame) {
    ctx.clearRect(0, 0, size, size);
    if (background === 'white') { ctx.fillStyle = '#ffffff'; ctx.fillRect(0, 0, size, size); }
    ctx.save();
    // Frame the entire animation, including its motion trails, inside the output square.
    ctx.translate(size / 2 + Number(outputX), size / 2 + Number(outputY));
    ctx.scale(Number(outputZoom), Number(outputZoom));
    ctx.translate(-size / 2, -size / 2);
    function drawAt(index, opacity) {
      const [x, y, angle] = motion[(index + frames) % frames];
      ctx.save(); ctx.globalAlpha = opacity;
      ctx.translate(size/2 + x * intensity, size/2 + y * intensity);
      ctx.rotate(angle * intensity / 5 * Math.PI / 180);
      if (image) {
        const side = Math.min(image.width, image.height) / Number(cropZoom);
        const sx = (image.width - side) * (Number(cropX) + 100) / 200;
        const sy = (image.height - side) * (Number(cropY) + 100) / 200;
        ctx.drawImage(image, sx, sy, side, side, -50, -50, 100, 100);
      } else {
        const layout = getEmojiLayout();
        ctx.scale(layout.scale, layout.scale); ctx.translate(-layout.x, -layout.y);
        ctx.textAlign = 'center'; ctx.textBaseline = 'middle'; ctx.font = '90px "Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji", sans-serif';
        ctx.fillText(emoji || '😂', 0, 5, 108);
      }
      ctx.restore();
    }
    // Layer earlier positions underneath the current frame to leave a motion trail.
    const amount = Number(blur) / 5;
    if (amount > 0) {
      drawAt(frame - 3, 0.18 * amount);
      drawAt(frame - 2, 0.30 * amount);
      drawAt(frame - 1, 0.48 * amount);
    }
    drawAt(frame, 1);
    ctx.restore();
  }
  onMount(() => {
    let id, start = performance.now();
    function tick(now) { if (canvas) draw(canvas.getContext('2d'), Math.floor((now-start)/Number(speed)) % frames); id = requestAnimationFrame(tick); }
    id = requestAnimationFrame(tick); return () => cancelAnimationFrame(id);
  });
  function loadFile(file) {
    if (!file) return;
    if (!file.type.startsWith('image/')) { message = 'Please choose an image file.'; return; }
    const url = URL.createObjectURL(file); const next = new Image();
    next.onload = () => { image = next; fileName = file.name; cropZoom = 1; cropX = 0; cropY = 0; message = ''; URL.revokeObjectURL(url); };
    next.onerror = () => { message = 'Could not open that image. Try a PNG, JPG, WebP or GIF.'; URL.revokeObjectURL(url); };
    next.src = url;
  }
  function useEmoji() { image = null; fileName = ''; message = ''; }
  async function makeGif() {
    const offscreen = document.createElement('canvas'); offscreen.width = offscreen.height = size;
    const ctx = offscreen.getContext('2d', { willReadFrequently: true }); const gif = GIFEncoder();
    for (let i = 0; i < frames; i++) {
      draw(ctx, i);
      const data = ctx.getImageData(0, 0, size, size).data;
      // Reserve index 0 for transparency; GIF supports one-bit alpha only.
      const palette = quantize(data, 255, { format: 'rgb565' });
      const indexed = applyPalette(data, palette, 'rgb565');
      // GIF has binary transparency. A white background preserves smooth translucent
      // trails; transparent output uses a solid alpha cutoff instead of dotted dithering.
      for (let p = 0; p < indexed.length; p++) indexed[p] = data[p*4+3] < 128 ? 0 : indexed[p] + 1;
      gif.writeFrame(indexed, size, size, { palette: [[0,0,0], ...palette], transparent: true, transparentIndex: 0, delay: Number(speed), dispose: 2, repeat: 0 });
      await new Promise(resolve => setTimeout(resolve, 0));
    }
    gif.finish(); return new Blob([gif.bytes()], { type: 'image/gif' });
  }
  async function exportGif(action) {
    busy = true; message = '';
    try {
      const blob = await makeGif();
      if (action === 'copy') {
        if (!navigator.clipboard?.write || !window.ClipboardItem || (ClipboardItem.supports && !ClipboardItem.supports('image/gif'))) throw new Error('This browser can’t copy GIFs directly. Download the GIF instead.');
        await navigator.clipboard.write([new ClipboardItem({ 'image/gif': blob })]);
        message = 'Copied GIF to clipboard!';
      } else {
        const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = 'emoji-intensified.gif'; a.click(); setTimeout(() => URL.revokeObjectURL(url), 1000);
        message = 'Your GIF is ready!';
      }
    } catch (error) { message = error.message || 'Something went wrong while creating the GIF.'; }
    finally { busy = false; }
  }
</script>

<svelte:head><meta name="description" content="Turn any emoji into a wonderfully wobbly GIF." /></svelte:head>
<div class="page">
  <header class="topbar"><a class="brand" href="/" aria-label="Emoji Intensifier home"><span class="brand-icon">✳</span><span>emoji<span class="brand-light">intensifier</span></span></a><span class="top-note"><span class="sparkle">✳</span> A LITTLE TOOL FOR BIG FEELINGS</span></header>
  <main>
    <div class="intro"><div class="eyebrow"><span></span> THE EMOJI LAB <span></span></div><h1>Make it <em>move.</em></h1><p>For when an emoji just isn't feeling enough.</p></div>
    <section class="workspace" aria-label="Emoji editor">
      <div class="controls">
        <div class="section-heading"><span class="step">01</span><div><h2>Pick your emoji</h2><p>Start with a face or bring your own.</p></div></div>
        <div class="emoji-entry"><span class="input-label">TYPE AN EMOJI</span><div class="emoji-field"><input aria-label="Type an emoji" bind:value={emoji} oninput={useEmoji} maxlength="16"/><button class="use-btn" onclick={useEmoji}>Use emoji <span>↗</span></button></div></div>
        <div class="or"><span></span> OR <span></span></div>
        <button class:dragging class="dropzone" onclick={() => fileInput.click()} ondragover={(e) => {e.preventDefault(); dragging = true;}} ondragleave={() => dragging = false} ondrop={(e) => {e.preventDefault(); dragging = false; loadFile(e.dataTransfer.files[0]);}}>
          <span class="upload-icon">↑</span><span><strong>{fileName || 'Drop an image here'}</strong><small>or click to upload · PNG, JPG, GIF, WebP</small></span><span class="upload-arrow">↗</span>
        </button><input bind:this={fileInput} type="file" accept="image/*" hidden onchange={(e) => loadFile(e.currentTarget.files[0])}/>
        {#if image}
          <div class="crop-controls" aria-label="Image crop controls">
            <div class="crop-header"><div><strong>Crop image</strong><small>Adjust the square area shown in the preview.</small></div><button class="reset-crop" onclick={() => { cropZoom = 1; cropX = 0; cropY = 0; }}>Reset</button></div>
            <div class="crop-slider"><label for="crop-zoom">Zoom <span>{Number(cropZoom).toFixed(1)}×</span></label><input id="crop-zoom" type="range" min="1" max="4" step="0.1" bind:value={cropZoom}/></div>
            <div class="crop-slider"><label for="crop-x">Horizontal <span>{cropX}</span></label><input id="crop-x" type="range" min="-100" max="100" step="1" bind:value={cropX}/></div>
            <div class="crop-slider"><label for="crop-y">Vertical <span>{cropY}</span></label><input id="crop-y" type="range" min="-100" max="100" step="1" bind:value={cropY}/></div>
          </div>
        {/if}
        <div class="crop-controls" aria-label="Output crop controls">
          <div class="background-control"><label for="background">Background</label><select id="background" bind:value={background}><option value="white">White · smooth trails</option><option value="transparent">Transparent</option></select><small>GIF transparency cannot preserve soft edges. White keeps motion blur smooth.</small></div>
          <div class="crop-header"><div><strong>Crop output</strong><small>Frame the finished animation in the GIF.</small></div><button class="reset-crop" onclick={() => { outputZoom = 1; outputX = 0; outputY = 0; }}>Reset</button></div>
          <div class="crop-slider"><label for="output-zoom">Zoom <span>{Number(outputZoom).toFixed(1)}×</span></label><input id="output-zoom" type="range" min="0.5" max="3" step="0.1" bind:value={outputZoom}/></div>
          <div class="crop-slider"><label for="output-x">Horizontal <span>{outputX} px</span></label><input id="output-x" type="range" min="-64" max="64" step="1" bind:value={outputX}/></div>
          <div class="crop-slider"><label for="output-y">Vertical <span>{outputY} px</span></label><input id="output-y" type="range" min="-64" max="64" step="1" bind:value={outputY}/></div>
        </div>
        <div class="divider"></div>
        <div class="section-heading tune-heading"><span class="step">02</span><div><h2>Turn it up</h2><p>Find the right amount of chaos.</p></div></div>
        <div class="slider-row"><div class="slider-title"><label for="intensity">Intensity</label><span>{intensity} / 10</span></div><input id="intensity" type="range" min="1" max="10" bind:value={intensity}/><div class="slider-ends"><span>SUBTLE</span><span>UNHINGED</span></div></div>
        <div class="slider-row"><div class="slider-title"><label for="speed">Speed</label><span>{speed <= 50 ? 'Fast' : speed <= 90 ? 'Medium' : 'Chill'}</span></div><input id="speed" type="range" min="30" max="150" step="10" value={180-speed} oninput={(e) => speed = 180 - Number(e.currentTarget.value)}/><div class="slider-ends"><span>CHILL</span><span>ZOOMIES</span></div></div>
        <div class="slider-row"><div class="slider-title"><label for="blur">Motion blur</label><span>{blur} / 5</span></div><input id="blur" type="range" min="0" max="5" step="0.5" bind:value={blur}/><div class="slider-ends"><span>OFF</span><span>TRAILS</span></div></div>
      </div>
      <div class="preview-panel"><div class="preview-top"><span><span class="live-dot"></span> LIVE PREVIEW</span><span>128 × 128 PX</span></div><div class="preview-stage"><span class="deco deco-a">✳</span><span class="deco deco-b">✦</span><div class="preview-box"><canvas bind:this={canvas} width={size} height={size} aria-label="Animated emoji preview"></canvas></div><span class="deco deco-c">✳</span><span class="deco deco-d">✦</span></div><div class="preview-caption"><strong>Looking good!</strong><span>{background === 'white' ? 'Smooth trails on white.' : 'Transparent background, infinite energy.'}</span></div><div class="actions"><button class="download" disabled={busy} onclick={() => exportGif('download')}><span>↓</span> {busy ? 'Making magic…' : 'Download GIF'}</button><button class="copy" disabled={busy} onclick={() => exportGif('copy')}><span>▢</span> Copy GIF</button></div><p class="export-note" aria-live="polite">{message || 'MADE TO LOOP FOREVER  ·  GIF FORMAT'}</p></div>
    </section>
    <div class="bottom-note"><span>✦</span> A little shake goes a long way. <span>✦</span></div>
  </main><footer><span>EMOJI INTENSIFIER <span class="footer-star">✳</span></span><span>MADE FOR THE DRAMATIC</span></footer>
</div>
