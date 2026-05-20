<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>高保真流体视觉空间</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            user-select: none;
        }

        body, html {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #0d0202;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            color: #fff;
        }

        #canvas-container {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 1;
        }

        canvas {
            width: 100%;
            height: 100%;
            display: block;
        }

        /* 顶部调色质感微粒层，增强胶片颗粒感 */
        .film-grain {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 2;
            pointer-events: none;
            opacity: 0.04;
            background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)'/%3E%3C/svg%3E");
        }

        /* UI层：移除多余文字，仅保留底部控制器 */
        .ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 10;
            display: flex;
            flex-direction: column;
            justify-content: flex-end;
            padding: 35px 40px;
            pointer-events: none;
        }

        .ui-layer * {
            pointer-events: auto;
        }

        /* 底部交互控制台 */
        .footer {
            display: flex;
            justify-content: center;
            align-items: flex-end;
            width: 100%;
            position: relative;
        }

        /* 控制面板及扫描框的过渡动画设定 */
        .control-panel, .scan-frame {
            transition: all 0.5s cubic-bezier(0.16, 1, 0.3, 1);
        }

        /* 隐藏状态样式 (由 JS 切换 body 的 class 触发) */
        body.ui-hidden .control-panel,
        body.ui-hidden .scan-frame {
            transform: translateY(120px) !important;
            opacity: 0;
            pointer-events: none;
        }

        body.ui-hidden .toast-hint {
            opacity: 0;
            transform: translateX(-50%) translateY(-20px);
            pointer-events: none;
        }

        /* 高级毛玻璃悬浮交互面板 */
        .control-panel {
            background: rgba(18, 3, 2, 0.45);
            border: 1px solid rgba(255, 70, 0, 0.25);
            backdrop-filter: blur(25px);
            -webkit-backdrop-filter: blur(25px);
            padding: 12px 28px;
            border-radius: 100px;
            display: flex;
            align-items: center;
            gap: 22px;
            box-shadow: 0 15px 50px rgba(0, 0, 0, 0.65), inset 0 1px 3px rgba(255,255,255,0.1);
            width: 100%;
            max-width: 480px;
            transform: translateY(5px);
        }

        .control-panel:hover {
            border-color: rgba(255, 90, 0, 0.45);
            box-shadow: 0 15px 50px rgba(255, 68, 0, 0.15), 0 20px 60px rgba(0,0,0,0.7);
        }

        .btn-play {
            background: #ff3c00;
            border: none;
            width: 38px;
            height: 38px;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 4px 15px rgba(255, 60, 0, 0.4);
            flex-shrink: 0;
        }

        .btn-play:hover {
            transform: scale(1.08);
            background: #ff5500;
            box-shadow: 0 6px 20px rgba(255, 60, 0, 0.6);
        }

        .btn-play svg {
            width: 14px;
            height: 14px;
            fill: #fff;
        }

        .status-container {
            flex-grow: 1;
            display: flex;
            flex-direction: column;
            gap: 2px;
            overflow: hidden;
        }

        .status-title {
            font-size: 13px;
            font-weight: 600;
            color: #ffffff;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }

        .status-meta {
            font-size: 10px;
            color: rgba(255, 255, 255, 0.45);
            display: flex;
            justify-content: space-between;
            letter-spacing: 1px;
        }

        #audio-file {
            display: none;
        }

        .btn-upload {
            background: rgba(255, 255, 255, 0.08);
            border: 1px solid rgba(255, 255, 255, 0.15);
            padding: 7px 15px;
            border-radius: 20px;
            font-size: 11px;
            font-weight: 700;
            color: #fff;
            cursor: pointer;
            transition: all 0.25s;
            white-space: nowrap;
            letter-spacing: 1px;
        }

        .btn-upload:hover {
            background: rgba(255, 68, 0, 0.2);
            border-color: rgba(255, 68, 0, 0.6);
        }

        .scan-frame {
            width: 40px;
            height: 40px;
            border: 1px solid rgba(255,255,255,0.25);
            border-radius: 10px;
            display: flex;
            align-items: center;
            justify-content: center;
            opacity: 0.8;
            transition: all 0.5s cubic-bezier(0.16, 1, 0.3, 1);
            position: absolute;
            right: 0;
            bottom: 0;
        }
        .scan-frame:hover {
            border-color: #ff3c00;
            opacity: 1;
        }

        /* 顶部中央动态提示框 */
        .toast-hint {
            position: absolute;
            top: 30px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(255, 60, 0, 0.15);
            border: 1px solid rgba(255, 60, 0, 0.35);
            padding: 8px 18px;
            border-radius: 6px;
            font-size: 11px;
            font-weight: 600;
            letter-spacing: 2px;
            animation: pulseWave 2.5s infinite ease-in-out;
            pointer-events: none;
            z-index: 100;
            box-shadow: 0 10px 30px rgba(0,0,0,0.4);
            transition: all 0.5s ease;
        }

        @keyframes pulseWave {
            0% { opacity: 0.7; transform: translateX(-50%) scale(1); }
            50% { opacity: 1; transform: translateX(-50%) scale(1.03); background: rgba(255, 60, 0, 0.25); }
            100% { opacity: 0.7; transform: translateX(-50%) scale(1); }
        }

        /* 右上角 UI 显隐切换按钮 */
        .btn-toggle-ui {
            position: absolute;
            top: 30px;
            right: 40px;
            background: rgba(18, 3, 2, 0.45);
            border: 1px solid rgba(255, 70, 0, 0.25);
            backdrop-filter: blur(15px);
            border-radius: 50%;
            width: 42px;
            height: 42px;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
            color: rgba(255, 255, 255, 0.7);
            z-index: 100;
        }

        .btn-toggle-ui:hover {
            border-color: rgba(255, 90, 0, 0.6);
            color: #ff3c00;
            box-shadow: 0 0 20px rgba(255, 60, 0, 0.3);
            transform: scale(1.05);
        }

        .btn-toggle-ui svg {
            width: 20px;
            height: 20px;
            stroke: currentColor;
            fill: none;
            stroke-width: 2;
            stroke-linecap: round;
            stroke-linejoin: round;
        }
    </style>
