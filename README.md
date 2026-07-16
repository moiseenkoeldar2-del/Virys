const factions = {
    UA: {
        name: "ВСУ",
        flag: "🇺🇦",
        enemyFlag: "🇷🇺",
        drones: {
            fpv1: { name: "FPV-камикадзе v1 (Hornet Mini)", costParts: 1, costEnergy: 1, power: 1, ewerChance: 0.1, icon: "🛸", desc: "Быстрый точечный удар по блиндажам." },
            fpv2: { name: "FPV-камикадзе v2 (Тепловизор)", costParts: 2, costEnergy: 1, power: 2, icon: "🛩️", ewerChance: 0.2, desc: "Игнорирует ночную маскировку." },
            hornet: { name: "Тяжелый БПЛА Hornet", costParts: 4, costEnergy: 3, power: 5, icon: "🦅", ewerChance: 0.4, desc: "Многоразовый бомбардировщик. Пробивает укрепления." }
        }
    },
    RU: {
        name: "Армия РФ",
        flag: "🇷🇺",
        enemyFlag: "🇺🇦",
        drones: {
            fpv: { name: "ВТ-40 FPV-дрон", costParts: 1, costEnergy: 1, power: 1, icon: "🛸", ewerChance: 0.1, desc: "Массовый тактический коптер." },
            seeker: { name: "Скаут Seeker", costParts: 2, costEnergy: 2, power: 2, icon: "👁️", ewerChance: 0.3, desc: "Корректировщик, вскрывает ПВО врага." },
            geran: { name: "Герань-2/4 (Дальний радиус)", costParts: 5, costEnergy: 3, power: 6, icon: "📐", ewerChance: 0.5, desc: "Дальнобойный камикадзе для стратегических ударов." },
            iskander: { name: "ОТРК Искандер-М", costParts: 12, costEnergy: 6, power: 10, icon: "🚀", ewerChance: 0.02, desc: "Тяжелая баллистическая ракета. Почти неуязвима для РЭБ." }
        }
    }
};

let gameMode = "UA"; // "UA", "RU" или "AI"
let currentFaction = "UA"; 
let resources = { energy: 10, parts: 15, labs: 2 };
let fleet = {}; 

let turn = 1;
let frontLine = 0; 
let enemyEwer = 20; 
let activeConvoy = null; 
let aiInterval = null;

function selectMode(mode) {
    gameMode = mode;
    
    // Настройка интерфейса под режим
    if (mode === "AI") {
        currentFaction = "UA"; 
        document.getElementById('ai-badge').style.display = 'inline';
        document.getElementById('manual-switch-btn').style.display = 'none';
        document.getElementById('turn-btn').style.display = 'none';
    } else {
        currentFaction = mode;
        document.getElementById('ai-badge').style.display = 'none';
        document.getElementById('manual-switch-btn').style.display = 'block';
        document.getElementById('turn-btn').style.display = 'block';
    }

    document.getElementById('start-menu').style.display = 'none';
    document.getElementById('game-interface').style.display = 'block';
    
    // Сброс параметров
    turn = 1; frontLine = 0; enemyEwer = 20; activeConvoy = null;
    resources = { energy: 10, parts: 15, labs: 2 };
    
    initFaction();

    // Запуск цикла автоматического режима ИИ
    if (mode === "AI") {
        if(aiInterval) clearInterval(aiInterval);
        aiInterval = setInterval(runAiVsAiCycle, 3000); // ИИ делает ходы каждые 3 секунды
    }
}

function initFaction() {
    const fData = factions[currentFaction];
    document.getElementById('faction-title').innerText = fData.name;
    document.getElementById('our-flag').innerText = fData.flag;
    document.getElementById('enemy-flag').innerText = fData.enemyFlag;

    fleet = {};
    for (let key in fData.drones) { fleet[key] = 2; }

    renderControls();
    updateUI();
    log(`Режим инициализирован. Сторона: ${fData.name}.`, 'system');
}

// ЭКРАН ТАКТИЧЕСКОГО ПРИБЛИЖЕНИЯ (ВИД СВЕРХУ НА КОЛОННУ ИЛИ РАКЕТУ)
function showZoomAnimation(droneName, droneIcon) {
    const overlay = document.getElementById('zoom-overlay');
    const title = document.getElementById('zoom-title');
    const container = document.getElementById('battlefield-animation');
    
    title.innerText = `ПРИБЛИЖЕНИЕ КАМЕРЫ Х24: ВЫЛЕТ СРЕДСТВА [${droneName.toUpperCase()}]`;
    container.innerHTML = "";
    
    overlay.style.display = 'flex';

    // Создаем объект «вид сверху», падающий по экрану
    let object = document.createElement('div');
    object.className = "falling-object";
    object.innerText = droneIcon;
    container.appendChild(object);

    // Скрываем анимацию приближения через 1.8 секунды
    setTimeout(() => {
        overlay.style.display = 'none';
    }, 1800);
}

