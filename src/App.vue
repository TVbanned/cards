<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

function adjustColor(color, amount) {
  const num = parseInt(color.slice(1), 16);
  const r = Math.min(255, Math.max(0, (num >> 16) + amount));
  const g = Math.min(255, Math.max(0, ((num >> 8) & 0x00FF) + amount));
  const b = Math.min(255, Math.max(0, (num & 0x0000FF) + amount));
  return '#' + (0x1000000 + r * 0x10000 + g * 0x100 + b).toString(16).slice(1);
}

function generateGradient(color) {
  const canvas = document.createElement('canvas');
  canvas.width = 400;
  canvas.height = 500;
  const ctx = canvas.getContext('2d');

  const gradient = ctx.createLinearGradient(0, 0, canvas.width, canvas.height);
  gradient.addColorStop(0, color);
  gradient.addColorStop(1, adjustColor(color, -30));
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.globalAlpha = 0.1;

  for (let i = 0; i < 50; i++) {
    ctx.beginPath();
    ctx.arc(Math.random() * canvas.width, Math.random() * canvas.height, Math.random() * 50, 0, Math.PI * 2);
    ctx.fillStyle = '#fff';
    ctx.fill();
  }

  return canvas.toDataURL();
}

// 作品数据
const baseWorks = [
  { id: 1, title: "FOLDED MATTER", year: "2020", color: "#F59E0B" },
  { id: 2, title: "WAVE THEORY", year: "2020", color: "#3B82F6" },
  { id: 3, title: "HUMAN SIGNAL", year: "2021", color: "#EF4444" },
  { id: 4, title: "QUIET BLOOM", year: "2021", color: "#10B981" },
  { id: 5, title: "MEASURED LIGHT", year: "2021", color: "#8B5CF6" },
  { id: 6, title: "TORN HORIZON", year: "2022", color: "#F97316" },
  { id: 7, title: "SOFT STRUCTURE", year: "2022", color: "#EC4899" },
  { id: 8, title: "DARK EQUATOR", year: "2022", color: "#6366F1" },
  { id: 9, title: "FLOATING WORLD", year: "2023", color: "#14B8A6" },
  { id: 10, title: "PAPER SKY", year: "2023", color: "#84CC16" },
  { id: 11, title: "COLD MOUNTAIN", year: "2023", color: "#06B6D4" },
  { id: 12, title: "SILENT ROOM", year: "2024", color: "#F43F5E" },
  { id: 13, title: "GLASS TIME", year: "2024", color: "#A855F7" },
  { id: 14, title: "BRICK MEMORY", year: "2024", color: "#0EA5E9" },
  { id: 15, title: "WIND STUDY", year: "2024", color: "#84CC16" },
  { id: 16, title: "SLOW MOTION", year: "2025", color: "#F97316" },
  { id: 17, title: "BLUE PRINT", year: "2025", color: "#3B82F6" },
  { id: 18, title: "GRAY AREA", year: "2025", color: "#6B7280" },
  { id: 19, title: "GOLDEN HOUR", year: "2025", color: "#F59E0B" },
  { id: 20, title: "NEW MORNING", year: "2026", color: "#10B981" },
  { id: 21, title: "DEEP SPACE", year: "2026", color: "#1E3A8A" },
  { id: 22, title: "HIGH TIDE", year: "2026", color: "#0891B2" },
  { id: 23, title: "LOW PRESSURE", year: "2026", color: "#7C3AED" },
  { id: 24, title: "CLEAR VIEW", year: "2026", color: "#DC2626" }
];

const works = baseWorks.map((work) => ({
  ...work,
  slug: `${work.title.toLowerCase().replace(/\s+/g, '_')}.jpg`,
  image: generateGradient(work.color)
}));

// 状态管理
const currentView = ref('layered');
let mouseGX = Infinity;
let mouseGY = Infinity;
let isMouseInGallery = false;
const cardStates = [];

// 布局常量
const DEG2RAD = Math.PI / 180;
const ANGLE = 15 * DEG2RAD; // 15度倾斜角
const CARD_DISTANCE = 45; // 卡片之间的距离
const STEP_X = CARD_DISTANCE * Math.cos(ANGLE); // 沿着角度的X分量
const STEP_Y = -CARD_DISTANCE * Math.sin(ANGLE); // 沿着角度的Y分量（向下为正，所以是负的）
const NUM = works.length;
const MID = (NUM - 1) / 2;
const ROTATE_SPREAD = 5;
const MOTION_EPSILON = 0.0005;

