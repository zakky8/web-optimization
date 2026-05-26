# Framer Motion — React Animation Library

Repo: https://github.com/framer/motion
Current version: v11+ — Active as of 2026-05

## Installation

```bash
npm install framer-motion
```

## Core — motion Components

```jsx
import { motion, AnimatePresence } from 'framer-motion';

// Any HTML element → motion.div, motion.span, motion.path, etc.
<motion.div
  initial={{ opacity: 0, y: 20 }}  // Start state
  animate={{ opacity: 1, y: 0 }}   // Target state
  exit={{ opacity: 0, y: -20 }}    // Exit state (needs AnimatePresence)
  transition={{
    duration: 0.4,
    ease: [0.22, 1, 0.36, 1],  // Custom cubic-bezier
    // OR: ease: 'easeOut', 'spring', 'tween'
  }}
/>

// Spring physics
<motion.div
  animate={{ x: 100 }}
  transition={{
    type: 'spring',
    stiffness: 300,
    damping: 20,
    mass: 1,
  }}
/>
```

## AnimatePresence — Mount/Unmount Animations

```jsx
<AnimatePresence mode="wait">
  {isVisible && (
    <motion.div
      key="modal"  // REQUIRED: unique key for AnimatePresence to track
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95 }}
      transition={{ duration: 0.2 }}
    >
      Modal content
    </motion.div>
  )}
</AnimatePresence>
```

## Variants — Coordinated Animations

```jsx
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,   // Each child delayed by 0.1s
      delayChildren: 0.2,     // Initial delay
    },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0 },
};

<motion.ul variants={containerVariants} initial="hidden" animate="visible">
  {items.map(item => (
    <motion.li key={item} variants={itemVariants}>
      {item}
    </motion.li>
  ))}
</motion.ul>
```

## useMotionValue + useTransform

```jsx
import { useMotionValue, useTransform, motion } from 'framer-motion';

function ParallaxCard() {
  const x = useMotionValue(0);
  const y = useMotionValue(0);

  // Transform cursor position to rotation
  const rotateX = useTransform(y, [-200, 200], [15, -15]);
  const rotateY = useTransform(x, [-200, 200], [-15, 15]);

  return (
    <motion.div
      style={{ rotateX, rotateY, perspective: 1000 }}
      onMouseMove={(e) => {
        const rect = e.currentTarget.getBoundingClientRect();
        x.set(e.clientX - rect.left - rect.width / 2);
        y.set(e.clientY - rect.top - rect.height / 2);
      }}
      onMouseLeave={() => { x.set(0); y.set(0); }}
    >
      Card content
    </motion.div>
  );
}
```

## Scroll-Based Animation

```jsx
import { useScroll, useTransform, motion } from 'framer-motion';

function ParallaxSection({ children }) {
  const ref = useRef(null);

  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],
  });

  const y = useTransform(scrollYProgress, [0, 1], ['-20%', '20%']);

  return (
    <div ref={ref} style={{ overflow: 'hidden' }}>
      <motion.div style={{ y }}>
        {children}
      </motion.div>
    </div>
  );
}
```

## Layout Animations

```jsx
// Animates layout changes automatically
<motion.div
  layout
  style={{ width: isExpanded ? 300 : 100 }}
>
  {isExpanded ? 'Full content' : 'Collapsed'}
</motion.div>

// Shared element transitions between components
<motion.div layoutId="modal-bg" />  // In list
<motion.div layoutId="modal-bg" />  // In modal — animates between positions
```

## Sources
- Framer Motion docs: https://www.framer.com/motion/
- GitHub: https://github.com/framer/motion
- Motion for React blog: https://motion.dev/
