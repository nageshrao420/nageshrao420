import backtrader as bt
import math
import numpy as np
import pandas as pd

class BitcoinBreakoutStrategy(bt.Strategy):
    # User parameters
    lengthBB = 20  # Bollinger Bands period
    multBB = 2.0  # Bollinger Bands multiplier
    rsiLength = 14  # RSI period
    rsiOverbought = 70  # RSI Overbought level
    rsiOversold = 30  # RSI Oversold level
    riskRewardRatio = 2.0  # Risk-Reward Ratio
    atrLength = 14  # ATR period
    riskPercent = 1.0  # Risk percent per trade
    emaShortPeriod = 9  # Short EMA period
    emaLongPeriod = 21  # Long EMA period

    def __init__(self):
        # Calculate Bollinger Bands
        self.bb = bt.indicators.BollingerBands(self.data.close, period=self.lengthBB, dev=self.multBB)

        # Calculate RSI
        self.rsi = bt.indicators.RSI(self.data.close, period=self.rsiLength)

        # Calculate ATR
        self.atr = bt.indicators.AverageTrueRange(self.data, period=self.atrLength)

        # Calculate EMA
        self.emaShort = bt.indicators.ExponentialMovingAverage(self.data.close, period=self.emaShortPeriod)
        self.emaLong = bt.indicators.ExponentialMovingAverage(self.data.close, period=self.emaLongPeriod)

        # Calculate MACD
        self.macd = bt.indicators.MACD(self.data.close)

    def next(self):
        # Entry logic for long position (Breakout Upwards)
        if self.data.close[0] > self.bb.lines.bot[0] and \
           self.rsi[0] < self.rsiOverbought and \
           self.emaShort[0] > self.emaLong[0] and \
           self.macd.macd[0] > self.macd.signal[0]:

            if not self.position:  # if no position is open
                risk = self.riskPercent / 100 * self.broker.get_value()
                position_size = risk / self.atr[0]
                stop_loss = self.data.close[0] - self.atr[0]
                take_profit = self.data.close[0] + (self.riskRewardRatio * self.atr[0])

                self.buy(size=position_size)  # Enter Long Position
                self.sell(exectype=bt.Order.Stop, price=stop_loss)  # Stop Loss
                self.sell(exectype=bt.Order.Limit, price=take_profit)  # Take Profit

        # Entry logic for short position (Breakout Downwards)
        elif self.data.close[0] < self.bb.lines.top[0] and \
             self.rsi[0] > self.rsiOversold and \
             self.emaShort[0] < self.emaLong[0] and \
             self.macd.macd[0] < self.macd.signal[0]:

            if not self.position:  # if no position is open
                risk = self.riskPercent / 100 * self.broker.get_value()
                position_size = risk / self.atr[0]
                stop_loss = self.data.close[0] + self.atr[0]
                take_profit = self.data.close[0] - (self.riskRewardRatio * self.atr[0])

                self.sell(size=position_size)  # Enter Short Position
                self.buy(exectype=bt.Order.Stop, price=stop_loss)  # Stop Loss
                self.buy(exectype=bt.Order.Limit, price=take_profit)  # Take Profit

    def notify_order(self, order):
        if order.status in [bt.Order.Submitted, bt.Order.Accepted]:
            return

        if order.status in [bt.Order.Completed]:
            if order.isbuy():
                print(f"Bought {order.size} at {order.executed.price}")
            elif order.issell():
                print(f"Sold {order.size} at {order.executed.price}")

        elif order.status in [bt.Order.Canceled, bt.Order.Margin, bt.Order.Rejected]:
            print(f"Order failed: {order.getstatusname()}")

# Create a Cerebro instance and set up the strategy
cerebro = bt.Cerebro()
cerebro.addstrategy(BitcoinBreakoutStrategy)

# Data (Load your Bitcoin dataset - example CSV)
data = bt.feeds.YahooFinanceData(dataname='path_to_bitcoin_data.csv', fromdate=pd.Timestamp('2021-01-01'),
                                 todate=pd.Timestamp('2022-01-01'))

cerebro.adddata(data)

# Set the starting cash, commission (for simulation)
cerebro.broker.set_cash(10000)
cerebro.broker.set_commission(commission=0.001)

# Set slippage (optional)
cerebro.broker.set_slippage_perc(0.001)

# Run the backtest
cerebro.run()

# Print final portfolio value
print(f"Final Portfolio Value: {cerebro.broker.get_value()}")

# Plot the result
cerebro.plot()

