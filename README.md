# Tavus Green Screen WebGL CVI Demo

This project demonstrates a Tavus feature that enables real-time green screen removal in video calls using WebGL.

[LIVE DEMO](https://andy-tavus.github.io/CVI-greenscreen-webGL/)

## Features
- Connects to a video call created via the Tavus API.
- Subscribes to the existing participant’s (replica’s) video and audio streams.
- Removes the green screen background (RGB: `[0, 255, 155]`) using WebGL.
- Allows users to change the background color dynamically with a color picker.

## Usage
1. Use the Tavus API's [`create-conversation` endpoint](https://docs.tavus.io/api-reference/conversations/create-conversation) to generate a conversation with the `apply_greenscreen` property set to `true`.
2. Enter the conversation URL in the input field.
3. Click the "JOIN" button to connect.
4. The app will display the remote participant’s video with the green screen background removed.
5. Use the color picker to adjust the background color.

## Requirements
- A valid conversation URL created via the Tavus API.
- A browser that supports WebGL and the MediaStream API.

---

## Removing Green Fringe from Tavus CVI Replicas

When rendering Tavus CVI replicas with transparent backgrounds, you may notice a green outline around the replica. This guide covers GPU-accelerated techniques using a WebGL fragment shader to achieve clean, artifact-free edges.

## Fragment Shader

The demo uses uniforms for all tunable parameters so you can replicate the same values in your own app (see [Exposed settings](#exposed-settings-for-integration)).

```glsl
precision mediump float;
uniform sampler2D u_texture;
uniform float u_brightnessGate;
uniform float u_brightnessLow;
uniform float u_brightnessHigh;
uniform float u_chromaLow;
uniform float u_chromaHigh;
uniform float u_spillSuppress;
uniform float u_spillAmount;
uniform float u_premultiplyAlpha;
varying vec2 v_texCoord;

void main() {
  vec4 color = texture2D(u_texture, v_texCoord);
  float maxRB = max(color.r, color.b);
  float greenDominance = color.g - maxRB;
  float brightness = dot(color.rgb, vec3(0.299, 0.587, 0.114));
  float chromaKey = smoothstep(u_chromaLow, u_chromaHigh, greenDominance);
  float brightnessGate = u_brightnessGate > 0.5
    ? smoothstep(u_brightnessLow, u_brightnessHigh, brightness) : 1.0;
  float alpha = 1.0 - (chromaKey * brightnessGate);
  if (u_spillSuppress > 0.5) {
    float spillSuppress = smoothstep(0.0, 0.15, greenDominance);
    color.g = mix(color.g, min(color.g, maxRB + u_spillAmount), spillSuppress);
  }
  if (u_premultiplyAlpha > 0.5) {
    gl_FragColor = vec4(color.rgb * alpha, alpha);
  } else {
    gl_FragColor = vec4(color.rgb, alpha);
  }
}
```

## Technique Breakdown

### 1. Green Dominance Detection

```glsl
float maxRB = max(color.r, color.b);
float greenDominance = color.g - maxRB;
```

Instead of matching a specific RGB value, measure how much green exceeds the other channels. This catches all green screen variations regardless of lighting.

### 2. Brightness Gating

```glsl
float brightness = dot(color.rgb, vec3(0.299, 0.587, 0.114));
float brightnessGate = smoothstep(0.35, 0.45, brightness);
```

Dark pixels (shadows, hair) can appear greenish but shouldn't be keyed out. The brightness gate prevents removing dark areas that happen to have green tint.

### 3. Smooth Alpha Transitions

```glsl
float chromaKey = smoothstep(0.02, 0.25, greenDominance);
float alpha = 1.0 - (chromaKey * brightnessGate);
```

`smoothstep` creates mathematically smooth gradients instead of hard if/else cutoffs. This produces anti-aliased edges around hair, motion blur, and semi-transparent areas where the replica partially overlaps the green screen.

### 4. Green Spill Suppression

```glsl
float spillSuppress = smoothstep(0.0, 0.15, greenDominance);
color.g = mix(color.g, min(color.g, maxRB + 0.06), spillSuppress);
```

Green light reflects onto the replica, tinting skin and clothing edges. This clamps the green channel proportionally—stronger near green areas, leaving non-green colors untouched.

### 5. Premultiplied Alpha (optional)

```glsl
if (u_premultiplyAlpha > 0.5) {
  gl_FragColor = vec4(color.rgb * alpha, alpha);
} else {
  gl_FragColor = vec4(color.rgb, alpha);
}
```

WebGL’s default compositing assumes premultiplied alpha. If you output straight alpha `(r, g, b, a)` at semi-transparent edges, the compositor can show a white or light halo on non-white backgrounds. Outputting `(r*α, g*α, b*α, α)` fixes this so the canvas composites correctly over any background. Enable this when the replica has a visible light outline on colored backgrounds.

## Tuning Parameters

All of these are tunable in the demo’s “Edge effects” panel and are exposed via `TavusGreenscreen.getSettings()` for use in your own app.

| Parameter | Default | Effect |
|-----------|---------|--------|
| `u_premultiplyAlpha` | on | Premultiply RGB by alpha to avoid white/light outline when compositing |
| `u_chromaLow`, `u_chromaHigh` | 0.02–0.25 | Chroma key sensitivity. Lower = more aggressive keying |
| `u_brightnessGate` | on | When on, use brightness range below |
| `u_brightnessLow`, `u_brightnessHigh` | 0.35–0.45 | Brightness threshold. Higher = preserve more dark pixels |
| `u_spillSuppress` | on | When on, apply green spill suppression |
| `u_spillAmount` | 0.06 | Spill tolerance. Lower = stronger green removal |

## Exposed Settings for Integration

The demo exposes the current greenscreen settings so you can copy the same behavior into your own app or persist/restore user tweaks.

**API (global after the page loads):**

- **`TavusGreenscreen.getSettings()`**  
  Returns an object with the current values:

  | Property | Type | Description |
  |----------|------|--------------|
  | `chromaKeyOverride` | boolean | Whether a custom chroma key color is used instead of Tavus default |
  | `chromaKeyColor` | `{ r, g, b }` | 0–255. The key color (Tavus default or user override). |
  | `premultiplyAlpha` | boolean | Use premultiplied alpha output to fix outline on colored backgrounds |
  | `brightnessGate` | boolean | Enable brightness gating (preserve dark pixels) |
  | `brightnessLow` | number | 0–1. Brightness range low (default 0.35) |
  | `brightnessHigh` | number | 0–1. Brightness range high (default 0.45) |
  | `chromaLow` | number | 0–1. Chroma key sensitivity low (default 0.02) |
  | `chromaHigh` | number | 0–1. Chroma key sensitivity high (default 0.25) |
  | `spillSuppress` | boolean | Enable green spill suppression |
  | `spillAmount` | number | 0–1. Spill amount (default 0.06) |

- **`TavusGreenscreen.setSettings(settings)`**  
  Apply a partial or full settings object (e.g. from `getSettings()` or your own config). Only provided keys are updated; others keep the current UI state. Useful for restoring saved preferences or driving the demo from another app.

**Example: read and reuse in your app**

```javascript
const settings = window.TavusGreenscreen.getSettings();
// Use settings.chromaLow, settings.chromaHigh, etc. as shader uniforms in your WebGL pipeline.
```

**Example: restore saved preferences**

```javascript
const saved = JSON.parse(localStorage.getItem('greenscreenSettings'));
if (saved) window.TavusGreenscreen.setSettings(saved);
// ... later: localStorage.setItem('greenscreenSettings', JSON.stringify(window.TavusGreenscreen.getSettings()));
```

## Result

With these techniques combined, your Tavus replica floats cleanly over any background—no green halo, no white outline, smooth edges, and 60fps performance at any resolution.