// 物理常量
const STIFFNESS = 0.1; // 大幅提升速度
const DAMPING = 1; // 大幅增加阻尼到0.9，运动更柔和，延迟下落时间更长
const INFLUENCE_RADIUS = 220;

// 卡片DOM引用
const cardsRef = [];
const galleryRef = ref(null);
let galleryBounds = null;
let layeredNeedsFullRender = true;

// 静息位置
const cardRestX = (index) => {
  return -(MID * STEP_X) + index * STEP_X;
};

const cardRestY = (index) => {
  const offsetY = Math.abs(STEP_Y) * 0.4;
  return MID * offsetY + index * STEP_Y;
};

// 获取卡片 transform
const getLayeredTransform = (index) => {
  const state = cardStates[index];
  const lift = Math.max(0, Math.min(1, state.lift));
  const n = NUM - 1;

  // 基础位置
  let x = cardRestX(index);
  let y = cardRestY(index);

  // 静息旋转
  const restRotateY = -14; // 调小倾斜角度，更朝向用户
  const restRotateX = -3;

  // translateZ
  const restTz = (n - index) * 4;

  // 卡片向上抬起一部分，保持夹在中间的感觉
  const outY = y - lift * 120; // 抬起幅度更大
  // 使用平滑的rotateLift状态，从-14到-10，增加4度
  const rotateY = restRotateY + state.rotateLift * 4; 
  const rotateX = restRotateX;
  const rotateZ = lift * -5; // 向左倾斜5度
  const tz = restTz; // 保持原始的深度！！！不改变translateZ
  const scaleVal = 1; // 不放大！！！

  // z-index
  const zi = 500 - index + Math.round(lift * 500); // 稍微调整一点点
  if (cardsRef[index] && state.lastZIndex !== zi) {
    cardsRef[index].style.zIndex = zi;
    state.lastZIndex = zi;
  }

  return `translateX(${x}px) translateY(${outY}px) translateZ(${tz}px) rotateY(${rotateY}deg) rotateX(${rotateX}deg) rotateZ(${rotateZ}deg) scale(${scaleVal})`;
};

const getOrbitTransform = (index) => {
  const angle = (index / NUM) * Math.PI * 2;
  const radius = 380;
  const dynamicAngle = angle + (mouseGX / 1000) * 0.5;

  const x = Math.sin(dynamicAngle) * radius;
  const z = Math.cos(dynamicAngle) * radius - 300;
  const y = (mouseGY / 1000) * 60 * Math.sin(dynamicAngle);
  const rotateY = (dynamicAngle * 180 / Math.PI) + 90;

  return `translateX(${x}px) translateY(${y}px) translateZ(${z}px) rotateY(${rotateY}deg)`;
};

// 物理模拟
let animationId = null;

// 找到离鼠标最近的卡片
let closestIndex = -1;
const lastClosestIndex = { value: -1 };

const isStateActive = (state) => {
  return (
    Math.abs(state.lift) > MOTION_EPSILON ||
    Math.abs(state.velocity) > MOTION_EPSILON ||
    Math.abs(state.rotateLift) > MOTION_EPSILON ||
    Math.abs(state.rotateVelocity) > MOTION_EPSILON
  );
};

const settleState = (state) => {
  if (Math.abs(state.lift) <= MOTION_EPSILON && Math.abs(state.velocity) <= MOTION_EPSILON) {
    state.lift = 0;
    state.velocity = 0;
  }

  if (Math.abs(state.rotateLift) <= MOTION_EPSILON && Math.abs(state.rotateVelocity) <= MOTION_EPSILON) {
    state.rotateLift = 0;
    state.rotateVelocity = 0;
  }
};

