<img src="assets/icon.svg" width="64" height="64" alt="">

# Emoji Intensifier

A Svelte proof of concept for turning an emoji or image into a shaking, looping GIF. Images are processed locally in the browser.

```sh
npm install
npm run dev
```

Type an emoji or upload an image, crop uploaded images, frame the output, and adjust intensity, speed, and motion blur, then download the 128 × 128 GIF. White backgrounds preserve smooth motion trails; transparent GIFs are limited to one-bit alpha; animated uploads use their first frame. Copying GIFs depends on browser clipboard support; download works as a fallback.
