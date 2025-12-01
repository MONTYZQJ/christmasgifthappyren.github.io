<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>清晰版 3D 旋转粒子圣诞树与爱心</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #000; /* 纯黑背景 */
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        canvas {
            position: absolute;
            top: 0;
            left: 0;
        }
        #instruction {
            position: absolute;
            bottom: 20px;
            color: rgba(255, 255, 255, 0.6);
            font-family: sans-serif;
            font-size: 14px;
            pointer-events: none;
            text-align: center;
            width: 100%;
            text-shadow: 0 0 5px #000;
        }
    </style>
</head>
<body>

<canvas id="particleCanvas"></canvas>
<div id="instruction">按空格键切换：树形 → 心形 → 文字+烟花 (Press SPACE to cycle: Tree → Heart → Text+Fireworks)</div>

<script>
    const canvas = document.getElementById('particleCanvas');
    const ctx = canvas.getContext('2d');

    let width, height, cx, cy;
    let particles = [];
    let displayMode = 0; // 0: 树形, 1: 心形, 2: 文字+烟花
    let rotationAngle = 0;

    // --- 配置参数 ---
    // 建议：如果觉得心形还不够清晰，可以稍微增加粒子数量，例如到 3000
    const PARTICLE_COUNT = 2500; 
    const EASING = 0.05;
    const ROTATION_SPEED = 0.005;
    const FOCAL_LENGTH = 400;

    // 颜色配置
    const TREE_LEAF_COLORS = ['#0f0', '#00ff00', '#adff2f', '#ffd700', '#ffffff', '#32CD32'];
    const TREE_TRUNK_COLOR = '#8B4513';
    const HEART_COLOR = '#ff69b4';
    const FIREWORK_COLORS = ['#ff69b4', '#ff1493', '#ff00ff', '#ffd700', '#ff6347', '#00ffff', '#ffff00', '#ff00ff', '#00ff00'];

    function resize() {
        width = canvas.width = window.innerWidth;
        height = canvas.height = window.innerHeight;
        cx = width / 2;
        cy = height / 2;
    }
    window.addEventListener('resize', () => {
        resize();
        // 重新生成文字数据（如果已经初始化）
        if (typeof createTextPositions !== 'undefined' && textData) {
            textData = createTextPositions('熊二❤️欢乐马');
            // 如果当前是心形+文字模式，更新文字粒子
            if (displayMode === 2 && textParticles.length > 0) {
                textParticles = [];
                textData.forEach(pos => {
                    const p = new Particle(false, true);
                    p.targetX = pos.x;
                    p.targetY = pos.y;
                    p.targetZ = pos.z;
                    p.x = (Math.random() - 0.5) * width;
                    p.y = (Math.random() - 0.5) * height;
                    p.z = 0;
                    p.heartColorHex = HEART_COLOR;
                    p.targetRgb = hexToRgb(HEART_COLOR);
                    p.currentRgb = hexToRgb(HEART_COLOR);
                    p.colorString = HEART_COLOR;
                    textParticles.push(p);
                });
            }
        }
    });
    resize();

    // --- 辅助工具 ---
    function hexToRgb(hex) {
        let expandedHex = hex;
        if (hex.length === 4) {
            expandedHex = "#" + hex[1] + hex[1] + hex[2] + hex[2] + hex[3] + hex[3];
        }
        const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(expandedHex);
        return result ? {
            r: parseInt(result[1], 16),
            g: parseInt(result[2], 16),
            b: parseInt(result[3], 16)
        } : { r: 255, g: 255, b: 255 };
    }

    // --- 粒子类 ---
    class Particle {
        constructor(isTrunk = false, isText = false) {
            this.x = 0; this.y = 0; this.z = 0;
            this.targetX = 0; this.targetY = 0; this.targetZ = 0;
            this.rx = 0; this.ry = 0; this.rz = 0;
            this.baseSize = Math.random() * 2 + 1;
            this.isTrunk = isTrunk;
            this.isText = isText; // 标记是否为文字粒子
            this.leafColorHex = TREE_LEAF_COLORS[Math.floor(Math.random() * TREE_LEAF_COLORS.length)];
            this.trunkColorHex = TREE_TRUNK_COLOR;
            this.heartColorHex = HEART_COLOR;
            const startHex = this.isTrunk ? this.trunkColorHex : this.leafColorHex;
            this.currentRgb = hexToRgb(startHex);
            this.targetRgb = { ...this.currentRgb };
            this.colorString = startHex;
        }

        update() {
            this.x += (this.targetX - this.x) * EASING;
            this.y += (this.targetY - this.y) * EASING;
            this.z += (this.targetZ - this.z) * EASING;

            this.currentRgb.r += (this.targetRgb.r - this.currentRgb.r) * EASING;
            this.currentRgb.g += (this.targetRgb.g - this.currentRgb.g) * EASING;
            this.currentRgb.b += (this.targetRgb.b - this.currentRgb.b) * EASING;
            this.colorString = `rgb(${Math.round(this.currentRgb.r)}, ${Math.round(this.currentRgb.g)}, ${Math.round(this.currentRgb.b)})`;

            // 文字粒子不旋转
            if (this.isText) {
                this.rx = this.x;
                this.ry = this.y;
                this.rz = this.z;
            } else {
                // 3D 旋转
                const cos = Math.cos(rotationAngle);
                const sin = Math.sin(rotationAngle);
                this.rx = this.x * cos - this.z * sin;
                this.ry = this.y;
                this.rz = this.x * sin + this.z * cos;
            }
        }

        draw() {
            const scale = FOCAL_LENGTH / (FOCAL_LENGTH + this.rz);
            if (scale <= 0) return;
            const px = cx + this.rx * scale;
            const py = cy + this.ry * scale;
            const currentSize = this.baseSize * scale;

            ctx.beginPath();
            ctx.arc(px, py, currentSize, 0, Math.PI * 2);
            ctx.shadowBlur = 15 * scale; 
            ctx.shadowColor = this.colorString;
            ctx.fillStyle = this.colorString;
            ctx.fill();
        }
    }

    // --- 烟花粒子类 ---
    class FireworkParticle {
        constructor(x, y) {
            this.x = x;
            this.y = y;
            this.vx = (Math.random() - 0.5) * 10;
            this.vy = (Math.random() - 0.5) * 10;
            this.life = 1.0;
            this.decay = Math.random() * 0.02 + 0.01;
            this.size = Math.random() * 3 + 2;
            this.color = FIREWORK_COLORS[Math.floor(Math.random() * FIREWORK_COLORS.length)];
            this.rgb = hexToRgb(this.color);
        }

        update() {
            this.x += this.vx;
            this.y += this.vy;
            this.vy += 0.1; // 重力
            this.vx *= 0.98; // 阻力
            this.vy *= 0.98;
            this.life -= this.decay;
        }

        draw() {
            if (this.life <= 0) return;
            const alpha = this.life;
            ctx.globalAlpha = alpha;
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
            ctx.shadowBlur = 20;
            ctx.shadowColor = this.color;
            ctx.fillStyle = this.color;
            ctx.fill();
            ctx.globalAlpha = 1.0;
        }
    }

    // --- 3D 形状生成函数 ---

    function createTreePositions3D() {
        const positions = [];
        const treeHeight = Math.min(width, height) * 0.6;
        const treeBaseRadius = treeHeight * 0.3;
        const trunkHeight = treeHeight * 0.2;
        const trunkRadius = treeBaseRadius * 0.15;
        const trunkParticleCount = Math.floor(PARTICLE_COUNT * 0.15);
        const leafParticleCount = PARTICLE_COUNT - trunkParticleCount;

        for (let i = 0; i < trunkParticleCount; i++) {
            const h = Math.random();
            const angle = Math.random() * Math.PI * 2;
            const r = Math.sqrt(Math.random()) * trunkRadius;
            const ty = (h * trunkHeight) + (treeHeight / 2) - trunkHeight;
            const tx = r * Math.cos(angle);
            const tz = r * Math.sin(angle);
            positions.push({ x: tx, y: ty, z: tz, isTrunk: true });
        }

        for (let i = 0; i < leafParticleCount; i++) {
            const h = Math.random();
            const currentMaxRadius = h * treeBaseRadius;
            const angle = Math.random() * Math.PI * 2;
            const rSpread = Math.random() > 0.8 ? Math.sqrt(Math.random()) : Math.random() * 0.2 + 0.8;
            const r = rSpread * currentMaxRadius;
            const ty = (h * treeHeight) - (treeHeight / 2) - trunkHeight/2;
            const tx = r * Math.cos(angle);
            const tz = r * Math.sin(angle);
            positions.push({ x: tx, y: ty, z: tz, isTrunk: false });
        }
        return positions;
    }

    // --- 核心修改：改进的心形生成函数 ---
    function createHeartPositions3D() {
        const positions = [];
        // 减小尺寸，让心形更小但更清晰
        const scale = Math.min(width, height) * 0.02;

        for (let i = 0; i < PARTICLE_COUNT; i++) {
            let t = Math.random() * Math.PI * 2;
            let p = Math.random() * Math.PI;

            // 改进的 3D 心形表面方程，让心形特征更明显
            // 使用更标准的心形参数方程
            let x = 16 * Math.pow(Math.sin(t), 3) * Math.sin(p);
            // 调整Y轴系数，让心形的两个"半圆"和底部的"尖"更明显
            let y = -(13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t));
            let z = 16 * Math.pow(Math.sin(t), 3) * Math.cos(p);

            // 改进的内部填充策略 - 让表面更密集，心形轮廓更清晰
            // 使用更小的指数值，让粒子更集中在表面
            const internalScale = Math.pow(Math.random(), 0.15);
            
            x *= internalScale;
            y *= internalScale;
            z *= internalScale;
            
            positions.push({ 
                x: x * scale, 
                y: y * scale, 
                z: z * scale, 
                isTrunk: false
            });
        }
        return positions;
    }

    // --- 文字粒子生成函数 ---
    function createTextPositions(text) {
        try {
            const positions = [];
            const offCanvas = document.createElement('canvas');
            const offCtx = offCanvas.getContext('2d');
            // 增大字体，让文字更明显
            const fontSize = Math.min(width, height) * 0.2;
            const textY = height * 0.5; // 文字位置在屏幕中央
            
            if (!offCtx) {
                console.error('无法创建离屏canvas上下文');
                return positions;
            }
            
            offCanvas.width = width;
            offCanvas.height = height;
            
            // 清除画布
            offCtx.clearRect(0, 0, width, height);
            
            offCtx.fillStyle = '#ffffff';
            // 使用支持中文的字体，加粗显示
            offCtx.font = `bold ${fontSize}px "Microsoft YaHei", "SimHei", "Arial Unicode MS", Arial, sans-serif`;
            offCtx.textAlign = 'center';
            offCtx.textBaseline = 'middle';
            offCtx.fillText(text, width / 2, textY);
            
            const imageData = offCtx.getImageData(0, 0, width, height);
            const data = imageData.data;
            const step = 2; // 减小采样步长，让文字更密集清晰
            
            for (let y = 0; y < height; y += step) {
                for (let x = 0; x < width; x += step) {
                    const index = (y * width + x) * 4;
                    if (data[index + 3] > 128) { // 如果像素不透明
                        const px = (x - width / 2);
                        const py = (y - height / 2);
                        positions.push({ 
                            x: px, 
                            y: py, 
                            z: 0, // 文字在z=0平面，不旋转
                            isTrunk: false 
                        });
                    }
                }
            }
            
            return positions;
        } catch (error) {
            console.error('文字生成错误:', error);
            return [];
        }
    }

    // --- 初始化与动画 ---

    let treeData, heartData, textData;
    let textParticles = [];
    let fireworkParticles = [];

    function init() {
        try {
            particles = [];
            textParticles = [];
            fireworkParticles = [];
            resize();
            
            if (!width || !height) {
                console.error('Canvas尺寸无效');
                return;
            }
            
            treeData = createTreePositions3D();
            heartData = createHeartPositions3D();
            textData = createTextPositions('熊二❤️欢乐马');

            treeData.forEach(pos => {
                const p = new Particle(pos.isTrunk);
                p.targetX = pos.x; p.targetY = pos.y; p.targetZ = pos.z;
                p.x = (Math.random() - 0.5) * width;
                p.y = (Math.random() - 0.5) * height;
                p.z = (Math.random() - 0.5) * width;
                particles.push(p);
            });
        } catch (error) {
            console.error('初始化错误:', error);
        }
    }

    function animate() {
        if (!width || !height) {
            requestAnimationFrame(animate);
            return;
        }
        
        ctx.fillStyle = 'rgba(0, 0, 0, 0.2)';
        ctx.fillRect(0, 0, width, height);

        rotationAngle += ROTATION_SPEED;

        if (particles.length > 0) {
            particles.forEach(p => p.update());
            particles.sort((a, b) => b.rz - a.rz);

            ctx.globalCompositeOperation = 'lighter';
            // 在文字+烟花模式下，隐藏心形粒子（不绘制）
            if (displayMode !== 2) {
                particles.forEach(p => p.draw());
            }
        }
        
        // 如果是文字+烟花模式，绘制文字粒子和烟花
        if (displayMode === 2) {
            if (textParticles.length > 0) {
                textParticles.forEach(p => {
                    p.update();
                    p.draw();
                });
            }
            
            // 更新和绘制烟花粒子
            fireworkParticles = fireworkParticles.filter(fw => {
                fw.update();
                if (fw.life > 0) {
                    fw.draw();
                    return true;
                }
                return false;
            });
            
            // 持续生成新烟花
            if (Math.random() < 0.3) {
                const fireworkX = cx + (Math.random() - 0.5) * width * 0.8;
                const fireworkY = cy + (Math.random() - 0.5) * height * 0.8;
                for (let i = 0; i < 30; i++) {
                    fireworkParticles.push(new FireworkParticle(fireworkX, fireworkY));
                }
            }
        }
        
        ctx.globalCompositeOperation = 'source-over';

        requestAnimationFrame(animate);
    }

    window.addEventListener('keydown', (e) => {
        if (e.code === 'Space') {
            // 循环切换：树形 -> 心形 -> 文字+烟花 -> 树形
            displayMode = (displayMode + 1) % 3;
            
            if (displayMode === 0) {
                // 树形模式
                const targetData = treeData;
                particles.forEach((p, i) => {
                    p.targetX = targetData[i].x;
                    p.targetY = targetData[i].y;
                    p.targetZ = targetData[i].z;
                    p.targetRgb = hexToRgb(p.isTrunk ? p.trunkColorHex : p.leafColorHex);
                });
                textParticles = [];
                fireworkParticles = [];
            } else if (displayMode === 1) {
                // 心形模式
                const targetData = heartData;
                particles.forEach((p, i) => {
                    p.targetX = targetData[i].x;
                    p.targetY = targetData[i].y;
                    p.targetZ = targetData[i].z;
                    p.targetRgb = hexToRgb(p.heartColorHex);
                });
                textParticles = [];
                fireworkParticles = [];
            } else if (displayMode === 2) {
                // 文字+烟花模式：隐藏心形，显示文字和烟花
                // 将心形粒子移动到屏幕外（隐藏）
                particles.forEach((p) => {
                    p.targetX = width * 10; // 移到屏幕外
                    p.targetY = height * 10;
                    p.targetZ = 0;
                });
                
                // 创建文字粒子
                textParticles = [];
                textData.forEach(pos => {
                    const p = new Particle(false, true);
                    p.targetX = pos.x;
                    p.targetY = pos.y;
                    p.targetZ = pos.z;
                    p.x = (Math.random() - 0.5) * width;
                    p.y = (Math.random() - 0.5) * height;
                    p.z = 0;
                    p.heartColorHex = HEART_COLOR;
                    p.targetRgb = hexToRgb(HEART_COLOR);
                    p.currentRgb = hexToRgb(HEART_COLOR);
                    p.colorString = HEART_COLOR;
                    textParticles.push(p);
                });
                
                // 创建初始烟花爆炸
                fireworkParticles = [];
                for (let i = 0; i < 5; i++) {
                    const fireworkX = cx + (Math.random() - 0.5) * width * 0.6;
                    const fireworkY = cy + (Math.random() - 0.5) * height * 0.6;
                    for (let j = 0; j < 50; j++) {
                        fireworkParticles.push(new FireworkParticle(fireworkX, fireworkY));
                    }
                }
            }
        }
    });

    // 确保页面加载完成后再初始化
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => {
            init();
            animate();
        });
    } else {
        init();
        animate();
    }

</script>
</body>
</html>