const updateClosestIndex = () => {
  if (!isMouseInGallery || currentView.value !== 'layered') {
    if (closestIndex !== -1) layeredNeedsFullRender = true;
    closestIndex = -1;
    return;
  }

  let minDistSq = Infinity;
  let nextClosestIndex = -1;

  for (let i = 0; i < NUM; i++) {
    const cx = cardRestX(i);
    const cy = cardRestY(i);
    const dx = mouseGX - cx;
    const dy = mouseGY - cy;
    const distSq = dx * dx + dy * dy;
    if (distSq < minDistSq) {
      minDistSq = distSq;
      nextClosestIndex = i;
    }
  }

  if (closestIndex !== nextClosestIndex) {
    layeredNeedsFullRender = true;
    lastClosestIndex.value = closestIndex;
    closestIndex = nextClosestIndex;
  }
};

const updatePhysics = () => {
  let hasActiveMotion = false;

  cardStates.forEach((state, i) => {
    const distance = closestIndex >= 0 ? i - closestIndex : Infinity;
    const wasNearPrevious = lastClosestIndex.value >= 0 && i >= lastClosestIndex.value && i < lastClosestIndex.value + ROTATE_SPREAD;
    const isNearCurrent = distance >= 0 && distance < ROTATE_SPREAD;
    const shouldSimulate = i === closestIndex || isNearCurrent || wasNearPrevious || isStateActive(state);

    if (!shouldSimulate) {
      return;
    }

    // 原有的lift物理模拟
    let targetLift;
    if (i === closestIndex && isMouseInGallery) {
      targetLift = 1;
    } else {
      targetLift = 0;
    }

    let stiffness;
    let damping;
    if (targetLift > state.lift) {
      // 在抬起，运动曲线：一开始快，然后慢
      const t = state.lift;
      const factor = Math.pow(1 - t, 1.5);
      stiffness = 0.1 + factor * 0.25;
      damping = 0.88;
    } else {
      // 在下落下，运动曲线：一开始极其慢，然后变正常下落速度
      const t = state.lift;
      const factor = Math.pow(1 - t, 2);
      stiffness = 0.003 + factor * 0.12;
      damping = 0.96;
    }

    const force = stiffness * (targetLift - state.lift);
    const dampedVel = damping * state.velocity;
    state.velocity += force - dampedVel;
    state.lift += state.velocity;

    // 新增：rotateLift的物理模拟
    let targetRotateLift = 0;
    if (closestIndex >= 0) {
      if (distance >= 0) {
        const weight = Math.max(0, 1 - distance / ROTATE_SPREAD);
        targetRotateLift = weight * (cardStates[closestIndex]?.lift || 0);
      }
    }

    // 给rotateLift也加上缓动曲线
    let rotateStiffness;
    let rotateDamping;
    if (targetRotateLift > state.rotateLift) {
      // 正在抬起，快一点
      rotateStiffness = 0.2;
      rotateDamping = 0.88;
    } else {
      // 正在下落，慢一点，带曲线
      const t = state.rotateLift;
      const factor = Math.pow(1 - t, 2);
      rotateStiffness = 0.02 + factor * 0.1;
      rotateDamping = 0.95;
    }
    const rotateForce = rotateStiffness * (targetRotateLift - state.rotateLift);
    const rotateDampedVel = rotateDamping * state.rotateVelocity;
    state.rotateVelocity += rotateForce - rotateDampedVel;
    state.rotateLift += state.rotateVelocity;

    settleState(state);
    if (isStateActive(state)) {
      hasActiveMotion = true;
    }
  });

  if (!hasActiveMotion) {
    lastClosestIndex.value = closestIndex;
  }

  return hasActiveMotion;
};

const animate = () => {
  if (currentView.value === 'layered') {
    updatePhysics();

    for (let index = 0; index < cardsRef.length; index++) {
      const card = cardsRef[index];
      const state = cardStates[index];
      if (!card || !state) continue;

      const distance = closestIndex >= 0 ? index - closestIndex : Infinity;
      const wasNearPrevious = lastClosestIndex.value >= 0 && index >= lastClosestIndex.value && index < lastClosestIndex.value + ROTATE_SPREAD;
      const isNearCurrent = distance >= 0 && distance < ROTATE_SPREAD;
      const shouldRender =
        layeredNeedsFullRender ||
        index === closestIndex ||
        isNearCurrent ||
        wasNearPrevious ||
        isStateActive(state);

      if (!shouldRender) continue;

      const nextTransform = getLayeredTransform(index);
      if (nextTransform !== state.lastTransform) {
        card.style.transform = nextTransform;
        state.lastTransform = nextTransform;
      }
    }

    layeredNeedsFullRender = false;
  } else if (currentView.value === 'orbit') {
    for (let index = 0; index < cardsRef.length; index++) {
      const card = cardsRef[index];
      const state = cardStates[index];
      if (!card || !state) continue;

      const nextTransform = getOrbitTransform(index);
      if (nextTransform !== state.lastTransform) {
        card.style.transform = nextTransform;
        state.lastTransform = nextTransform;
      }
    }
  }

  animationId = requestAnimationFrame(animate);
};