function renderControls() {
    const fData = factions[currentFaction];
    const container = document.getElementById('drone-controls');
    container.innerHTML = "";

    for (let key in fData.drones) {
        let d = fData.drones[key];
        let btn = document.createElement('button');
        btn.className = "drone-strike-btn";
        btn.disabled = (gameMode === "AI"); // Отключаем кнопки в режиме наблюдения
        btn.onclick = () => launchDrone(key);
        btn.innerHTML = `
            <strong>🚀 Запустить ${d.name}</strong><br>
            <small>${d.desc}</small><br>
            <small style="color: #94a3b8;">Расход: 🔧 ${d.costParts} | 🔋 ${d.costEnergy} (На складе: <span id="stock-${key}">0</span>)</small>
        `;
        container.appendChild(btn);
    }
}

function updateUI() {
    document.getElementById('energy-txt').innerText = resources.energy;
    document.getElementById('parts-txt').innerText = resources.parts;
    document.getElementById('labs-txt').innerText = resources.labs;

    const fData = factions[currentFaction];
    const arsenalGrid = document.getElementById('arsenal-grid');
    arsenalGrid.innerHTML = "";

    for (let key in fData.drones) {
        if(document.getElementById(`stock-${key}`)) {
            document.getElementById(`stock-${key}`).innerText = fleet[key];
        }
        
        let item = document.createElement('div');
        item.className = "stat-box";
        item.innerHTML = `<div>${fData.drones[key].name}</div><div class="stat-val">${fleet[key]} шт</div>`;
        arsenalGrid.appendChild(item);
    }

    const statusTxt = document.getElementById('front-status');
    
    if (activeConvoy === 'RU_CONVOY') { statusTxt.innerText = "🛑 КОЛОННА РФ: Из Запорожской обл. в Крым!"; statusTxt.style.color = "#ffb4ab"; }
    else if (activeConvoy === 'UA_CONVOY_POKROVSK') { statusTxt.innerText = "⚡ КОЛОННА ВСУ: Из Днепра на Покровск!"; statusTxt.style.color = "#60a5fa"; }
    else if (activeConvoy === 'UA_CONVOY_SUMY') { statusTxt.innerText = "🛡️ ВОЕННЫЕ ВСУ: Из Львова в Сумы!"; statusTxt.style.color = "#38bdf8"; }
    else {
        if (frontLine === 0) { statusTxt.innerText = "Линия стабильна"; statusTxt.style.color = "#cbd5e1"; }
        else if (frontLine > 0) { statusTxt.innerText = `Успех ВСУ (+${frontLine})`; statusTxt.style.color = "#b4e6b7"; }
        else { statusTxt.innerText = `Успех РФ (${frontLine})`; statusTxt.style.color = "#ffb4ab"; }
    }

    document.getElementById('enemy-defense').innerText = `РЭБ противника: ${enemyEwer}%`;
}

function log(msg, type = 'system') {
    const container = document.getElementById('log-container');
    const entry = document.createElement('div');
    entry.className = `log-entry ${type}`;
    entry.innerText = `[Ход ${turn}] ${msg}`;
    container.appendChild(entry);
    container.scrollTop = container.scrollHeight;
}

function switchCountry() {
    if(gameMode === "AI") return;
    currentFaction = currentFaction === "UA" ? "RU" : "UA";
    frontLine = -frontLine; 
    initFaction();
}

