from AlgorithmImports import *

class BinanceTOBImbalanceAlgo(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2023, 4, 24)
        self.SetEndDate(2023, 5, 24)
        self.SetCash(10000)

        self.symbol = self.AddCrypto("BTCUSDT", Resolution.Tick, Market.Binance).Symbol

        self.imbalance_threshold = 0.4
        self.position_size = 0.02  # ← Aggressive exposure (~$1250)
        self.cooldown = timedelta(minutes=5)
        self.last_trade_time = None

        self.AddAlpha(NullAlphaModel())

        # For Sharpe ratio calculation
        self.daily_returns = []
        self.last_portfolio_value = self.Portfolio.TotalPortfolioValue
        self.last_day = self.Time.date()

    def OnData(self, data):
        if not data.Ticks.ContainsKey(self.symbol):
            return

        for tick in data.Ticks[self.symbol]:
            if tick.TickType != TickType.Quote:
                continue

            bid_size = tick.BidSize
            ask_size = tick.AskSize

            if bid_size == 0 and ask_size == 0:
                continue

            total = bid_size + ask_size
            imbalance = (bid_size - ask_size) / total if total > 0 else 0

            signal = 0
            if imbalance > self.imbalance_threshold:
                signal = 1
            elif imbalance < -self.imbalance_threshold:
                signal = -1

            if signal == 0:
                continue

            if self.last_trade_time and self.Time - self.last_trade_time < self.cooldown:
                continue

            invested = self.Portfolio[self.symbol].Invested

            self.Log(f"[SIGNAL] {self.Time} | Signal: {signal} | Imbalance: {imbalance:.2f}")

            if signal == 1 and not invested:
                self.MarketOrder(self.symbol, self.position_size)
                self.last_trade_time = self.Time
            elif signal == -1 and invested:
                self.Liquidate(self.symbol)
                self.last_trade_time = self.Time

        # Track daily return
        if self.Time.date() != self.last_day:
            pnl = self.Portfolio.TotalPortfolioValue - self.last_portfolio_value
            daily_return = pnl / self.last_portfolio_value if self.last_portfolio_value > 0 else 0
            self.daily_returns.append(daily_return)
            self.last_portfolio_value = self.Portfolio.TotalPortfolioValue
            self.last_day = self.Time.date()

    def OnOrderEvent(self, orderEvent):
        if orderEvent.Status != OrderStatus.Filled:
            return

        order = self.Transactions.GetOrderById(orderEvent.OrderId)
        direction = "BUY" if order.Direction == OrderDirection.Buy else "SELL"
        quantity = orderEvent.FillQuantity
        fill_price = orderEvent.FillPrice

        self.Log(
            f"[TRADE] {self.Time} | {direction} {quantity} {order.Symbol.Value} @ {fill_price:.2f} | "
            f"Portfolio: {self.Portfolio[order.Symbol].Quantity}"
        )

    def OnEndOfAlgorithm(self):
        if len(self.daily_returns) < 2:
            self.Log("Not enough data to compute Sharpe.")
            return

        mean_return = sum(self.daily_returns) / len(self.daily_returns)
        variance = sum((r - mean_return) ** 2 for r in self.daily_returns) / (len(self.daily_returns) - 1)
        std_dev = variance ** 0.5
        sharpe = (mean_return / std_dev) * (252 ** 0.5) if std_dev > 0 else 0

        total_return = (self.Portfolio.TotalPortfolioValue - 10000) / 10000

        self.Log("--- Strategy Performance ---")
        self.Log(f"Total Return: {total_return * 100:.2f}%")
        self.Log(f"Sharpe Ratio: {sharpe:.2f}")
