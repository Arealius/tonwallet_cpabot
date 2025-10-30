Техническое задание: интеграция TON Connect подписки с расчётом в USD и выбором кошелька

🎯 Цель
Реализовать подписку в Telegram мини-приложении CPABot, где:

- Пользователь может подключить любой TON-кошелёк (Tonkeeper, Telegram Wallet, OpenMask, MyTonWallet и др.);
- Стоимость подписки задана в USD (например $15);
- На сервере Laravel курс TON→USD определяется автоматически;
- Пользователь оплачивает эквивалентную сумму в TON;
- После оплаты активируется подписка (30 дней);
- Истёкшие подписки автоматически деактивируются.

──────────────────────────────────────────────

⚙️ 1. Переменные окружения (.env)
TON_MERCHANT_ADDRESS=EQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TON_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SUB_PRICE_USD=15
SUB_TERM_DAYS=30

──────────────────────────────────────────────

🧩 2. Файл манифеста TON Connect
Создать файл: /public/tonconnect-manifest.json

{
  "url": "https://cpabot.app",
  "name": "CPABot",
  "iconUrl": "https://cpabot.app/content/images/icons/logo.svg",
  "termsOfUseUrl": "https://cpabot.app/terms",
  "privacyPolicyUrl": "https://cpabot.app/privacy"
}

──────────────────────────────────────────────

🗄 3. Миграции БД

3.1 Таблица ton_orders:
Schema::create('ton_orders', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id')->index();
    $table->string('order_id')->unique();
    $table->string('plan')->default('monthly');
    $table->unsignedBigInteger('amount_nano');
    $table->decimal('rate_ton_usd', 10, 4)->nullable();
    $table->string('payer_wallet')->nullable();
    $table->string('status')->default('pending'); // pending|paid|expired
    $table->timestamps();
});

3.2 Таблица subscriptions:
Schema::create('subscriptions', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id')->index();
    $table->string('plan')->default('monthly');
    $table->timestamp('starts_at')->nullable();
    $table->timestamp('expires_at')->nullable();
    $table->boolean('is_active')->default(false);
    $table->timestamps();
});

──────────────────────────────────────────────

💻 4. Контроллер TonPaymentController.php
(Полный код контроллера из основного ответа — создание заказа, получение курса TON/USD, проверка оплаты, активация подписки)

──────────────────────────────────────────────

🔗 5. Маршруты routes/api.php
Route::post('/ton/order-create', 'TonPaymentController@createOrder');
Route::post('/ton/order-verify', 'TonPaymentController@verifyOrder');

──────────────────────────────────────────────

🌐 6. HTML + JS фронтенд (Telegram mini-app)

<script src="https://unpkg.com/@tonconnect/sdk@latest/dist/tonconnect-sdk.min.js"></script>

<button id="connectTon" class="home-form__button">🔗 Подключить TON-кошелёк</button>
<button id="buyTon" class="home-form__button">💎 Купить подписку</button>

<script>
const tonConnect = new TON_CONNECT.TonConnect({
  manifestUrl: 'https://cpabot.app/tonconnect-manifest.json'
});

// Подключение с выбором кошелька
document.getElementById('connectTon').onclick = async () => {
  try {
    const wallets = await tonConnect.getWallets();
    if (!wallets.length) {
      alert("Нет доступных кошельков. Установите Tonkeeper или Telegram Wallet.");
      return;
    }

    let html = '<h3 style="margin-bottom:10px">Выберите кошелёк:</h3>';
    wallets.forEach((w, i) => {
      html += `<button style="display:block;margin:5px;padding:8px 12px;border-radius:8px;width:100%;text-align:left"
                 onclick="selectWallet(${i})">
                 <img src="${w.image}" width="24" height="24" style="vertical-align:middle;margin-right:6px">
                 ${w.name}
               </button>`;
    });
    const modal = document.createElement('div');
    modal.innerHTML = html;
    modal.style.cssText = 'position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);background:#fff;padding:15px;border-radius:12px;z-index:9999;width:260px;text-align:center';
    document.body.appendChild(modal);

    window.selectWallet = async (i) => {
      const wallet = wallets[i];
      try {
        await tonConnect.connect(wallet);
        alert(`✅ Подключен ${wallet.name}`);
        modal.remove();
      } catch(e){ alert("Ошибка подключения"); }
    };

  } catch(e){
    alert("Ошибка при получении списка кошельков");
  }
};

// Оплата
document.getElementById('buyTon').onclick = async () => {
  if (!tonConnect.wallet) return alert("Сначала подключите кошелёк");

  const res = await fetch('/api/ton/order-create', {
    method:'POST',
    headers:{'Content-Type':'application/json'},
    body:JSON.stringify({ plan:'monthly' })
  }).then(r=>r.json());

  if (!res.ok) return alert("Ошибка: "+res.message);

  alert(`Стоимость подписки: ${res.usd}$ ≈ ${res.ton} TON\nКурс: 1 TON = ${res.rate}$`);

  const tx = {
    validUntil: Math.floor(Date.now()/1000)+600,
    messages: [{
      address: res.merchantAddress,
      amount: String(res.amountNano),
      payload: btoa(`CPABOT:${res.orderId}`)
    }]
  };

  try {
    await tonConnect.sendTransaction(tx);
    alert("Платёж отправлен, идёт проверка…");

    const verify = await fetch('/api/ton/order-verify', {
      method:'POST',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify({
        order_id:res.orderId,
        wallet:tonConnect.wallet.account.address
      })
    }).then(r=>r.json());

    alert(verify.ok ? "✅ Подписка активирована!" : "⏳ Оплата в пути, попробуйте позже");
  } catch(e){ alert("Платёж отменён"); }
};
</script>

──────────────────────────────────────────────

🕓 7. Автоматическая деактивация подписок (CRON)
app/Console/Commands/SubscriptionsCheck.php и Kernel.php — добавить CRON задачу

protected function schedule(Schedule $schedule)
{
    $schedule->command('subscriptions:check')->hourly();
}

──────────────────────────────────────────────

✅ Результат после внедрения

1. Пользователь открывает мини-приложение.
2. Нажимает «Подключить TON-кошелёк» — видит список (Tonkeeper, Wallet, OpenMask, MyTonWallet).
3. Выбирает кошелёк — происходит подключение.
4. Нажимает «Купить подписку» — Laravel рассчитывает курс TON/USD.
5. Пользователь оплачивает нужную сумму в TON.
6. Сервер через TonAPI проверяет транзакцию и активирует подписку.
7. Через 30 дней CRON автоматически отключает неактивные подписки.
