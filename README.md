<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <title>آزمایشگاه شیمی - تولید ELC</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700&display=swap');

        body {
            margin: 0;
            background-color: #020205;
            font-family: 'Tahoma', 'Segoe UI', sans-serif;
            color: white;
            overflow: hidden;
        }

        #ui-container {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            padding: 30px;
            box-sizing: border-box;
            z-index: 10;
        }

        .header {
            text-align: center;
            text-shadow: 0 0 10px #3498db;
        }

        .header h1 {
            font-family: 'Orbitron', sans-serif;
            margin: 0;
            letter-spacing: 5px;
            font-size: 24px;
            color: #3498db;
        }

        .shelf {
            pointer-events: auto;
            display: flex;
            gap: 15px;
            background: rgba(10, 10, 15, 0.7);
            padding: 20px;
            border-radius: 20px;
            border: 1px solid rgba(52, 152, 219, 0.3);
            backdrop-filter: blur(10px);
            width: fit-content;
            box-shadow: 0 0 30px rgba(0,0,0,0.5);
        }

        .chemical-item {
            background: rgba(40, 40, 50, 0.5);
            padding: 12px 20px;
            border-radius: 12px;
            cursor: pointer;
            transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            border: 1px solid rgba(255,255,255,0.1);
            text-align: center;
            font-weight: bold;
            min-width: 100px;
        }

        .chemical-item:hover {
            background: #3498db;
            color: black;
            box-shadow: 0 0 20px #3498db;
            transform: translateY(-10px) scale(1.05);
        }

        #progress-container {
            display: none;
            pointer-events: auto;
            width: 400px;
            background: rgba(255, 255, 255, 0.1);
            height: 10px;
            border-radius: 5px;
            margin: 20px auto;
            overflow: hidden;
            border: 1px solid rgba(52, 152, 219, 0.5);
        }

        #progress-bar {
            width: 0%;
            height: 100%;
            background: linear-gradient(90deg, #3498db, #2ecc71);
            box-shadow: 0 0 10px #3498db;
            transition: width 0.1s linear;
        }

        .controls {
            pointer-events: auto;
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 10px;
            margin-bottom: 20px;
        }

        .btn-group {
            display: flex;
            gap: 20px;
        }

        .btn {
            padding: 15px 40px;
            border-radius: 12px;
            border: none;
            cursor: pointer;
            font-weight: bold;
            font-size: 18px;
            font-family: 'Orbitron', sans-serif;
            text-transform: uppercase;
            transition: 0.3s;
        }

        .btn-mix { 
            background: linear-gradient(45deg, #2ecc71, #27ae60); 
            color: white;
            box-shadow: 0 0 15px rgba(46, 204, 113, 0.4);
        }
        .btn-reset { 
            background: linear-gradient(45deg, #e74c3c, #c0392b); 
            color: white;
        }
        .btn:hover:not(:disabled) { transform: scale(1.1); box-shadow: 0 0 25px currentColor; }
        .btn:disabled { opacity: 0.3; cursor: not-allowed; }

        #log-box {
            position: absolute;
            bottom: 120px;
            right: 30px;
            width: 320px;
            background: rgba(0, 0, 0, 0.6);
            backdrop-filter: blur(5px);
            padding: 15px;
            border-radius: 12px;
            border-left: 4px solid #3498db;
            font-family: 'Courier New', Courier, monospace;
            font-size: 13px;
            color: #3498db;
            max-height: 200px;
            overflow-y: auto;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
        }

        canvas { display: block; position: fixed; top: 0; left: 0; z-index: 1; }
    </style>
</head>
<body>

<div id="ui-container">
    <div class="shelf">
        <div class="chemical-item" onclick="pourChemical('متیل‌آمین', 0x3498db)">🧪 متیل‌آمین</div>
        <div class="chemical-item" onclick="pourChemical('اسید کلرید', 0xe74c3c)">🧪 اسید کلرید</div>
        <div class="chemical-item" onclick="pourChemical('سود سوزآور', 0xf1c40f)">🧪 سود سوزآور</div>
        <div class="chemical-item" onclick="pourChemical('فسفر قرمز', 0x9b59b6)">🧪 فسفر قرمز</div>
    </div>

    <div id="log-box">> سیستم آنلاین است. منتظر مواد اولیه...</div>

    <div class="controls">
        <div id="progress-container">
            <div id="progress-bar"></div>
        </div>
        <div class="btn-group">
            <button class="btn btn-mix" id="mixBtn" onclick="processCraft()" disabled>ترکیب نهایی</button>
            <button class="btn btn-reset" id="resetBtn" onclick="resetLab()">تخلیه مخزن</button>
        </div>
    </div>
</div>

<script>
    let scene, camera, renderer, beaker, liquid, particles;
    let currentIngredients = [];
    let isPouring = false;
    let isProcessing = false;

    // لیست دستورالعمل‌ها - نتیجه اول به ELC تغییر یافت
    const recipes = [
        { 
            ingredients: ['متیل‌آمین', 'اسید کلرید', 'سود سوزآور'], 
            result: 'ELC', 
            color: 0x00f7ff 
        },
        { 
            ingredients: ['فسفر قرمز', 'اسید کلرید'], 
            result: 'فسفر مایع تصفیه شده', 
            color: 0x9b59b6 
        }
    ];

    init3D();

    function init3D() {
        scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x020205, 0.15);

        camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        document.body.appendChild(renderer.domElement);

        const mainLight = new THREE.PointLight(0x3498db, 2, 15);
        mainLight.position.set(2, 3, 2);
        scene.add(mainLight);

        const fillLight = new THREE.PointLight(0xffffff, 1, 15);
        fillLight.position.set(-2, 1, -2);
        scene.add(fillLight);

        scene.add(new THREE.AmbientLight(0x404040, 0.8));

        const geometry = new THREE.CylinderGeometry(1.2, 1.2, 3, 64, 1, true);
        const material = new THREE.MeshPhysicalMaterial({ 
            color: 0xffffff,
            metalness: 0.1, roughness: 0.05, transmission: 0.9, thickness: 0.5,
            transparent: true, opacity: 0.4, side: THREE.DoubleSide
        });
        beaker = new THREE.Mesh(geometry, material);
        scene.add(beaker);

        const baseGeom = new THREE.CylinderGeometry(1.3, 1.3, 0.2, 64);
        const baseMat = new THREE.MeshPhongMaterial({ color: 0x222222 });
        const base = new THREE.Mesh(baseGeom, baseMat);
        base.position.y = -1.6;
        scene.add(base);

        const liqGeom = new THREE.CylinderGeometry(1.15, 1.15, 2.9, 64);
        const liqMat = new THREE.MeshStandardMaterial({ 
            color: 0x3498db, transparent: true, opacity: 0.8,
            emissive: 0x3498db, emissiveIntensity: 0.2
        });
        liquid = new THREE.Mesh(liqGeom, liqMat);
        liquid.scale.set(1, 0.01, 1);
        liquid.position.y = -1.45;
        scene.add(liquid);

        const grid = new THREE.GridHelper(20, 40, 0x3498db, 0x111111);
        grid.position.y = -1.7;
        scene.add(grid);

        camera.position.set(0, 1.5, 7);
        camera.lookAt(0, 0, 0);

        animate();
    }

    function animate() {
        requestAnimationFrame(animate);
        const time = Date.now() * 0.001;
        if (beaker) beaker.rotation.y = Math.sin(time * 0.5) * 0.1;
        if (isProcessing) {
            liquid.material.emissiveIntensity = 0.5 + Math.sin(time * 10) * 0.5;
        }
        renderer.render(scene, camera);
    }

    function createBottle(color) {
        const bottleGroup = new THREE.Group();
        const bodyMat = new THREE.MeshPhongMaterial({ color: color, transparent: true, opacity: 0.7 });
        const body = new THREE.Mesh(new THREE.CylinderGeometry(0.4, 0.4, 1, 32), bodyMat);
        bottleGroup.add(body);
        const shoulder = new THREE.Mesh(new THREE.SphereGeometry(0.4, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2), bodyMat);
        shoulder.position.y = 0.5;
        bottleGroup.add(shoulder);
        const neck = new THREE.Mesh(new THREE.CylinderGeometry(0.15, 0.15, 0.5, 32), bodyMat);
        neck.position.y = 0.8;
        bottleGroup.add(neck);
        const cap = new THREE.Mesh(new THREE.CylinderGeometry(0.18, 0.18, 0.15, 32), new THREE.MeshPhongMaterial({ color: 0x333333 }));
        cap.position.y = 1.05;
        bottleGroup.add(cap);
        return bottleGroup;
    }

    function pourChemical(name, colorCode) {
        if (isPouring || isProcessing || currentIngredients.length >= 4) return;
        isPouring = true;
        
        addLog(`اضافه کردن: ${name}`);
        const bottle = createBottle(colorCode);
        bottle.position.set(4, 3, 0);
        scene.add(bottle);

        let steps = 0;
        const interval = setInterval(() => {
            steps++;
            if (steps < 30) { bottle.position.x -= 0.1; bottle.position.y -= 0.03; }
            else if (steps < 60) { bottle.rotation.z += 0.05; }
            else if (steps < 100) {
                liquid.scale.y = Math.min(liquid.scale.y + 0.005, 1);
                liquid.position.y = -1.45 + (liquid.scale.y * 1.45);
                liquid.material.color.lerp(new THREE.Color(colorCode), 0.1);
            }
            else if (steps < 130) { bottle.rotation.z -= 0.05; }
            else if (steps < 160) { bottle.position.x += 0.1; bottle.position.y += 0.03; }
            else {
                clearInterval(interval);
                scene.remove(bottle);
                currentIngredients.push(name);
                isPouring = false;
                if (currentIngredients.length >= 2) document.getElementById('mixBtn').disabled = false;
            }
        }, 16);
    }

    function addLog(msg, color = "#3498db") {
        const logBox = document.getElementById('log-box');
        logBox.innerHTML += `<br><span style="color:${color}">></span> ${msg}`;
        logBox.scrollTop = logBox.scrollHeight;
    }

    function processCraft() {
        if (isProcessing) return;
        
        isProcessing = true;
        document.getElementById('mixBtn').disabled = true;
        document.getElementById('resetBtn').disabled = true;
        document.getElementById('progress-container').style.display = 'block';
        
        const progressBar = document.getElementById('progress-bar');
        const duration = 10000; // زمان انتظار ۱۰ ثانیه
        let startTime = Date.now();
        
        addLog("شروع واکنش شیمیایی (۱۰ ثانیه)...", "#f1c40f");

        const timer = setInterval(() => {
            let elapsed = Date.now() - startTime;
            let progress = (elapsed / duration) * 100;
            
            progressBar.style.width = Math.min(progress, 100) + "%";

            if (progress >= 100) {
                clearInterval(timer);
                isProcessing = false;
                document.getElementById('progress-container').style.display = 'none';
                document.getElementById('resetBtn').disabled = false;
                validateRecipe();
            }
        }, 100);
    }

    function validateRecipe() {
        const found = recipes.find(r => 
            r.ingredients.length === currentIngredients.length &&
            r.ingredients.every(i => currentIngredients.includes(i))
        );

        if (found) {
            addLog(`تکمیل واکنش: ${found.result} با موفقیت سنتز شد!`, "#2ecc71");
            liquid.material.color.setHex(found.color);
            liquid.material.emissive.setHex(found.color);
            liquid.material.emissiveIntensity = 2;
            
            // ارسال رویداد به MTA با نام محصول جدید
            if(typeof mta !== "undefined") mta.triggerEvent("onDrugLaboratoryCraft", found.result);
        } else {
            addLog("خطا در سنتز: نسبت مواد نادرست بود. انفجار درونی!", "#e74c3c");
            liquid.material.color.setHex(0xff0000);
            setTimeout(() => resetLab(), 2000);
        }
    }

    function resetLab() {
        if (isProcessing) return;
        currentIngredients = [];
        liquid.scale.y = 0.01;
        liquid.position.y = -1.45;
        liquid.material.color.setHex(0x3498db);
        liquid.material.emissiveIntensity = 0.2;
        document.getElementById('mixBtn').disabled = true;
        document.getElementById('progress-bar').style.width = "0%";
        document.getElementById('progress-container').style.display = 'none';
        addLog("مخزن تخلیه و پاکسازی شد.");
    }

    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });
</script>

</body>
</html>