// 事件处理
const updateGalleryBounds = () => {
  if (galleryRef.value) {
    galleryBounds = galleryRef.value.getBoundingClientRect();
  }
};

const handleMouseMove = (e) => {
  if (!galleryBounds) updateGalleryBounds();
  if (!galleryBounds) return;

  mouseGX = e.clientX - (galleryBounds.left + galleryBounds.width / 2);
  mouseGY = e.clientY - (galleryBounds.top + galleryBounds.height / 2);
  updateClosestIndex();
};

const handleMouseEnter = () => {
  isMouseInGallery = true;
  updateGalleryBounds();
  updateClosestIndex();
};

const handleMouseLeave = () => {
  isMouseInGallery = false;
  updateClosestIndex();
};

const setView = (view) => {
  currentView.value = view;
  updateClosestIndex();
  layeredNeedsFullRender = true;
  // 重置物理状态
  cardStates.forEach(s => {
    s.lift = 0; 
    s.velocity = 0; 
    s.rotateLift = 0;
    s.rotateVelocity = 0;
    s.lastTransform = '';
    s.lastZIndex = null;
  });
};

// 生命周期
onMounted(() => {
  // 初始化卡片状态
  cardStates.splice(0, cardStates.length, ...works.map(() => ({
    lift: 0,
    velocity: 0,
    rotateLift: 0,
    rotateVelocity: 0,
    lastTransform: '',
    lastZIndex: null
  })));
  updateGalleryBounds();
  window.addEventListener('resize', updateGalleryBounds);
  animate();
});

onUnmounted(() => {
  window.removeEventListener('resize', updateGalleryBounds);
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
});
</script>

<template>
  <div class="container">
    <aside class="sidebar">
      <h1 class="title">PARALLAX<br>ARCHIVE</h1>
      <p class="description">
        A living collection of independent visual studies. Explore twenty-four original works through layered, orbital, and archival perspectives.
      </p>
      <div class="divider"></div>
      <div class="view-selector">
        <p class="label">SELECT PERSPECTIVE</p>
        <button
          v-for="view in [
            { id: 'orbit', icon: '⟳', label: 'ORBIT VIEW' },
            { id: 'layered', icon: '◐', label: 'LAYERED VIEW' },
            { id: 'archive', icon: '▦', label: 'ARCHIVE VIEW' }
          ]"
          :key="view.id"
          :class="['view-btn', { active: currentView === view.id }]"
          @click="setView(view.id)"
        >
          <span class="icon">{{ view.icon }}</span>
          {{ view.label }}
        </button>
      </div>
      <div class="footer">
        <p>© 2026 • PARALLAX ARCHIVE</p>
      </div>
    </aside>

    <main class="gallery" ref="galleryRef" @mousemove="handleMouseMove" @mouseenter="handleMouseEnter" @mouseleave="handleMouseLeave">
      <div :class="['gallery-inner', `view-${currentView}`]">
        <div
          v-for="(work, index) in works"
          :key="work.id"
          class="card"
          :ref="el => { if (el) cardsRef[index] = el }"
        >
          <div class="card-spine" :style="{ background: work.color }"></div>
          <div class="card-inner">
            <div class="card-header">Parallax Archive</div>
            <div class="card-title">{{ work.title }}</div>
            <div class="card-image" :style="{ backgroundImage: `url(${work.image})` }"></div>
            <div class="card-footer">
              <span>{{ work.slug }}</span>
              <span>{{ work.year }}</span>
            </div>
          </div>
        </div>
      </div>
      <div class="hint">MOVE YOUR POINTER TO EXPLORE THE COLLECTION</div>
    </main>
  </div>
</template>