function launchDrone(droneKey) {
    const d = factions[currentFaction].drones[droneKey];
    const colorClass = currentFaction === "UA" ? "ua-color" : "ru-color";

    if (gameMode !== "AI" && (resources.parts < d.costParts || resources.energy < d.costEnergy)) {
        log(`Недостаточно ресурсов для пуска ${d.name}!`, 'warn');
        return;
    }

    resources.parts -= d.costParts;
    resources.energy -= d.costEnergy;

    // Срабатывание тактического приближения сверху
    showZoomAnimation(d.name, d.icon);

    setTimeout(() => {
        let isSuppressed = Math.random() * 100 < (enemyEwer * d.ewerChance);

        if (isSuppressed) {
            log(`❌ ${d.name} был подавлен системами РЭБ/ПРО противника.`, 'warn');
        } else {
            let damage = d.power;
            
            if (currentFaction === "UA" && activeConvoy === "RU_CONVOY") {
                damage *= 2; activeConvoy = null; frontLine += damage;
                log(`🎯 Разведка нашла цель! ${d.name} разгромил колонну РФ «Запорожье ➔ Крым»! Критический урон: +${damage}`, 'ua-color');
            } else if (currentFaction === "RU" && (activeConvoy === "UA_CONVOY_POKROVSK" || activeConvoy === "UA_CONVOY_SUMY")) {
                damage *= 2; 
                if (activeConvoy === "UA_CONVOY_SUMY") log(`🎯 Сокрушительный удар! ${d.name} уничтожил эшелон ВСУ «Львов — Сумы»! Критический урон: +${damage}`, 'ru-color');
                else log(`🎯 Перехват! ${d.name} накрыл резервы ВСУ на марше из Днепра в Покровск! Критический урон: +${damage}`, 'ru-color');
                activeConvoy = null; frontLine -= damage; 
            } else {
                frontLine += (currentFaction === "UA") ? damage : -damage;
                log(`🎯 Точный прилет! ${d.name} нанес удар по объектам. Урон: ${damage}`, colorClass);
            }
        }

        enemyEwer = Math.min(80, enemyEwer + Math.floor(Math.random() * 4));
        updateUI();
    }, 1800); // Ждем окончания анимации
}

function checkRandomEvents() {
    if (activeConvoy === 'RU_CONVOY') { frontLine -= 4; log(`⚠️ Колонна РФ успешно прибыла в Крым. (-4 к фронту)`, 'warn'); activeConvoy = null; }
    else if (activeConvoy === 'UA_CONVOY_POKROVSK') { frontLine += 4; log(`⚠️ Колонна ВСУ прибыла в Покровск. (+4 к фронту)`, 'ua-color'); activeConvoy = null; }
    else if (activeConvoy === 'UA_CONVOY_SUMY') { frontLine += 5; log(`⚠️ Военные ВСУ прибыли из Львова в Сумы. (+5 к фронту)`, 'ua-color'); activeConvoy = null; }

    if (Math.random() < 0.35) {
        let route = Math.floor(Math.random() * 3);
        if (route === 0) { activeConvoy = 'RU_CONVOY'; enemyEwer = Math.max(10, enemyEwer - 15); log(🚨 ВНИМАНИЕ: Из Запорожской области в сторону Крыма выдвинулась колонна техники РФ!, 'warn'); }else if (route === 1) { activeConvoy = 'UA_CONVOY_POKROVSK'; log(🚨 СВОДКА: Из Днепра в направлении Покровска начала движение колонна ВСУ!, 'system'); }else if (route === 2) { activeConvoy = 'UA_CONVOY_SUMY'; log(🚨 МАНЕВР: Из Львова в Сумы отправился крупный эшелон с военными ВСУ!, 'system'); }}}function nextTurn() {turn++;resources.energy += resources.labs * 3;resources.parts += 5;checkRandomEvents();// Имитация огня противника в ручных режимахif (gameMode !== "AI") {let aiStrike = Math.floor(Math.random() * 3);if(aiStrike > 0) {frontLine += (currentFaction === "UA") ? -aiStrike : aiStrike;log(⚠️ Противник применил ответный рой БПЛА. Нанесен урон: -${aiStrike}, 'warn');}}enemyEwer = Math.max(10, enemyEwer - 10);updateUI();if (frontLine >= 12 || frontLine <= -12) {if(aiInterval) clearInterval(aiInterval);alert("Симуляция завершена. Возврат в тактический штаб.");returnToMenu();}}// ЦИКЛ АВТОМАТИЧЕСКОГО РЕЖИМА ИИ ПРОТИВ ИИfunction runAiVsAiCycle() {// Случайный выбор средства для текущей фракцииconst dKeys = Object.keys(factions[currentFaction].drones);const randKey = dKeys[Math.floor(Math.random() * dKeys.length)];log([ИИ Наблюдение] Фракция ${factions[currentFaction].name} принимает решение о пуске...);launchDrone(randKey);// Смена активной фракции для следующего шага симуляции ИИsetTimeout(() => {currentFaction = currentFaction === "UA" ? "RU" : "UA";frontLine = -frontLine; // Инвертируем карту под другую фракциюconst fData = factions[currentFaction];document.getElementById('faction-title').innerText = fData.name;document.getElementById('our-flag').innerText = fData.flag;document.getElementById('enemy-flag').innerText = fData.enemyFlag;renderControls();nextTurn();}, 2000);}function returnToMenu() {if(aiInterval) clearInterval(aiInterval);document.getElementById('start-menu').style.display = 'flex';document.getElementById('game-interface').style.display = 'none';}
