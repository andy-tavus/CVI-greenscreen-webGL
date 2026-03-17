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

```glsl
precision mediump float;
uniform sampler2D u_texture;
varying vec2 v_texCoord;

void main() {
  vec4 color = texture2D(u_texture, v_texCoord);
  
  // 1. Green dominance detection
  float maxRB = max(color.r, color.b);
  float greenDominance = color.g - maxRB;
  
  // 2. Brightness gating
  float brightness = dot(color.rgb, vec3(0.299, 0.587, 0.114));
  
  // 3. Smooth alpha with softened edges
  float chromaKey = smoothstep(0.02, 0.25, greenDominance);
  float brightnessGate = smoothstep(0.35, 0.45, brightness);
  float alpha = 1.0 - (chromaKey * brightnessGate);
  
  // 4. Green spill suppression
  float spillSuppress = smoothstep(0.0, 0.15, greenDominance);
  color.g = mix(color.g, min(color.g, maxRB + 0.06), spillSuppress);
  
  gl_FragColor = vec4(color.rgb, alpha);
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

## Tuning Parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `smoothstep(0.02, 0.25, ...)` | 0.02–0.25 | Chroma key sensitivity. Lower = more aggressive keying |
| `smoothstep(0.35, 0.45, ...)` | 0.35–0.45 | Brightness threshold. Higher = preserve more dark pixels |
| `maxRB + 0.06` | 0.06 | Spill tolerance. Lower = stronger green removal |

## Result

With all four techniques combined, your Tavus replica floats cleanly over any background—no green halo, smooth edges, and 60fps performance at any resolution.
