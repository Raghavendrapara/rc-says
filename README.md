# Raghavendra Singh Chauhan - Personal Portfolio

A modern, interactive portfolio website showcasing my professional experience as a Software Engineer. Features stunning visual effects including particle constellations, 3D card interactions, and smooth animations.

> 🤖 **Powered by [Claude](https://claude.ai)** - Built with AI-assisted development

## 🌐 Live Site

**[https://Raghavendrapara.github.io/rc-says](https://Raghavendrapara.github.io/rc-says)**

## ✨ Features

### Visual Effects
- **Particle Constellation Background** - Interactive particles that connect with lines and react to cursor movement
- **3D Tilt Cards** - Cards rotate in 3D space based on mouse position
- **Cursor Glow** - Ambient gradient that follows your cursor
- **Magnetic Buttons** - Buttons that attract toward the cursor
- **Diffusion Text Reveal** - Blur-to-sharp animation for header text
- **Scroll Reveal** - Sections fade in beautifully as you scroll
- **Animated Gradient Header** - Flowing color gradient with mesh overlays

### Design
- **Light/Dark Mode** - Toggle between themes with particle color adaptation
- **Responsive Layout** - Optimized for desktop and mobile
- **Glassmorphism** - Frosted glass effect with backdrop blur
- **Modern Typography** - Inter font with carefully tuned weights

### Content
- Professional experience timeline
- Project showcase with tech stacks
- Skills organized by category
- Education and achievements

## 🛠️ Tech Stack

| Technology | Usage |
|------------|-------|
| HTML5 | Semantic structure |
| CSS3 | Custom properties, Grid, Flexbox, Animations |
| JavaScript | Canvas API, Intersection Observer, Event handling |
| GitHub Pages | Hosting |

## 📖 Local Setup

```bash
git clone https://github.com/Raghavendrapara/rc-says.git
cd rc-says
# Open index.html in your browser
```

## 🎨 Customization

### Colors
Edit CSS custom properties in `style.css`:
```css
:root {
    --accent: #6366f1;      /* Primary accent */
    --secondary: #8b5cf6;   /* Secondary accent */
    --tertiary: #06b6d4;    /* Tertiary accent */
}
```

### Particles
Adjust particle behavior in `index.html`:
```javascript
const numberOfParticles = 120;  // Particle count
mouse.radius = 150;             // Interaction radius
```

## 📱 Performance

- Animations disabled on mobile for smooth scrolling
- Respects `prefers-reduced-motion` accessibility setting
- Optimized particle count based on screen size

## 📜 License

Code is open for public use. Personal content and resume details are proprietary.
