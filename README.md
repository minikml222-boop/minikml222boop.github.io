import json
import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, WebAppInfo
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# ========== НАСТРОЙКИ ==========
BOT_TOKEN = "7891647573:AAFRH9ViaiBrplNwLZ6bY9nDAWNdCeaEfAw"  # Замените на токен от @BotFather
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)

# ========== HTML-ИГРА (ПОЛНОСТЬЮ ВСТРОЕНА) ==========
GAME_HTML = """<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>ЭПШТЕЙН · ОСТРОВ</title>
    <style>
        * { user-select: none; -webkit-tap-highlight-color: transparent; }
        body {
            background: linear-gradient(145deg, #0a0f1a 0%, #03060c 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Courier New', 'VT323', monospace;
            margin: 0;
            padding: 16px;
        }
        .game-container {
            background: #1a1f2a;
            border-radius: 64px;
            max-width: 550px;
            width: 100%;
            padding: 20px;
            border: 2px solid #7a6a3a;
            text-align: center;
        }
        .top-panel { display: flex; justify-content: space-between; margin-bottom: 16px; flex-wrap: wrap; gap: 10px; }
        .lives { background: #0c0f0b; border-radius: 60px; padding: 6px 12px; color: #ff5555; font-weight: bold; flex: 1; }
        .coins { background: #2c2b1f; border-radius: 60px; padding: 6px 12px; color: #ffd966; font-weight: bold; flex: 1; }
        .score-box { background: #2c2b1f; border-radius: 40px; padding: 6px 12px; margin-bottom: 16px; display: inline-block; }
        .event-log { background: #0a0f05e0; border-radius: 32px; padding: 12px; min-height: 110px; margin: 14px 0; color: #e2ffd6; }
        .eat-btn { background: radial-gradient(circle at 30% 10%, #b34e2e, #5a1e0c); border: none; font-size: 1.6rem; padding: 14px; width: 85%; border-radius: 100px; color: #ffefc0; cursor: pointer; margin-bottom: 14px; }
        .shop { background: #2a2a1a; border-radius: 32px; padding: 10px; margin: 12px 0; display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; }
        .shop-btn { background: #3a3a2a; border: none; padding: 8px 12px; border-radius: 40px; color: #ffefc0; cursor: pointer; }
        .reset-btn { background: #2a2a2a; border: none; padding: 8px 16px; border-radius: 40px; color: #dddddd; width: 70%; cursor: pointer; margin-top: 6px; }
        .footer { font-size: 0.5rem; color: #799b6e; margin-top: 14px; }
        button:active { transform: scale(0.98); }
    </style>
</head>
<body>
<div class="game-container" id="game">
    <div class="top-panel"><div class="lives" id="livesDisplay">❤️❤️❤️</div><div class="coins">🪙 <span id="coinsValue">0</span></div></div>
    <div class="score-box">🍼 СЪЕДЕНО: <span id="scoreValue">0</span></div>
    <div class="event-log" id="eventLog">👉 ЖМИ НА КНОПКУ 👈</div>
    <button class="eat-btn" id="eatButton">🍖 НЯМ-НЯМ 🍖</button>
    <div class="shop" id="shopContainer"></div>
    <button class="reset-btn" id="resetButton">🔄 НАЧАТЬ ЗАНОВО</button>
    <div class="footer">💰 +5(никого) | +3(добавили жизнь) | +1(отняли жизнь)</div>
</div>
<script>
    let lives = 3, score = 0, coins = 0, gameActive = true;
    const livesSpan = document.getElementById('livesDisplay');
    const scoreSpan = document.getElementById('scoreValue');
    const coinsSpan = document.getElementById('coinsValue');
    const eventDiv = document.getElementById('eventLog');
    const eatBtn = document.getElementById('eatButton');
    const resetBtn = document.getElementById('resetButton');
    const shopContainer = document.getElementById('shopContainer');

    function updateUI() {
        let hearts = ''; for(let i=0;i<lives;i++) hearts+='❤️ ';
        if(lives<=0) hearts='💀 МЕРТВ 💀';
        livesSpan.innerText = hearts.trim() || '💀';
        scoreSpan.innerText = score;
        coinsSpan.innerText = Math.floor(coins);
    }
    function setEventMessage(text) { eventDiv.innerHTML = text.replace(/\\n/g,'<br>'); }
    function addCoins(amount) { coins += amount; updateUI(); return amount; }
    function endGame() { gameActive = false; eatBtn.disabled = true; setEventMessage("💀 ИГРА ОКОНЧЕНА 💀"); }
    function applyDamage(dmg, reason) { let old = lives; lives = Math.max(0, lives - dmg); updateUI(); setEventMessage(`${reason}\\n❤️ ${old} → ${lives}`); if(lives<=0) endGame(); return dmg; }
    function applyHeal(heal, reason) { let old = lives; lives += heal; updateUI(); setEventMessage(`${reason}\\n❤️ ${old} → ${lives}`); }
    
    function triggerRandomGuest() {
        const rand = Math.random()*100;
        if(rand<30) { let g=addCoins(5); setEventMessage(`😶 Никого... +${g}🪙`); }
        else if(rand<45) { applyDamage(3, "🚨 ПОЛИЦИЯ! -3 жизни"); addCoins(1); }
        else if(rand<55) { applyDamage(1, "🔪 ТЕСАК! -1 жизнь"); addCoins(1); }
        else if(rand<65) { applyDamage(1, "🩸 ЧИКАТИЛО! -1 жизнь"); addCoins(1); }
        else if(rand<77) { applyHeal(1, "🥒 ОГУЗОК! +1 жизнь"); addCoins(3); }
        else if(rand<89) { applyHeal(3, "🕊️ НАВАЛЬНЫЙ! +3 жизни"); addCoins(3); }
        else if(rand<99) { applyDamage(2, "🐻 ПУТИН! -2 жизни"); addCoins(1); }
        else { setEventMessage("🇺🇦 ЗЕЛЕНСКИЙ УБИЛ! 💀"); lives=0; updateUI(); endGame(); }
    }
    
    function eatChild() {
        if(!gameActive || lives<=0) return;
        score++; updateUI(); setEventMessage(`🍽️ Съеден ребёнок №${score}\\n🌑 Ожидание...`);
        setTimeout(()=>{ if(gameActive && lives>0) triggerRandomGuest(); if(lives<=0) endGame(); updateUI(); },250);
    }
    
    function renderShop() {
        shopContainer.innerHTML = '';
        const btn1 = document.createElement('button'); btn1.innerText='❤️ +1 жизнь (15🪙)'; btn1.className='shop-btn';
        btn1.onclick=()=>{ if(coins>=15){ coins-=15; lives+=1; updateUI(); setEventMessage(`🛒 +1 жизнь! ❤️ ${lives}`); } else setEventMessage(`❌ Нужно 15🪙`); };
        const btn3 = document.createElement('button'); btn3.innerText='❤️❤️❤️ +3 жизни (40🪙)'; btn3.className='shop-btn';
        btn3.onclick=()=>{ if(coins>=40){ coins-=40; lives+=3; updateUI(); setEventMessage(`🛒 +3 жизни! ❤️ ${lives}`); } else setEventMessage(`❌ Нужно 40🪙`); };
        shopContainer.append(btn1,btn3);
    }
    
    eatBtn.onclick = eatChild;
    resetBtn.onclick = () => { lives=3; score=0; coins=0; gameActive=true; eatBtn.disabled=false; updateUI(); setEventMessage("🔄 ИГРА ПЕРЕЗАПУЩЕНА"); renderShop(); };
    renderShop();
    setEventMessage("🍭 ИГРА ГОТОВА! Жми НЯМ-НЯМ");
</script>
</body>
</html>"""

# ========== КОМАНДА /START ==========
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("🍆 ЗАПУСТИТЬ ИГРУ 🍆", web_app=WebAppInfo(url="https://core.telegram.org/widgets/"))]
    ]
    # ВНИМАНИЕ: WebApp требует HTTPS и реальный домен.
    # НО мы используем хитрость: передаём HTML через data: URL
    # Для этого нужно создать data:text/html,<HTML>
    import urllib.parse
    encoded_html = urllib.parse.quote(GAME_HTML)
    webapp_url = f"data:text/html,{encoded_html}"
    
    keyboard = [
        [InlineKeyboardButton("🍆 ЗАПУСТИТЬ ИГРУ 🍆", web_app=WebAppInfo(url=webapp_url))]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("🎮 Нажми на кнопку, чтобы запустить игру прямо в Telegram:", reply_markup=reply_markup)

# ========== ЗАПУСК БОТА ==========
def main():
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    print("🤖 Бот запущен...")
    app.run_polling(allowed_updates=["message", "callback_query"])

if __name__ == "__main__":
    main()
