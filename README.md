import tkinter as tk
import threading
import time
import ccxt


class DVYUDesign:
    def __init__(self, root):
        self.root = root
        root.title("DVYU")
        root.geometry("250x600")
        root.configure(bg="#000000")
        root.resizable(False, False)

        self.running = False
        self.exchange = None
        self.last_signal = None
        self.entry_price = 0.0
        self.last_closed_side = None
        self.demo_balance = 10_000_000.0
        self.demo_start_balance = 10_000_000.0
        self.demo_position = 0.0
        self.demo_entry = 0.0
        self.demo_side = None
        self.demo_capital_deployed = 0.0
        self.demo_entries_count = 0

        self.real_position_amount = 0.0
        self.real_entry_cost = 0.0
        self.real_entry_price = 0.0
        self.real_side = None
        self.real_entries_count = 0

        self.max_entries = 4
        self.max_adverse_move = 0.03
        self.demo_compound = True
        self.target_pct_default = 0.50  # 3% from average entry
        self.entry_candle_ts = None
        self.total_profit = 0.0
        self.trade_count = 0
        self.win_count = 0
        self.last_trade_profit = 0.0
        self.notification_buffer = []
        self._notification_lock = threading.Lock()
        self._notification_index = 0
        self.million_target = 1_000_000.0
        self.live_exchange = None
        self.live_symbol = None
        self.live_price_running = True
        self.display_price = None
        self.target_display_price = None
        self.price_animation_running = False

        self.build_ui()
        self.start_live_price_updater()
        self.start_price_smoothing()

    # =========================================================
    # UI - CLEAN PRO 250x520
    # =========================================================
    def build_ui(self):

        BLACK = "#000000"
        PANEL = "#0B0B0B"
        PANEL2 = "#101010"
        WHITE = "#FFFFFF"
        MUTED = "#777777"
        BORDER = "#3A3A3A"
        RED = "#E53935"

        self.root.geometry("250x600")
        self.root.configure(bg=BLACK)
        self.root.resizable(False, False)

        # Main shell
        shell = tk.Frame(self.root, bg=BLACK)
        shell.pack(fill="both", expand=True, padx=7, pady=7)

        # Top accent
        tk.Frame(shell, bg=RED, height=2).pack(fill="x", pady=(0, 7))

        # Header
        header = tk.Frame(shell, bg=BLACK)
        header.pack(fill="x")

        tk.Label(
            header,
            text="DVYU",
            bg=BLACK,
            fg=WHITE,
            font=("Consolas", 13, "bold")
        ).pack(side="left")

        tk.Label(
            header,
            text="PRO",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 9, "bold")
        ).pack(side="left", padx=(4, 0), pady=(3, 0))

        tk.Button(
            header,
            text="×",
            command=self.root.destroy,
            bg=BLACK,
            fg=WHITE,
            activebackground=BLACK,
            activeforeground=RED,
            relief="flat",
            bd=0,
            font=("Segoe UI", 12, "bold"),
            padx=3,
            pady=0
        ).pack(side="right")

        # Status / mode
        mode_row = tk.Frame(shell, bg=BLACK)
        mode_row.pack(fill="x", pady=(5, 7))

        self.mode = tk.Label(
            mode_row,
            text="DEMO",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 7, "bold")
        )
        self.mode.pack(side="left")

        self.sig_l = tk.Label(
            mode_row,
            text="● STANDBY",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 7, "bold")
        )
        self.sig_l.pack(side="right")

        # Price card
        price_card = tk.Frame(
            shell,
            bg=PANEL,
            highlightbackground=BORDER,
            highlightthickness=1
        )
        price_card.pack(fill="x", pady=(0, 7))

        self.price_l = tk.Label(
            price_card,
            text="$0.0000",
            bg=PANEL,
            fg=WHITE,
            font=("Consolas", 17, "bold")
        )
        self.price_l.pack(pady=(9, 1))

        self.position_l = tk.Label(
            price_card,
            text="",
            bg=PANEL,
            fg=WHITE,
            font=("Consolas", 7, "bold")
        )
        self.position_l.pack(pady=(0, 8))

        # Section title
        tk.Label(
            shell,
            text="MARKET",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 7, "bold")
        ).pack(anchor="w", pady=(0, 4))

        # Market fields
        market = tk.Frame(
            shell,
            bg=BLACK
        )
        market.pack(fill="x")

        def field(parent, label, default, row, col):
            box = tk.Frame(parent, bg=BLACK)
            box.grid(
                row=row,
                column=col,
                padx=(0 if col == 0 else 5, 0 if col == 1 else 5),
                pady=3,
                sticky="ew"
            )

            tk.Label(
                box,
                text=label,
                bg=BLACK,
                fg=MUTED,
                font=("Consolas", 6, "bold")
            ).pack(anchor="w")

            e = tk.Entry(
                box,
                bg=PANEL2,
                fg=WHITE,
                insertbackground=WHITE,
                relief="flat",
                bd=0,
                highlightthickness=1,
                highlightbackground=BORDER,
                highlightcolor=RED,
                justify="center",
                font=("Consolas", 9, "bold")
            )
            e.insert(0, default)
            e.pack(fill="x", ipady=3)
            return e

        self.sym = field(market, "PAIR", "SOL/USDT", 0, 0)
        self.tf_e = field(market, "TIMEFRAME", "1m", 0, 1)
        self.cap = field(market, "CAPITAL", "10000000", 1, 0)
        self.tgt = field(market, "TARGET %", "0.0001", 1, 1)

        market.columnconfigure(0, weight=1)
        market.columnconfigure(1, weight=1)

        # Metrics
        tk.Label(
            shell,
            text="PERFORMANCE",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 7, "bold")
        ).pack(anchor="w", pady=(5, 3))

        perf = tk.Frame(
            shell,
            bg=PANEL,
            highlightbackground=BORDER,
            highlightthickness=1
        )
        perf.pack(fill="x")

        self.pft = tk.Label(
            perf,
            text="TOTAL: +0.00 USDT",
            bg=PANEL,
            fg=WHITE,
            font=("Consolas", 10, "bold")
        )
        self.pft.pack(anchor="w", padx=7, pady=(7, 1))

        self.million_l = tk.Label(
            perf,
            text="To $1M: $1,000,000.00",
            bg=PANEL,
            fg=WHITE,
            font=("Consolas", 6, "bold")
        )
        self.million_l.pack(anchor="w", padx=7)

        self.trades_left_l = tk.Label(
            perf,
            text="Trades to $1M: --",
            bg=PANEL,
            fg=MUTED,
            font=("Consolas", 6, "bold")
        )
        self.trades_left_l.pack(anchor="w", padx=7, pady=(0, 2))

        sub = tk.Frame(perf, bg=PANEL)
        sub.pack(fill="x", padx=7, pady=(0, 7))

        self.active = tk.Label(
            sub,
            text="Act: $0",
            bg=PANEL,
            fg=MUTED,
            font=("Consolas", 6, "bold")
        )
        self.active.pack(side="left")

        self.vault_l = tk.Label(
            sub,
            text="Vlt: $10000000",
            bg=PANEL,
            fg=MUTED,
            font=("Consolas", 6, "bold")
        )
        self.vault_l.pack(side="right")

        # Controls
        controls = tk.Frame(shell, bg=BLACK)
        controls.pack(fill="x", pady=(5, 0))

        self.mode_btn = tk.Button(
            controls,
            text="MODE: DEMO",
            bg=PANEL2,
            fg=WHITE,
            activebackground="#151515",
            activeforeground=WHITE,
            relief="flat",
            bd=0,
            highlightthickness=1,
            highlightbackground=BORDER,
            font=("Consolas", 7, "bold"),
            pady=5,
            command=self.toggle_mode
        )
        self.mode_btn.pack(side="left", fill="x", expand=True, padx=(0, 3))

        self.btn = tk.Button(
            controls,
            text="▶ START BOT",
            bg=RED,
            fg=WHITE,
            activebackground="#B71C1C",
            activeforeground=WHITE,
            relief="flat",
            bd=0,
            font=("Consolas", 9, "bold"),
            pady=5,
            command=self.start_bot
        )
        self.btn.pack(side="left", fill="x", expand=True, padx=(3, 0))


        # API
        tk.Label(
            shell,
            text="API",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 7, "bold")
        ).pack(anchor="w", pady=(5, 3))

        api = tk.Frame(shell, bg=BLACK)
        api.pack(fill="x")

        def api_field(label, secret=False):
            tk.Label(
                api,
                text=label,
                bg=BLACK,
                fg=MUTED,
                font=("Consolas", 6, "bold")
            ).pack(anchor="w")

            e = tk.Entry(
                api,
                bg=PANEL2,
                fg=WHITE,
                insertbackground=WHITE,
                relief="flat",
                bd=0,
                highlightthickness=1,
                highlightbackground=BORDER,
                highlightcolor=RED,
                justify="center",
                font=("Consolas", 7),
                show="•" if secret else ""
            )
            e.pack(fill="x", ipady=3, pady=(1, 4))
            return e

        self.api_key = api_field("API KEY")
        self.api_secret = api_field("API SECRET", secret=True)

        self.saved_api_key = ""
        self.saved_api_secret = ""

        self.api_status = tk.Label(
            api,
            text="● API NOT CONNECTED",
            bg=BLACK,
            fg=MUTED,
            font=("Consolas", 6, "bold")
        )
        self.api_status.pack(anchor="w")

        self.balance_l = tk.Label(
            api,
            text="BALANCE: -- USDT",
            bg=BLACK,
            fg=WHITE,
            font=("Consolas", 7, "bold")
        )
        self.balance_l.pack(anchor="w", pady=(3, 0))

        # Bottom status
        self.status = tk.Label(
            shell,
            text="● READY",
            bg=BLACK,
            fg=MUTED,
            anchor="w",
            font=("Consolas", 6, "bold")
        )
        self.status.pack(fill="x", pady=(5, 0))

    # =========================================================
    # TRADE COMPLETION POPUP
    # =========================================================
    def show_trade_notification(self, message, color="#00FF66"):
        """Show the original native Windows DVYU message box at a free position."""
        import ctypes
        import threading
        import time as _time

        text = str(message).replace("✓ ", "")

        def worker():
            user32 = ctypes.windll.user32
            # MB_OK | MB_ICONINFORMATION
            MB_OK = 0x00000000
            MB_ICONINFORMATION = 0x00000040

            # Find an unused position before the native message box appears.
            with self._notification_lock:
                index = self._notification_index
                self._notification_index += 1

            sw = user32.GetSystemMetrics(0)
            sh = user32.GetSystemMetrics(1)
            ww, wh = 300, 190
            gap = 18
            margin = 25

            cols = max(1, (sw - 2 * margin + gap) // (ww + gap))
            rows = max(1, (sh - 2 * margin + gap) // (wh + gap))
            slot = index % (cols * rows)
            col = slot % cols
            row = slot // cols
            x = margin + col * (ww + gap)
            y = margin + row * (wh + gap)

            # In a background helper, watch for the native DVYU box and move it.
            def position_box():
                for _ in range(100):
                    hwnd = user32.FindWindowW(None, "DVYU")
                    if hwnd:
                        user32.SetWindowPos(hwnd, 0, x, y, 0, 0, 0x0001 | 0x0004)
                        return
                    _time.sleep(0.01)

            threading.Thread(target=position_box, daemon=True).start()
            user32.MessageBoxW(0, text, "DVYU", MB_OK | MB_ICONINFORMATION)

        threading.Thread(target=worker, daemon=True).start()

    def queue_notification(self, message):
        """Display every notification immediately in its own native window."""
        self.show_trade_notification(
            str(message).replace("✓ ", ""),
            "#00FF66"
        )

    # =========================================================
    # ENTRY
    # =========================================================
    def create_entry(self, parent, label, default, r, c):

        f = tk.Frame(parent, bg="#000000")

        f.grid(
            row=r,
            column=c,
            padx=2,
            pady=2,
            sticky="ew"
        )

        tk.Label(
            f,
            text=label,
            bg="#000000",
            fg="#FFFFFF",
            font=("Segoe UI", 7, "bold")
        ).pack(anchor="w")

        e = tk.Entry(
            f,
            justify="center",
            bg="#000000",
            fg="#FFFFFF",
            insertbackground="#FFFFFF",
            relief="flat",
            highlightthickness=1,
            highlightbackground="#000000",
            highlightcolor="#FFFFFF",
            font=("Consolas", 9, "bold")
        )

        e.insert(0, default)
        e.pack(fill="x")

        return e

    # =========================================================
    # ULTRA SMOOTH PRICE DISPLAY
    # =========================================================
    def start_price_smoothing(self):
        self.price_animation_running = True
        self.price_smoothing_loop()

    def price_smoothing_loop(self):
        if not self.price_animation_running:
            return

        if self.target_display_price is not None:
            if self.display_price is None:
                self.display_price = self.target_display_price
            else:
                # Very small interpolation step = visually smooth movement.
                # This does NOT alter the real market price.
                difference = self.target_display_price - self.display_price
                self.display_price += difference * 0.18

                if abs(difference) < max(abs(self.target_display_price) * 0.00000001, 0.00000001):
                    self.display_price = self.target_display_price

            self.price_l.config(
                text=f"${self.display_price:,.8f}"
            )

        # ~10 ms UI animation, independent from network polling.
        self.root.after(10, self.price_smoothing_loop)

    # =========================================================
    # LIVE PRICE - 250ms
    # =========================================================
    def start_live_price_updater(self):
        threading.Thread(
            target=self.live_price_loop,
            daemon=True
        ).start()

    def live_price_loop(self):
        exchange = ccxt.mexc({
            "enableRateLimit": True,
            "options": {"defaultType": "swap"}
        })

        # Markets MUST be loaded before resolving SOL/USDT -> SOL/USDT:USDT.
        try:
            exchange.load_markets()
        except Exception as e:
            self.root.after(
                0,
                self.error_status,
                f"LIVE PRICE INIT: {e}"
            )

        while self.live_price_running:
            try:
                symbol_input = self.sym.get().strip()

                if not symbol_input:
                    time.sleep(0.25)
                    continue

                symbol = self.get_futures_symbol(
                    exchange,
                    symbol_input
                )

                # Fetch the live perpetual price frequently; UI smoothing runs separately.
                ticker = exchange.fetch_ticker(symbol)
                live_price = ticker.get("last")

                if live_price is not None:
                    live_price = float(live_price)

                    self.root.after(
                        0,
                        self.update_price,
                        live_price
                    )

                    self.root.after(
                        0,
                        self.update_live_pnl,
                        live_price
                    )

                time.sleep(0.10)

            except Exception as e:
                # Keep trying forever. The bot must not die because the network dropped.
                msg = str(e).replace("\n", " ")
                if len(msg) > 50:
                    msg = msg[:47] + "..."

                self.root.after(
                    0,
                    self.error_status,
                    f"NETWORK LOST - RETRYING IN 5s"
                )
                try:
                    exchange = ccxt.mexc({
                        "enableRateLimit": True,
                        "options": {"defaultType": "swap"}
                    })
                    exchange.load_markets()
                except Exception:
                    pass
                time.sleep(5)

    def update_live_pnl(self, price):
        # Unrealized P&L is intentionally hidden until the trade is closed.
        return

    # =========================================================
    # MODE
    # =========================================================
    def toggle_mode(self):

        if self.running:
            return

        if self.mode.cget("text") == "DEMO":

            self.mode.config(
                text="REAL",
                fg="#FF5555"
            )

            self.mode_btn.config(
                text="MODE: REAL"
            )

        else:

            self.mode.config(
                text="DEMO",
                fg="#FFFFFF"
            )

            self.mode_btn.config(
                text="MODE: DEMO"
            )

    # =========================================================
    # START
    # =========================================================
    def start_bot(self):

        if self.running:
            self.stop_bot()
            return

        if self.mode.cget("text") == "REAL":

            key = self.api_key.get().strip()
            secret = self.api_secret.get().strip()
            self.saved_api_key = key
            self.saved_api_secret = secret

            if not key or not secret:

                self.api_status.config(
                    text="● API INVALID",
                    fg="#FF3333"
                )

                self.status.config(
                    text="● API KEY REQUIRED",
                    fg="#FF3333"
                )

                return

            self.btn.config(
                text="CHECKING...",
                state="disabled"
            )

            threading.Thread(
                target=self.check_real_api,
                args=(key, secret),
                daemon=True
            ).start()

        else:

            self.start_trading()

    # =========================================================
    # AUTO RECONNECT
    # =========================================================
    def reconnect_real_api(self):
        key = self.saved_api_key or self.api_key.get().strip()
        secret = self.saved_api_secret or self.api_secret.get().strip()

        if not key or not secret:
            return False

        try:
            exchange = ccxt.mexc({
                "apiKey": key,
                "secret": secret,
                "enableRateLimit": True,
                "options": {"defaultType": "swap"}
            })
            exchange.load_markets()
            symbol = self.get_futures_symbol(
                exchange,
                self.sym.get().strip()
            )
            if symbol not in exchange.markets:
                raise RuntimeError(f"FUTURES PAIR NOT AVAILABLE: {symbol}")

            # Verify private API access too.
            balance = exchange.fetch_balance()
            if not isinstance(balance, dict) or "free" not in balance:
                raise RuntimeError("API BALANCE ACCESS FAILED")

            self.exchange = exchange
            self.root.after(0, self.api_success)

            # Refresh the real wallet balance after reconnecting.
            self.update_real_balance(exchange)
            return True

        except Exception as e:
            msg = str(e).replace("\n", " ")
            if len(msg) > 55:
                msg = msg[:52] + "..."
            self.root.after(
                0,
                self.error_status,
                f"RECONNECT FAILED: {msg}"
            )
            return False

    # =========================================================
    def update_real_balance(self, exchange):
        try:
            balance = exchange.fetch_balance()

            usdt = balance.get("USDT", {})
            total = usdt.get("total")

            if total is None:
                total = balance.get("total", {}).get("USDT", 0)

            total = float(total or 0)

            self.root.after(
                0,
                self.balance_l.config,
                {"text": f"BALANCE: {total:,.2f} USDT", "fg": "#00FF66"}
            )
        except Exception:
            self.root.after(
                0,
                self.balance_l.config,
                {"text": "BALANCE: ERROR", "fg": "#FF3333"}
            )

    # =========================================================
    # API CHECK
    # =========================================================
    def check_real_api(self, key, secret):

        self.saved_api_key = key
        self.saved_api_secret = secret

        try:

            exchange = ccxt.mexc({
                "apiKey": key,
                "secret": secret,
                "enableRateLimit": True,
                "options": {"defaultType": "swap"}
            })

            balance = exchange.fetch_balance()
            if not isinstance(balance, dict) or "free" not in balance:
                raise RuntimeError("API BALANCE ACCESS FAILED")

            exchange.load_markets()
            symbol = self.get_futures_symbol(exchange, self.sym.get().strip())
            if symbol not in exchange.markets:
                raise RuntimeError(f"FUTURES PAIR NOT AVAILABLE: {symbol}")

            self.exchange = exchange

            self.root.after(
                0,
                self.api_success
            )

            # Read the real wallet balance immediately after connecting.
            self.update_real_balance(exchange)

        except Exception:

            self.root.after(
                0,
                self.api_failed
            )

    def api_success(self):

        self.api_status.config(
            text="● API CONNECTED",
            fg="#00FF66"
        )

        self.status.config(
            text="● API CONNECTED",
            fg="#00FF66"
        )

        self.btn.config(
            text="▶ START BOT",
            state="normal",
            fg="#FFFFFF"
        )

        self.start_trading()

    def api_failed(self):

        self.api_status.config(
            text="● API INVALID",
            fg="#FF3333"
        )

        self.status.config(
            text="● CONNECTION FAILED",
            fg="#FF3333"
        )

        self.btn.config(
            text="▶ START BOT",
            state="normal"
        )

    # =========================================================
    # START TRADING
    # =========================================================
    def start_trading(self):

        try:
            capital = float(self.cap.get())
            target = float(self.tgt.get())
            if capital <= 0 or target < 0:
                raise ValueError
            if not self.sym.get().strip() or not self.tf_e.get().strip():
                raise ValueError

        except ValueError:

            self.status.config(
                text="● INVALID SETTINGS",
                fg="#FF3333"
            )

            return

        if self.mode.cget("text") == "DEMO":
            self.demo_start_balance = capital
            self.demo_balance = capital
            self.demo_position = 0.0
            self.demo_entry = 0.0
            self.demo_side = None
            self.demo_capital_deployed = 0.0
            self.demo_entries_count = 0
            self.notification_buffer = []

        self.running = True

        self.btn.config(
            text="■ STOP BOT",
            fg="#FFFFFF"
        )

        self.status.config(
            text="● BOT RUNNING • 4×25% • COMPOUND",
            fg="#00FF66"
        )

        threading.Thread(
            target=self.bot_loop,
            daemon=True
        ).start()

    # =========================================================
    # STOP
    # =========================================================
    def stop_bot(self):

        self.running = False
        self.last_signal = None
        self.last_closed_side = None
        self.clear_position_state()

        self.btn.config(
            text="START BOT",
            state="normal"
        )

        self.status.config(
            text="● STOPPED",
            fg="#FFFFFF"
        )

        self.sig_l.config(
            text="● STANDBY",
            fg="#FFFFFF"
        )

        self.position_l.config(
            text="",
            fg="#FFFFFF"
        )

    # =========================================================
    # BOT LOOP
    # =========================================================
    def bot_loop(self):

        while self.running:

            try:

                symbol_input = self.sym.get().strip()
                timeframe = self.tf_e.get().strip()

                if self.mode.cget("text") == "REAL":

                    exchange = self.exchange
                    if exchange is None:
                        raise RuntimeError("REAL FUTURES API NOT CONNECTED")

                else:

                    exchange = ccxt.mexc({
                        "enableRateLimit": True,
                        "options": {"defaultType": "swap"}
                    })

                exchange.load_markets()

                symbol = self.get_futures_symbol(
                    exchange,
                    symbol_input
                )

                if symbol not in exchange.markets:
                    raise RuntimeError(f"FUTURES PAIR NOT AVAILABLE: {symbol}")

                candles = exchange.fetch_ohlcv(
                    symbol,
                    timeframe=timeframe,
                    limit=100
                )

                closes = [x[4] for x in candles]
                highs = [x[2] for x in candles]
                lows = [x[3] for x in candles]

                # Use the LIVE ticker price, not the last candle close.
                ticker = exchange.fetch_ticker(symbol)
                live_price = ticker.get("last")

                if live_price is None:
                    live_price = closes[-1]

                price = float(live_price)

                # PSAR
                psar = self.calculate_psar(
                    highs,
                    lows,
                    closes
                )

                current_psar = psar[-1]

                self.root.after(
                    0,
                    self.update_price,
                    price
                )

                # AUTO ENTRY / AUTO REVERSAL:
                # Do not reopen the same side after a close.
                # A new position in that direction requires the live PSAR
                # side to switch to the opposite direction first.
                current_side = self.current_side()
                desired_side = (
                    "LONG" if price > current_psar
                    else "SHORT" if price < current_psar
                    else None
                )

                if desired_side is not None:

                    # PSAR reversal: do NOT close the position.
                    # If the market moves against the position, reinforce
                    # on the next new candle until the 4 x 25% limit is reached.
                    # A 3% adverse move from the average entry is the safety exit.
                    if current_side is not None and current_side != desired_side:
                        entry = (
                            self.real_entry_price
                            if self.mode.cget("text") == "REAL"
                            else self.demo_entry
                        )

                        adverse = (
                            (entry - price) / entry
                            if current_side == "LONG"
                            else (price - entry) / entry
                        )

                        if adverse >= self.max_adverse_move:
                            self.execute_close(
                                exchange,
                                symbol,
                                price
                            )
                            current_side = self.current_side()
                        elif self.position_entries_count() < self.max_entries:
                            current_candle_ts = candles[-1][0]
                            if (
                                self.entry_candle_ts is not None
                                and current_candle_ts > self.entry_candle_ts
                            ):
                                if current_side == "LONG":
                                    self.execute_long(
                                        exchange,
                                        symbol,
                                        price,
                                        candle_ts=current_candle_ts
                                    )
                                else:
                                    self.execute_short(
                                        exchange,
                                        symbol,
                                        price,
                                        candle_ts=current_candle_ts
                                    )

                    # First entry: only if the same direction has not just
                    # been closed. This prevents instant re-entry at the
                    # same signal/price.
                    if current_side is None:
                        if self.last_closed_side == desired_side:
                            # Wait for a genuine opposite-direction state.
                            pass
                        elif desired_side == "LONG":
                            self.execute_long(
                                exchange,
                                symbol,
                                price,
                                candle_ts=candles[-1][0]
                            )
                        else:
                            self.execute_short(
                                exchange,
                                symbol,
                                price,
                                candle_ts=candles[-1][0]
                            )

                    else:
                        # Scale in once per new candle, up to 4 total entries.
                        current_count = self.position_entries_count()
                        current_candle_ts = candles[-1][0]

                        if (
                            current_count < self.max_entries
                            and self.entry_candle_ts is not None
                            and current_candle_ts > self.entry_candle_ts
                        ):
                            if desired_side == "LONG":
                                self.execute_long(
                                    exchange,
                                    symbol,
                                    price,
                                    candle_ts=current_candle_ts
                                )
                            else:
                                self.execute_short(
                                    exchange,
                                    symbol,
                                    price,
                                    candle_ts=current_candle_ts
                                )

                    # Once the market genuinely switches to the opposite side,
                    # unlock the new direction.
                    if self.last_closed_side is not None:
                        if desired_side != self.last_closed_side:
                            self.last_closed_side = None

                    actual_side = self.current_side()
                    if actual_side == "LONG":
                        self.last_signal = "BUY"
                        self.root.after(
                            0,
                            self.signal_update,
                            "● LONG",
                            "#00FF66"
                        )
                    elif actual_side == "SHORT":
                        self.last_signal = "SELL"
                        self.root.after(
                            0,
                            self.signal_update,
                            "● SHORT",
                            "#FF3333"
                        )

                # Check target against the live ticker price.
                self.check_target(
                    exchange,
                    symbol,
                    price
                )

            except Exception as e:

                # Never terminate the strategy loop on a network/API failure.
                # Retry forever every 5 seconds until connectivity returns.
                self.root.after(
                    0,
                    self.error_status,
                    "NETWORK/API LOST - RETRYING IN 5s"
                )

                if self.mode.cget("text") == "REAL":
                    self.reconnect_real_api()

                time.sleep(5)
                continue

            time.sleep(0.25)

    # =========================================================
    # PSAR
    # =========================================================
    def calculate_psar(
        self,
        highs,
        lows,
        closes,
        step=0.02,
        maximum=0.2
    ):

        psar = [0.0] * len(closes)

        bull = True
        af = step

        ep = highs[0]
        psar[0] = lows[0]

        for i in range(1, len(closes)):

            previous_psar = psar[i - 1]

            if bull:

                psar[i] = previous_psar + af * (
                    ep - previous_psar
                )

                if i >= 2:
                    psar[i] = min(
                        psar[i],
                        lows[i - 1],
                        lows[i - 2]
                    )
                else:
                    psar[i] = min(
                        psar[i],
                        lows[i - 1]
                    )

                if lows[i] < psar[i]:

                    bull = False
                    psar[i] = ep
                    ep = lows[i]
                    af = step

                elif highs[i] > ep:

                    ep = highs[i]
                    af = min(
                        af + step,
                        maximum
                    )

            else:

                psar[i] = previous_psar + af * (
                    ep - previous_psar
                )

                if i >= 2:
                    psar[i] = max(
                        psar[i],
                        highs[i - 1],
                        highs[i - 2]
                    )
                else:
                    psar[i] = max(
                        psar[i],
                        highs[i - 1]
                    )

                if highs[i] > psar[i]:

                    bull = True
                    psar[i] = ep
                    ep = highs[i]
                    af = step

                elif lows[i] < ep:

                    ep = lows[i]
                    af = min(
                        af + step,
                        maximum
                    )

        return psar

    # =========================================================
    # FUTURES SYMBOL / POSITION HELPERS
    # =========================================================
    def get_futures_symbol(self, exchange, symbol):
        """
        Converts SOL/USDT -> SOL/USDT:USDT when the perpetual
        contract is available. Keeps an already-unified symbol unchanged.
        """
        symbol = symbol.strip()

        if symbol in exchange.markets:
            market = exchange.markets[symbol]
            if market.get("swap") or market.get("future"):
                return symbol

        candidates = [
            symbol if ":" in symbol else f"{symbol}:USDT",
            symbol.replace("/USDT", "/USDT:USDT")
        ]

        for candidate in candidates:
            if candidate in exchange.markets:
                market = exchange.markets[candidate]
                if market.get("swap") or market.get("future"):
                    return candidate

        # Fallback: find the first USDT perpetual with the same base/quote.
        base = symbol.split("/")[0].upper()
        for market_symbol, market in exchange.markets.items():
            if (
                market.get("swap")
                and market.get("base", "").upper() == base
                and market.get("quote", "").upper() == "USDT"
                and market.get("settle", "").upper() == "USDT"
            ):
                return market_symbol

        return symbol

    def current_side(self):
        if self.mode.cget("text") == "DEMO":
            return self.demo_side

        return self.real_side

    def clear_position_state(self):
        self.entry_price = 0.0
        self.demo_position = 0.0
        self.demo_entry = 0.0
        self.demo_side = None
        self.demo_capital_deployed = 0.0
        self.demo_entries_count = 0

        self.real_position_amount = 0.0
        self.real_entry_cost = 0.0
        self.real_entry_price = 0.0
        self.real_side = None
        self.real_entries_count = 0

        self.entry_candle_ts = None

    def tranche_capital(self):
        if self.mode.cget("text") == "DEMO" and self.demo_compound:
            # Each new trade cycle uses the current compounded demo balance.
            capital = max(0.0, float(self.demo_balance))
        else:
            capital = float(self.cap.get())

        return capital / self.max_entries

    def position_entries_count(self):
        if self.mode.cget("text") == "DEMO":
            return self.demo_entries_count
        return self.real_entries_count

    # =========================================================
    # LONG (4 x 25%)
    # =========================================================
    def execute_long(self, exchange, symbol, price, candle_ts=None):
        try:
            tranche = self.tranche_capital()

            if tranche <= 0:
                return

            if self.mode.cget("text") == "DEMO":

                if self.demo_side not in (None, "LONG"):
                    return

                if self.demo_entries_count >= self.max_entries:
                    return

                if tranche > self.demo_balance:
                    self.root.after(
                        0,
                        self.error_status,
                        "INSUFFICIENT DEMO BALANCE"
                    )
                    return

                amount = tranche / price

                # Weighted average entry across all LONG tranches.
                old_amount = self.demo_position
                new_amount = old_amount + amount

                if old_amount > 0:
                    self.demo_entry = (
                        (self.demo_entry * old_amount)
                        + (price * amount)
                    ) / new_amount
                else:
                    self.demo_entry = price

                self.demo_balance -= tranche
                self.demo_position = new_amount
                self.demo_side = "LONG"
                self.demo_capital_deployed += tranche
                self.demo_entries_count += 1
                self.entry_price = self.demo_entry

                self.entry_candle_ts = candle_ts

                self.root.after(
                    0,
                    self.update_position,
                    self.demo_capital_deployed
                )

            else:

                market = exchange.market(symbol)
                contract_size = float(market.get("contractSize") or 1)

                contracts = tranche / (price * contract_size)

                contracts = float(
                    exchange.amount_to_precision(
                        symbol,
                        contracts
                    )
                )

                if contracts <= 0:
                    raise RuntimeError("LONG ORDER SIZE IS ZERO")

                order = exchange.create_order(
                    symbol,
                    "market",
                    "buy",
                    contracts,
                    None,
                    {}
                )

                filled = float(order.get("filled") or contracts)

                avg_price = float(
                    order.get("average")
                    or (
                        order.get("cost", 0) / filled
                        if filled else price
                    )
                    or price
                )

                if filled <= 0:
                    raise RuntimeError("LONG ORDER RETURNED ZERO FILL")

                old_amount = self.real_position_amount
                new_amount = old_amount + filled

                if old_amount > 0:
                    self.real_entry_price = (
                        (self.real_entry_price * old_amount)
                        + (avg_price * filled)
                    ) / new_amount
                else:
                    self.real_entry_price = avg_price

                order_cost = float(
                    order.get("cost")
                    or (filled * contract_size * avg_price)
                )

                self.real_position_amount = new_amount
                self.real_entry_cost += order_cost
                self.real_side = "LONG"
                self.real_entries_count += 1
                self.entry_price = self.real_entry_price
                self.entry_candle_ts = candle_ts

                self.root.after(
                    0,
                    self.update_position,
                    self.real_entry_cost
                )

        except Exception as e:
            self.root.after(
                0,
                self.error_status,
                str(e)
            )

    # =========================================================
    # SHORT (4 x 25%)
    # =========================================================
    def execute_short(self, exchange, symbol, price, candle_ts=None):
        try:
            tranche = self.tranche_capital()

            if tranche <= 0:
                return

            if self.mode.cget("text") == "DEMO":

                if self.demo_side not in (None, "SHORT"):
                    return

                if self.demo_entries_count >= self.max_entries:
                    return

                if tranche > self.demo_balance:
                    self.root.after(
                        0,
                        self.error_status,
                        "INSUFFICIENT DEMO BALANCE"
                    )
                    return

                amount = tranche / price

                # Weighted average entry across all SHORT tranches.
                old_amount = self.demo_position
                new_amount = old_amount + amount

                if old_amount > 0:
                    self.demo_entry = (
                        (self.demo_entry * old_amount)
                        + (price * amount)
                    ) / new_amount
                else:
                    self.demo_entry = price

                self.demo_balance -= tranche
                self.demo_position = new_amount
                self.demo_side = "SHORT"
                self.demo_capital_deployed += tranche
                self.demo_entries_count += 1
                self.entry_price = self.demo_entry

                self.entry_candle_ts = candle_ts

                self.root.after(
                    0,
                    self.update_position,
                    self.demo_capital_deployed
                )

            else:

                market = exchange.market(symbol)
                contract_size = float(market.get("contractSize") or 1)

                contracts = tranche / (price * contract_size)

                contracts = float(
                    exchange.amount_to_precision(
                        symbol,
                        contracts
                    )
                )

                if contracts <= 0:
                    raise RuntimeError("SHORT ORDER SIZE IS ZERO")

                order = exchange.create_order(
                    symbol,
                    "market",
                    "sell",
                    contracts,
                    None,
                    {}
                )

                filled = float(order.get("filled") or contracts)

                avg_price = float(
                    order.get("average")
                    or (
                        order.get("cost", 0) / filled
                        if filled else price
                    )
                    or price
                )

                if filled <= 0:
                    raise RuntimeError("SHORT ORDER RETURNED ZERO FILL")

                old_amount = self.real_position_amount
                new_amount = old_amount + filled

                if old_amount > 0:
                    self.real_entry_price = (
                        (self.real_entry_price * old_amount)
                        + (avg_price * filled)
                    ) / new_amount
                else:
                    self.real_entry_price = avg_price

                order_cost = float(
                    order.get("cost")
                    or (filled * contract_size * avg_price)
                )

                self.real_position_amount = new_amount
                self.real_entry_cost += order_cost
                self.real_side = "SHORT"
                self.real_entries_count += 1
                self.entry_price = self.real_entry_price
                self.entry_candle_ts = candle_ts

                self.root.after(
                    0,
                    self.update_position,
                    self.real_entry_cost
                )

        except Exception as e:
            self.root.after(
                0,
                self.error_status,
                str(e)
            )

    # =========================================================
    # CLOSE CURRENT POSITION
    # =========================================================
    def execute_close(self, exchange, symbol, price):
        try:
            side = self.current_side()

            if side is None:
                return

            if self.mode.cget("text") == "DEMO":

                amount = self.demo_position
                entry = self.demo_entry

                if amount <= 0 or entry <= 0:
                    self.clear_position_state()
                    return

                if side == "LONG":
                    profit = (price - entry) * amount
                else:
                    profit = (entry - price) * amount

                self.demo_balance += self.demo_capital_deployed + profit

                self.trade_count += 1
                self.total_profit += profit
                self.last_trade_profit = profit

                if profit > 0:
                    self.win_count += 1

                closed_side = side
                self.last_closed_side = closed_side
                self.clear_position_state()

                self.queue_notification(
                    f"{closed_side} • P&L: {profit:+.4f} USDT"
                )

                self.root.after(
                    0,
                    self.update_profit,
                    profit
                )

            else:

                amount = float(self.real_position_amount)

                if amount <= 0:
                    return

                close_side = "sell" if side == "LONG" else "buy"

                order = exchange.create_order(
                    symbol,
                    "market",
                    close_side,
                    amount,
                    None,
                    {"reduceOnly": True}
                )

                filled = float(order.get("filled") or amount)

                avg_price = float(
                    order.get("average")
                    or (
                        order.get("cost", 0) / filled
                        if filled else price
                    )
                    or price
                )

                entry_price = float(self.real_entry_price)

                if side == "LONG":
                    profit = (avg_price - entry_price) * filled
                else:
                    profit = (entry_price - avg_price) * filled

                self.trade_count += 1
                self.total_profit += profit
                self.last_trade_profit = profit

                if profit > 0:
                    self.win_count += 1

                closed_side = side
                self.last_closed_side = closed_side
                self.clear_position_state()

                self.queue_notification(
                    f"{closed_side} • P&L: {profit:+.4f} USDT"
                )

                self.root.after(
                    0,
                    self.update_profit,
                    profit
                )

        except Exception as e:
            self.root.after(
                0,
                self.error_status,
                str(e)
            )

    # =========================================================
    # TARGET
    # =========================================================

    # =========================================================
    def check_target(
        self,
        exchange,
        symbol,
        price
    ):
        """
        Close immediately when the REAL/LIVE ticker reaches the exact
        target percentage.

        TARGET % = percentage, so:
          0.0001 = 0.0001%
          0.01   = 0.01%
          0.50   = 0.50%

        Uses a small floating-point tolerance and checks continuously.
        """
        try:
            side = self.current_side()

            if side is None:
                return

            entry = (
                self.real_entry_price
                if self.mode.cget("text") == "REAL"
                else self.demo_entry
            )

            if entry <= 0:
                return

            target = float(self.tgt.get())

            if target < 0:
                return

            # Convert percentage to decimal.
            target_fraction = target / 100.0

            if side == "LONG":
                target_price = entry * (1.0 + target_fraction)
                hit = price >= target_price
            else:
                target_price = entry * (1.0 - target_fraction)
                hit = price <= target_price

            # Update UI with the exact target price.
            self.root.after(
                0,
                self.position_l.config,
                {
                    "text": (
                        f"{side} • ENTRY ${entry:,.8f}"
                        f" • TARGET ${target_price:,.8f}"
                        f" • {self.position_entries_count()}/4"
                    ),
                    "fg": "#00FF66" if side == "LONG" else "#FF3333"
                }
            )

            if hit:
                # Prevent the strategy from opening another order while
                # the closing order is being processed.
                was_running = self.running
                self.running = False

                try:
                    self.execute_close(
                        exchange,
                        symbol,
                        price
                    )
                finally:
                    if was_running:
                        self.running = True

                self.last_signal = None

        except Exception as e:
            self.root.after(
                0,
                self.error_status,
                f"TARGET ERROR: {str(e)}"
            )

    # =========================================================
    # MILLION TARGET
    # =========================================================
    def update_million_progress(self):
        remaining = max(0.0, self.million_target - self.total_profit)

        if self.total_profit >= self.million_target:
            self.million_l.config(
                text="TARGET REACHED: $1,000,000.00",
                fg="#00FF66"
            )
            self.trades_left_l.config(
                text="Trades to $1M: 0",
                fg="#00FF66"
            )
            return

        self.million_l.config(
            text=f"To $1M: ${remaining:,.2f}",
            fg="#FFFFFF"
        )

        # Estimate remaining trades from the average realized profit per
        # completed trade. This is only an estimate, not a guarantee.
        if self.trade_count > 0 and self.total_profit > 0:
            avg_profit = self.total_profit / self.trade_count
            trades_left = int((remaining / avg_profit) + 0.999999)
            self.trades_left_l.config(
                text=f"Trades to $1M: ~{trades_left} "
                     f"(avg ${avg_profit:,.4f}/trade)",
                fg="#FFFFFF"
            )
        else:
            self.trades_left_l.config(
                text="Trades to $1M: --",
                fg="#FFFFFF"
            )

    # =========================================================
    # UI UPDATES
    # =========================================================
    def update_price(self, price):
        try:
            price = float(price)
        except (TypeError, ValueError):
            return

        # Network price becomes the target; the UI moves toward it smoothly.
        self.target_display_price = price

        if self.display_price is None:
            self.display_price = price
            self.price_l.config(
                text=f"${price:,.8f}"
            )

    def signal_update(self, text, color):

        self.sig_l.config(
            text=text,
            fg=color
        )

        if "LONG" in text and self.entry_price > 0:
            self.position_l.config(
                text=(
                    f"LONG • ENTRY ${self.entry_price:,.6f}"
                    f" • {self.position_entries_count()}/4"
                ),
                fg="#00FF66"
            )
        elif "SHORT" in text and self.entry_price > 0:
            self.position_l.config(
                text=(
                    f"SHORT • ENTRY ${self.entry_price:,.6f}"
                    f" • {self.position_entries_count()}/4"
                ),
                fg="#FF3333"
            )
        else:
            self.position_l.config(
                text="",
                fg="#FFFFFF"
            )

    def update_position(self, value):

        self.active.config(
            text=f"Act: ${value:.2f}"
        )

        if self.entry_price > 0:
            side = self.current_side() or "POSITION"
            count = self.position_entries_count()
            self.position_l.config(
                text=f"{side} • ENTRY ${self.entry_price:,.6f} • {count}/4",
                fg="#00FF66" if side == "LONG" else "#FF3333"
            )

    def update_profit(self, profit):

        self.last_trade_profit = profit
        self.pft.config(
            text=f"TOTAL: {self.total_profit:+.2f} USDT",
            fg="#00FF66" if self.total_profit >= 0 else "#FF3333"
        )

        if hasattr(self, "live_pnl_l"):
            self.live_pnl_l.config(
                text="Live P&L: +0.0000 USDT",
                fg="#FFFFFF"
            )

        self.update_million_progress()

        self.active.config(
            text="Act: $0"
        )

        if self.mode.cget("text") == "DEMO":
            self.vault_l.config(
                text=f"Vlt: ${self.demo_balance:.2f}"
            )
        else:
            self.vault_l.config(
                text=f"P&L: {self.total_profit:+.2f}"
            )

    def error_status(self, error):

        msg = str(error).replace("\n", " ")
        if len(msg) > 34:
            msg = msg[:31] + "..."

        self.status.config(
            text=f"● ERROR: {msg}",
            fg="#FF3333"
        )

    # =========================================================
    # RUN
    # =========================================================
if __name__ == "__main__":

    root = tk.Tk()

    app = DVYUDesign(root)

    root.mainloop()