</head>
<body>

    <div class="toast-hint" id="guide-hint">请点击下方“载入音频”激活流体反应</div>

    <button class="btn-toggle-ui" id="btn-toggle-ui" title="隐藏/显示沉浸界面">
        <svg id="icon-eye-open" viewBox="0 0 24 24">
            <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"></path>
            <circle cx="12" cy="12" r="3"></circle>
        </svg>
        <svg id="icon-eye-closed" viewBox="0 0 24 24" style="display: none;">
            <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19m-6.72-1.07a3 3 0 1 1-4.24-4.24"></path>
            <line x1="1" y1="1" x2="23" y2="23"></line>
        </svg>
    </button>

    <div class="film-grain"></div>

    <div id="canvas-container">
        <canvas id="gl-canvas"></canvas>
    </div>

    <div class="ui-layer">
        <div class="footer">
            <div class="control-panel">
                <button class="btn-play" id="btn-toggle" title="播放/暂停">
                    <svg id="icon-play" viewBox="0 0 24 24"><path d="M8 5v14l11-7z"/></svg>
                    <svg id="icon-pause" viewBox="0 0 24 24" style="display:none;"><path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg>
                </button>
                
                <div class="status-container">
                    <div class="status-title" id="audio-title">未加载外部音轨</div>
                    <div class="status-meta">
                        <span id="engine-status">待机中</span>
                        <span id="realtime-freq">0.00</span>
                    </div>
                </div>

                <label for="audio-file" class="btn-upload">载入音频</label>
                <input type="file" id="audio-file" accept="audio/*">
            </div>

            <div class="scan-frame">
                <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.8)" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                    <path d="M3 7V5a2 2 0 0 1 2-2h2"></path>
                    <path d="M17 3h2a2 2 0 0 1 2 2v2"></path>
                    <path d="M21 17v2a2 2 0 0 1-2 2h-2"></path>
                    <path d="M7 21H5a2 2 0 0 1-2-2v-2"></path>
                    <circle cx="12" cy="12" r="3"></circle>
                </svg>
            </div>
        </div>
    </div>

    <script>
        // --- UI 显隐逻辑 ---
        const btnToggleUi = document.getElementById('btn-toggle-ui');
        const iconEyeOpen = document.getElementById('icon-eye-open');
        const iconEyeClosed = document.getElementById('icon-eye-closed');

        btnToggleUi.addEventListener('click', () => {
            document.body.classList.toggle('ui-hidden');
            if (document.body.classList.contains('ui-hidden')) {
                iconEyeOpen.style.display = 'none';
                iconEyeClosed.style.display = 'block';
            } else {
                iconEyeOpen.style.display = 'block';
                iconEyeClosed.style.display = 'none';
            }
        });

        // --- WebGL 核心逻辑 ---
        const canvas = document.getElementById('gl-canvas');
        const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

        if (!gl) {
            alert('您的浏览器不支持 WebGL，请更换现代浏览器以获得绝佳流体效果。');
        }

        const vsSource = `
            attribute vec2 position;
            void main() {
                gl_Position = vec4(position, 0.0, 1.0);
            }
        `;

        const fsSource = `
            precision highp float;

            uniform vec2 u_resolution;
            uniform float u_time;
            uniform vec2 u_mouse;
            uniform float u_audio_intensity;

            float hash(vec2 p) {
                p = fract(p * vec2(443.8975, 397.2973));
                p += dot(p.xy, p.yx + 19.19);
                return fract(p.x * p.y);
            }

            float noise(in vec2 p) {
                vec2 i = floor(p);
                vec2 f = fract(p);
                vec2 u = f*f*(3.0-2.0*f);
                return mix(mix(hash(i + vec2(0.0,0.0)), hash(i + vec2(1.0,0.0)), u.x),
                           mix(hash(i + vec2(0.0,1.0)), hash(i + vec2(1.0,1.0)), u.x), u.y);
            }

            float fbm(in vec2 p) {
                float v = 0.0;
                float a = 0.5;
                vec2 shift = vec2(100.0);
                mat2 rot = mat2(cos(0.5), sin(0.5), -sin(0.5), cos(0.50));
                for (int i = 0; i < 5; ++i) {
                    v += a * noise(p);
                    p = rot * p * 2.0 + shift;
                    a *= 0.55;
                }
                return v;
            }

            float cellDroplet(vec2 uv, vec2 center, float radius, float blur) {
                float d = length(uv - center);
                return smoothstep(radius + blur, radius - blur, d);
            }

            void main() {
                vec2 uv = gl_FragCoord.xy / u_resolution.xy;
                vec2 p = (gl_FragCoord.xy * 2.0 - u_resolution.xy) / min(u_resolution.x, u_resolution.y);
                p *= 1.35;

                float audioAmp = u_audio_intensity * 0.45;
                
                vec2 q = vec2(
                    fbm(p + vec2(0.0, 0.0) + u_time * 0.06),
                    fbm(p + vec2(5.2, 1.3) + u_time * 0.04)
                );
                
                vec2 r = vec2(
                    fbm(p + 4.0 * q + vec2(1.7, 9.2) + u_time * 0.09 + vec2(audioAmp * 0.5)),
                    fbm(p + 4.0 * q + vec2(8.3, 2.8) + u_time * 0.05 + vec2(audioAmp * 0.3))
                );

                float f = fbm(p + 3.8 * r);
                f = smoothstep(0.15, 0.85, f);

                float cells = 0.0;
                for(int i = 0; i < 8; i++) {
                    float fi = float(i);
                    vec2 c_pos = vec2(
                        sin(u_time * 0.2 + fi * 1.4) * 1.5 + cos(u_time * 0.1 + fi * 0.8) * 0.3,
                        cos(u_time * 0.15 + fi * 0.9) * 1.0 + sin(u_time * 0.3 + fi * 1.2) * 0.2
                    );
                    c_pos += r * 0.18;
                    float r_size = (0.06 + sin(fi * 3.3) * 0.03) * (1.0 + audioAmp * 1.2);
                    cells += cellDroplet(p, c_pos, r_size, 0.08);
                }
                
                f = mix(f, 1.0, clamp(cells, 0.0, 1.0));

                vec3 col_bg     = vec3(0.047, 0.008, 0.008); 
                vec3 col_dark   = vec3(0.32,  0.03,  0.02);  
                vec3 col_mid    = vec3(1.0,   0.20,  0.0);   
                vec3 col_bright = vec3(1.0,   0.72,  0.0);   
                vec3 col_core   = vec3(1.0,   0.98,  0.2);   

                vec3 color = col_bg;
                color = mix(color, col_dark, smoothstep(0.05, 0.35, f));
                color = mix(color, col_mid, smoothstep(0.30, 0.58, f));
                color = mix(color, col_bright, smoothstep(0.52, 0.76, f));
                color = mix(color, col_core, smoothstep(0.72, 0.95, f));

                color += vec3(1.0, 0.35, 0.0) * clamp(r.x * r.x * 0.12, 0.0, 1.0) * smoothstep(0.2, 0.8, f);
                
                float centerGlow = 1.0 - length(uv - vec2(0.65, 0.45));
                color += vec3(1.0, 0.45, 0.0) * pow(max(centerGlow, 0.0), 3.5) * 0.28;

                gl_FragColor = vec4(color, 1.0);
            }
        `;

        function createShader(gl, type, source) {
            const shader = gl.createShader(type);
            gl.shaderSource(shader, source);
            gl.compileShader(shader);
            if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) return null;
            return shader;
        }

        const vertexShader = createShader(gl, gl.VERTEX_SHADER, vsSource);
        const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fsSource);

        const program = gl.createProgram();
        gl.attachShader(program, vertexShader);
        gl.attachShader(program, fragmentShader);
        gl.linkProgram(program);
        gl.useProgram(program);

        const positionBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
            -1.0, -1.0, 1.0, -1.0, -1.0,  1.0,
            -1.0,  1.0, 1.0, -1.0, 1.0,  1.0,
        ]), gl.STATIC_DRAW);

        const positionLocation = gl.getAttribLocation(program, 'position');
        gl.enableVertexAttribArray(positionLocation);
        gl.vertexAttribPointer(positionLocation, 2, gl.FLOAT, false, 0, 0);

        const resLocation = gl.getUniformLocation(program, 'u_resolution');
        const timeLocation = gl.getUniformLocation(program, 'u_time');
        const mouseLocation = gl.getUniformLocation(program, 'u_mouse');
        const audioLocation = gl.getUniformLocation(program, 'u_audio_intensity');

        let mouseX = 0, mouseY = 0;
        window.addEventListener('mousemove', (e) => { mouseX = e.clientX; mouseY = e.clientY; });

        function resizeCanvas() {
            const dpr = Math.min(window.devicePixelRatio || 1, 2); 
            canvas.width = window.innerWidth * dpr;
            canvas.height = window.innerHeight * dpr;
            gl.viewport(0, 0, canvas.width, canvas.height);
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        const audioFile = document.getElementById('audio-file');
        const btnToggle = document.getElementById('btn-toggle');
        const iconPlay = document.getElementById('icon-play');
        const iconPause = document.getElementById('icon-pause');
        const audioTitle = document.getElementById('audio-title');
        const engineStatus = document.getElementById('engine-status');
        const realtimeFreq = document.getElementById('realtime-freq');
        const guideHint = document.getElementById('guide-hint');

        let audioCtx, analyser, audioSource, dataArray;
        let isPlaying = false;
        let audioUrl = null;
        let audioElement = new Audio();
        let targetIntensity = 0.0;
        let currentIntensity = 0.0;

        function initAudioEngine() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                analyser = audioCtx.createAnalyser();
                analyser.fftSize = 128; 
                const bufferLength = analyser.frequencyBinCount;
                dataArray = new Uint8Array(bufferLength);
                audioSource = audioCtx.createMediaElementSource(audioElement);
                audioSource.connect(analyser);
                analyser.connect(audioCtx.destination);
            }
        }

        audioFile.addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (!file) return;

            if (audioUrl) URL.revokeObjectURL(audioUrl);
            guideHint.style.display = 'none';

            audioUrl = URL.createObjectURL(file);
            audioTitle.innerText = file.name;
            engineStatus.innerText = "解析中";

            audioElement.src = audioUrl;
            audioElement.load();
            
            initAudioEngine();
            playAudio();
        });

        function playAudio() {
            if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume();
            audioElement.play();
            isPlaying = true;
            iconPlay.style.display = 'none';
            iconPause.style.display = 'block';
            engineStatus.innerText = "正在律动";
        }

        function pauseAudio() {
            audioElement.pause();
            isPlaying = false;
            iconPlay.style.display = 'block';
            iconPause.style.display = 'none';
            engineStatus.innerText = "已暂停";
        }

        btnToggle.addEventListener('click', () => {
            if (!audioElement.src) {
                alert("请先载入音频文件！");
                return;
            }
            if (isPlaying) pauseAudio(); else playAudio();
        });

        audioElement.addEventListener('ended', () => {
            pauseAudio();
            engineStatus.innerText = "播放结束";
        });

        function render(now) {
            now *= 0.001; 
            
            if (isPlaying && analyser) {
                analyser.getByteFrequencyData(dataArray);
                let energySum = 0;
                let sampleCount = 18;
                for(let i = 0; i < sampleCount; i++) energySum += dataArray[i];
                targetIntensity = energySum / (sampleCount * 255.0);
                realtimeFreq.innerText = (targetIntensity * 10).toFixed(2);
            } else {
                targetIntensity = 0.0;
                realtimeFreq.innerText = "0.00";
            }

            currentIntensity += (targetIntensity - currentIntensity) * 0.12;

            gl.uniform2f(resLocation, canvas.width, canvas.height);
            gl.uniform1f(timeLocation, now);
            gl.uniform2f(mouseLocation, mouseX, mouseY);
            gl.uniform1f(audioLocation, currentIntensity);

            gl.drawArrays(gl.TRIANGLES, 0, 6);
            requestAnimationFrame(render);
        }

        requestAnimationFrame(render);
    </script>
</body>
</html>
