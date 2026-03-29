This is a fantastic dataset to work with. Let me give you a comprehensive, practical roadmap tailored specifically to the **Colombo Stock Exchange (CSE)** and its unique characteristics.

#claude #Opus-4_6

---

## 1. Understanding the CSE's Unique Challenges First

Before jumping into models, you need to internalize these realities:

```
CSE-SPECIFIC CHALLENGES:
├── Low liquidity (thin trading in many stocks)
├── High volatility (currency crises, political instability)
├── Small number of actively traded stocks (~100 liquid ones)
├── Market manipulation in small caps
├── Foreign fund flow dependency
├── Rupee depreciation impact
├── Concentrated sectors (banks, plantations, conglomerates)
├── Regime changes in monetary/fiscal policy
└── Limited short-selling (can't easily profit from downside)
```

---

## 2. Data Preparation Pipeline (CRITICAL FIRST STEP)

```python
import pandas as pd
import numpy as np

# ============================================
# STEP 1: CLEAN YOUR DATA PROPERLY
# ============================================

class CSEDataPipeline:
    """
    Sri Lankan market data has unique issues:
    - Trading halts (2019 Easter attacks, COVID, 2022 crisis)
    - Stock splits, rights issues, bonus issues
    - Dividend adjustments
    - Thin trading days with zero volume
    """
    
    def __init__(self, price_data, financial_data):
        self.prices = price_data
        self.financials = financial_data
    
    def filter_liquid_stocks(self, min_avg_daily_volume=1_000_000,
                              min_trading_days_pct=0.80):
        """
        CRITICAL FOR CSE: Remove illiquid stocks.
        Many CSE stocks trade < 10 times a month.
        Trading illiquid stocks = guaranteed slippage losses.
        """
        trading_days = self.prices.groupby('ticker').agg(
            total_days=('volume', 'count'),
            active_days=('volume', lambda x: (x > 0).sum()),
            avg_volume=('volume', 'mean'),
            avg_turnover=('turnover', 'mean')  # LKR value traded
        )
        
        trading_days['active_pct'] = (
            trading_days['active_days'] / trading_days['total_days']
        )
        
        liquid = trading_days[
            (trading_days['avg_turnover'] > min_avg_daily_volume) &
            (trading_days['active_pct'] > min_trading_days_pct)
        ].index.tolist()
        
        print(f"Liquid stocks: {len(liquid)} out of "
              f"{trading_days.shape[0]} total")
        return liquid
    
    def adjust_for_corporate_actions(self, ticker_data):
        """Handle splits, bonuses, rights issues common in CSE"""
        # Use adjustment factor if available, or calculate
        pass
    
    def handle_crisis_periods(self):
        """
        Flag or handle special periods:
        - 2008-2009: GFC + end of civil war rally
        - 2019 April: Easter Sunday attacks
        - 2020 March: COVID crash + market closure
        - 2022: Economic crisis, sovereign default
        - 2023-2024: Recovery period
        """
        crisis_periods = {
            'easter_attacks': ('2019-04-21', '2019-05-15'),
            'covid_closure': ('2020-03-20', '2020-05-11'),
            'economic_crisis': ('2022-03-01', '2022-12-31'),
        }
        return crisis_periods
    
    def compute_features(self, df):
        """Compute features for each stock"""
        # Price-based
        df['returns'] = df['close'].pct_change()
        df['log_returns'] = np.log(df['close'] / df['close'].shift(1))
        
        # Volatility (crucial for CSE)
        df['volatility_20d'] = df['returns'].rolling(20).std() * np.sqrt(252)
        df['volatility_60d'] = df['returns'].rolling(60).std() * np.sqrt(252)
        
        # Volume features
        df['volume_ratio'] = df['volume'] / df['volume'].rolling(20).mean()
        df['turnover_ratio'] = df['turnover'] / df['turnover'].rolling(20).mean()
        
        # Technical
        df['sma_50'] = df['close'].rolling(50).mean()
        df['sma_200'] = df['close'].rolling(200).mean()
        df['rsi_14'] = self._compute_rsi(df['close'], 14)
        
        # Momentum
        df['momentum_1m'] = df['close'] / df['close'].shift(21) - 1
        df['momentum_3m'] = df['close'] / df['close'].shift(63) - 1
        df['momentum_6m'] = df['close'] / df['close'].shift(126) - 1
        df['momentum_12m'] = df['close'] / df['close'].shift(252) - 1
        
        return df
    
    def compute_fundamental_features(self, ticker, date):
        """
        From your financial reports database.
        Use TRAILING data only (no look-ahead bias!)
        """
        fin = self.financials[
            (self.financials['ticker'] == ticker) &
            (self.financials['report_date'] <= date)
        ].iloc[-1]  # Most recent report BEFORE date
        
        features = {
            'pe_ratio': fin.get('pe_ratio'),
            'pb_ratio': fin.get('pb_ratio'),
            'roe': fin.get('roe'),
            'roa': fin.get('roa'),
            'debt_to_equity': fin.get('debt_to_equity'),
            'current_ratio': fin.get('current_ratio'),
            'dividend_yield': fin.get('dividend_yield'),
            'earnings_growth': fin.get('earnings_growth_yoy'),
            'revenue_growth': fin.get('revenue_growth_yoy'),
            'net_profit_margin': fin.get('net_profit_margin'),
            'operating_cash_flow': fin.get('operating_cf'),
            'free_cash_flow': fin.get('free_cf'),
            # CSE-specific: many companies have forex exposure
            'forex_revenue_pct': fin.get('forex_revenue_pct'),
        }
        return features
    
    @staticmethod
    def _compute_rsi(series, period):
        delta = series.diff()
        gain = delta.where(delta > 0, 0).rolling(period).mean()
        loss = (-delta.where(delta < 0, 0)).rolling(period).mean()
        rs = gain / loss
        return 100 - (100 / (1 + rs))
```

---

## 3. ALGORITHMS & MODELS (Ordered by Practicality for CSE)

### TIER 1: Start Here (Highest Probability of Working)

#### A. Factor-Based / Smart Beta Models

```python
# ============================================
# MULTI-FACTOR RANKING MODEL
# This is your BEST starting point for CSE
# ============================================

class FactorModel:
    """
    Why this works for CSE:
    - Doesn't need massive data like deep learning
    - Interpretable (you know WHY you're buying)
    - Robust in small markets
    - Academic evidence across emerging markets
    
    Key factors that work in emerging markets:
    1. Value (low P/E, low P/B, high dividend yield)
    2. Quality (high ROE, low debt, stable earnings)
    3. Momentum (6-12 month price momentum)
    4. Low Volatility (paradoxically outperforms in CSE!)
    5. Size (small-cap premium, but BE CAREFUL with liquidity)
    """
    
    def __init__(self, universe):
        self.universe = universe  # List of liquid tickers
    
    def compute_factor_scores(self, date, price_data, fundamental_data):
        scores = pd.DataFrame(index=self.universe)
        
        # === VALUE FACTOR ===
        scores['value_pe'] = self._rank_ascending(
            fundamental_data.loc[self.universe, 'pe_ratio']
        )  # Lower P/E = higher rank
        
        scores['value_pb'] = self._rank_ascending(
            fundamental_data.loc[self.universe, 'pb_ratio']
        )
        
        scores['value_dy'] = self._rank_descending(
            fundamental_data.loc[self.universe, 'dividend_yield']
        )
        
        scores['value_combined'] = (
            scores['value_pe'] + scores['value_pb'] + scores['value_dy']
        ) / 3
        
        # === QUALITY FACTOR ===
        scores['quality_roe'] = self._rank_descending(
            fundamental_data.loc[self.universe, 'roe']
        )
        
        scores['quality_debt'] = self._rank_ascending(
            fundamental_data.loc[self.universe, 'debt_to_equity']
        )
        
        scores['quality_margin'] = self._rank_descending(
            fundamental_data.loc[self.universe, 'net_profit_margin']
        )
        
        # Earnings stability (std of earnings over past 5 years)
        scores['quality_stability'] = self._rank_ascending(
            fundamental_data.loc[self.universe, 'earnings_std_5y']
        )
        
        scores['quality_combined'] = (
            scores['quality_roe'] + scores['quality_debt'] + 
            scores['quality_margin'] + scores['quality_stability']
        ) / 4
        
        # === MOMENTUM FACTOR ===
        # 12-1 momentum (skip most recent month - reversal effect)
        scores['momentum'] = self._rank_descending(
            price_data.loc[self.universe, 'momentum_12m'] - 
            price_data.loc[self.universe, 'momentum_1m']
        )
        
        # === LOW VOLATILITY FACTOR ===
        # CRUCIAL FOR CSE - low vol stocks outperform
        scores['low_vol'] = self._rank_ascending(
            price_data.loc[self.universe, 'volatility_60d']
        )
        
        # === COMPOSITE SCORE ===
        # Weight factors (tune these based on your backtest)
        scores['composite'] = (
            0.30 * scores['value_combined'] +
            0.30 * scores['quality_combined'] +
            0.20 * scores['momentum'] +
            0.20 * scores['low_vol']
        )
        
        return scores.sort_values('composite', ascending=False)
    
    def construct_portfolio(self, scores, top_n=15, 
                           weighting='equal'):
        """
        Select top N stocks by composite score.
        
        For CSE: 
        - Don't hold too many (15-25 is enough)
        - Equal weight often beats cap-weight in small markets
        - Rebalance monthly or quarterly (lower frequency = lower costs)
        """
        selected = scores.head(top_n).index.tolist()
        
        if weighting == 'equal':
            weights = {s: 1.0/top_n for s in selected}
        elif weighting == 'score':
            total = scores.head(top_n)['composite'].sum()
            weights = {
                s: scores.loc[s, 'composite'] / total 
                for s in selected
            }
        elif weighting == 'inverse_vol':
            # Risk parity lite - great for volatile CSE
            vols = scores.head(top_n)['low_vol']
            inv_vol = 1.0 / vols
            weights = {
                s: inv_vol[s] / inv_vol.sum() 
                for s in selected
            }
        
        return weights
    
    @staticmethod
    def _rank_descending(series):
        return series.rank(pct=True)
    
    @staticmethod
    def _rank_ascending(series):
        return (1 - series.rank(pct=True))
```

#### B. Mean-Variance with Robust Estimation (Portfolio Optimization)

```python
# ============================================
# PORTFOLIO OPTIMIZATION FOR CSE
# Standard Markowitz FAILS in volatile markets
# Use robust/shrinkage estimators instead
# ============================================

from scipy.optimize import minimize

class RobustPortfolioOptimizer:
    """
    Why robust optimization for CSE:
    - Standard covariance estimation is garbage with CSE volatility
    - Ledoit-Wolf shrinkage dramatically improves estimates
    - Add constraints to prevent concentration
    """
    
    def __init__(self, returns_df, risk_free_rate=0.10):
        # Sri Lanka risk-free rate ~ 10% (T-bill rate, varies)
        self.returns = returns_df
        self.rf = risk_free_rate / 252  # Daily
        
    def ledoit_wolf_shrinkage(self):
        """
        Shrink sample covariance toward structured target.
        Much more stable than sample covariance.
        """
        from sklearn.covariance import LedoitWolf
        lw = LedoitWolf().fit(self.returns.dropna())
        return pd.DataFrame(
            lw.covariance_,
            index=self.returns.columns,
            columns=self.returns.columns
        )
    
    def minimum_variance_portfolio(self, cov_matrix, 
                                     max_weight=0.10,
                                     sector_constraints=None):
        """
        Minimum variance portfolio - IDEAL for CSE.
        You don't need to estimate expected returns 
        (which are nearly impossible to estimate).
        """
        n = len(cov_matrix)
        
        def portfolio_variance(weights):
            return weights @ cov_matrix.values @ weights
        
        constraints = [
            {'type': 'eq', 'fun': lambda w: np.sum(w) - 1.0}
        ]
        
        # Individual position limits
        bounds = [(0.02, max_weight)] * n  # Min 2%, max 10%
        
        # Sector constraints (important for CSE - 
        # don't load up on just banks!)
        if sector_constraints:
            for sector, max_alloc in sector_constraints.items():
                sector_mask = [
                    1 if s in sector_constraints[sector]['tickers'] 
                    else 0 
                    for s in cov_matrix.columns
                ]
                constraints.append({
                    'type': 'ineq',
                    'fun': lambda w, m=sector_mask, ma=max_alloc: 
                        ma - np.dot(w, m)
                })
        
        result = minimize(
            portfolio_variance,
            x0=np.ones(n) / n,
            method='SLSQP',
            bounds=bounds,
            constraints=constraints
        )
        
        return pd.Series(result.x, index=cov_matrix.columns)
    
    def hierarchical_risk_parity(self, cov_matrix):
        """
        HRP by Marcos López de Prado.
        Works MUCH better than Markowitz for:
        - Small samples
        - Noisy covariance matrices
        - Volatile markets (CSE!)
        
        Doesn't require expected return estimates.
        """
        # Use the riskfolio-lib or implement from scratch
        # pip install riskfolio-lib
        import riskfolio as rp
        
        port = rp.HCPortfolio(returns=self.returns)
        weights = port.optimization(
            model='HRP',
            codependence='pearson',
            rm='MV',  # risk measure
            rf=self.rf,
            linkage='single',
            leaf_order=True
        )
        return weights
```

### TIER 2: Machine Learning Models

#### C. Gradient Boosting (XGBoost / LightGBM)

```python
# ============================================
# GRADIENT BOOSTING FOR STOCK PREDICTION
# Best ML model for tabular financial data
# ============================================

import lightgbm as lgb
from sklearn.model_selection import TimeSeriesSplit

class StockPredictorML:
    """
    Predict: Forward 1-month return quintile (classification)
    or Forward 1-month return (regression)
    
    Classification often works better than regression
    for noisy financial data.
    """
    
    def __init__(self):
        self.feature_cols = []
        self.model = None
        
    def prepare_features(self, price_data, fundamental_data):
        """
        Feature engineering - THE MOST IMPORTANT PART.
        
        Categories of features:
        1. Technical / Price-based
        2. Fundamental / Financial statement
        3. Macro features (crucial for Sri Lanka!)
        4. Cross-sectional features (relative to market)
        """
        features = pd.DataFrame()
        
        # --- TECHNICAL FEATURES ---
        technical = [
            'rsi_14', 'volatility_20d', 'volatility_60d',
            'momentum_1m', 'momentum_3m', 'momentum_6m',
            'momentum_12m', 'volume_ratio', 'turnover_ratio',
        ]
        
        # Moving average distances
        features['dist_sma50'] = (
            price_data['close'] / price_data['sma_50'] - 1
        )
        features['dist_sma200'] = (
            price_data['close'] / price_data['sma_200'] - 1
        )
        
        # Drawdown from 52-week high
        features['drawdown_52w'] = (
            price_data['close'] / 
            price_data['close'].rolling(252).max() - 1
        )
        
        # --- FUNDAMENTAL FEATURES ---
        fundamental = [
            'pe_ratio', 'pb_ratio', 'roe', 'roa',
            'debt_to_equity', 'current_ratio', 'dividend_yield',
            'earnings_growth', 'revenue_growth',
            'net_profit_margin', 'operating_cash_flow',
        ]
        
        # --- MACRO FEATURES (CRITICAL FOR SRI LANKA) ---
        macro = {
            'usd_lkr_change_1m': 'USD/LKR 1-month change',
            'tbill_rate': '91-day T-bill rate',
            'inflation_yoy': 'Year-over-year inflation',
            'aspi_return_1m': 'ASPI index 1-month return',
            'foreign_net_flow': 'Foreign investor net buying',
            'cbsl_policy_rate': 'Central bank policy rate',
            'oil_price_change': 'Brent crude change (import cost)',
            'remittance_flow': 'Worker remittance data',
        }
        
        # --- CROSS-SECTIONAL FEATURES ---
        # Rank within universe at each point in time
        features['pe_rank'] = (
            fundamental_data['pe_ratio']
            .groupby(level='date')
            .rank(pct=True)
        )
        features['momentum_rank'] = (
            price_data['momentum_6m']
            .groupby(level='date')
            .rank(pct=True)
        )
        
        # --- TARGET VARIABLE ---
        # Forward 1-month return
        features['target_return'] = (
            price_data['close'].shift(-21) / price_data['close'] - 1
        )
        
        # Or classify into quintiles
        features['target_quintile'] = (
            features['target_return']
            .groupby(level='date')
            .apply(lambda x: pd.qcut(x, 5, labels=[0,1,2,3,4]))
        )
        
        return features
    
    def train_model(self, features, target_col='target_quintile'):
        """
        CRITICAL: Use proper time-series cross-validation!
        Never use random train/test split with time series.
        """
        
        # Purged walk-forward cross-validation
        # Train on past, predict future, never look ahead
        
        tscv = TimeSeriesSplit(n_splits=10, gap=21)  
        # gap=21 to avoid leakage (1 month gap)
        
        X = features[self.feature_cols]
        y = features[target_col]
        
        # Remove NaN
        mask = X.notna().all(axis=1) & y.notna()
        X, y = X[mask], y[mask]
        
        scores = []
        models = []
        
        for fold, (train_idx, val_idx) in enumerate(tscv.split(X)):
            X_train, X_val = X.iloc[train_idx], X.iloc[val_idx]
            y_train, y_val = y.iloc[train_idx], y.iloc[val_idx]
            
            model = lgb.LGBMClassifier(
                n_estimators=500,
                max_depth=5,           # Keep shallow - avoid overfit
                learning_rate=0.05,
                num_leaves=31,
                min_child_samples=50,  # Higher = more conservative
                subsample=0.8,
                colsample_bytree=0.8,
                reg_alpha=0.1,         # L1 regularization
                reg_lambda=1.0,        # L2 regularization
                random_state=42,
                n_jobs=-1,
                # Handle class imbalance
                class_weight='balanced',
            )
            
            model.fit(
                X_train, y_train,
                eval_set=[(X_val, y_val)],
                callbacks=[
                    lgb.early_stopping(50),
                    lgb.log_evaluation(100),
                ],
            )
            
            score = model.score(X_val, y_val)
            scores.append(score)
            models.append(model)
            print(f"Fold {fold}: Accuracy = {score:.4f}")
        
        print(f"\nMean CV Score: {np.mean(scores):.4f} "
              f"± {np.std(scores):.4f}")
        
        # Use ensemble of all fold models for prediction
        self.models = models
        return models
    
    def predict_ensemble(self, X_new):
        """Average predictions across all fold models"""
        predictions = np.array([
            m.predict_proba(X_new) for m in self.models
        ])
        return predictions.mean(axis=0)
    
    def feature_importance(self):
        """Understand what drives predictions"""
        importance = pd.DataFrame({
            f'fold_{i}': m.feature_importances_ 
            for i, m in enumerate(self.models)
        }, index=self.feature_cols)
        
        importance['mean'] = importance.mean(axis=1)
        return importance.sort_values('mean', ascending=False)
```

#### D. LSTM / Temporal Models

```python
# ============================================
# LSTM FOR SEQUENCE PREDICTION
# Works for capturing regime changes in CSE
# ============================================

import torch
import torch.nn as nn

class CSEStockLSTM(nn.Module):
    """
    LSTM for CSE stock prediction.
    
    Caution: 
    - Needs more data than you might have per stock
    - Train on ALL stocks together (panel data approach)
    - Don't expect miracles - use as ONE signal among many
    """
    
    def __init__(self, input_size, hidden_size=64, 
                 num_layers=2, dropout=0.3):
        super().__init__()
        
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            dropout=dropout,
            batch_first=True,
        )
        
        self.attention = nn.MultiheadAttention(
            embed_dim=hidden_size,
            num_heads=4,
            dropout=dropout,
        )
        
        self.fc = nn.Sequential(
            nn.Linear(hidden_size, 32),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(32, 3),  # 3 classes: down, neutral, up
        )
    
    def forward(self, x):
        # x shape: (batch, sequence_length, features)
        lstm_out, _ = self.lstm(x)
        
        # Attention over time steps
        attn_out, _ = self.attention(
            lstm_out, lstm_out, lstm_out
        )
        
        # Use last time step
        out = self.fc(attn_out[:, -1, :])
        return out


class LSTMTrainer:
    def __init__(self, lookback=60):
        """Use 60 trading days of history as input"""
        self.lookback = lookback
    
    def create_sequences(self, features_df, target_col):
        """
        Create sliding window sequences.
        Train on ALL stocks together for more data.
        """
        sequences = []
        targets = []
        
        for ticker in features_df['ticker'].unique():
            stock_data = features_df[
                features_df['ticker'] == ticker
            ].sort_values('date')
            
            feature_cols = [c for c in stock_data.columns 
                          if c not in ['ticker', 'date', target_col]]
            
            values = stock_data[feature_cols].values
            target = stock_data[target_col].values
            
            for i in range(self.lookback, len(values)):
                sequences.append(values[i-self.lookback:i])
                targets.append(target[i])
        
        return (
            torch.FloatTensor(np.array(sequences)),
            torch.LongTensor(np.array(targets))
        )
```

### TIER 3: Advanced Models

#### E. Regime Detection (Hidden Markov Models)

```python
# ============================================
# REGIME DETECTION - CRUCIAL FOR CSE
# Sri Lankan market has clear bull/bear/crisis regimes
# ============================================

from hmmlearn import hmm

class MarketRegimeDetector:
    """
    Detect market regimes to adjust strategy:
    - Bull regime: More aggressive, higher equity exposure
    - Normal regime: Standard factor strategy  
    - Bear/Crisis regime: Defensive, cash heavy
    
    CSE has had dramatic regime changes:
    - 2009-2011: Post-war euphoria (massive bull)
    - 2012-2016: Sideways/bear
    - 2017-2018: Mild bull
    - 2019-2020: Crisis (Easter + COVID)
    - 2021: Stimulus-driven rally
    - 2022: Economic collapse
    - 2023-2024: Recovery
    """
    
    def __init__(self, n_regimes=3):
        self.n_regimes = n_regimes
        self.model = None
    
    def fit(self, market_data):
        """
        Fit HMM on ASPI index features.
        Use: returns, volatility, volume as observables.
        """
        features = np.column_stack([
            market_data['returns'].values,
            market_data['volatility_20d'].values,
            market_data['volume_ratio'].values,
        ])
        
        # Remove NaN
        features = features[~np.isnan(features).any(axis=1)]
        
        self.model = hmm.GaussianHMM(
            n_components=self.n_regimes,
            covariance_type='full',
            n_iter=1000,
            random_state=42,
        )
        
        self.model.fit(features)
        
        # Identify which state is which
        regimes = self.model.predict(features)
        
        # State with highest mean return = bull
        # State with lowest mean return = bear
        regime_returns = pd.DataFrame({
            'regime': regimes[1:],
            'return': market_data['returns'].dropna().values[
                :len(regimes)-1
            ]
        }).groupby('regime')['return'].mean()
        
        self.regime_map = {
            regime_returns.idxmax(): 'bull',
            regime_returns.idxmin(): 'bear',
        }
        remaining = set(range(self.n_regimes)) - set(self.regime_map.keys())
        for r in remaining:
            self.regime_map[r] = 'neutral'
        
        return regimes
    
    def current_regime(self, recent_data):
        """Get current market regime"""
        features = np.column_stack([
            recent_data['returns'].values,
            recent_data['volatility_20d'].values,
            recent_data['volume_ratio'].values,
        ])
        features = features[~np.isnan(features).any(axis=1)]
        
        regime = self.model.predict(features)[-1]
        return self.regime_map[regime]
    
    def adjust_allocation(self, base_weights, regime):
        """
        Adjust portfolio based on regime.
        In crisis regime, reduce exposure dramatically.
        """
        regime_multipliers = {
            'bull': 1.0,      # Full exposure
            'neutral': 0.75,  # 75% equity, 25% cash/T-bills
            'bear': 0.30,     # 30% equity, 70% cash/T-bills
        }
        
        multiplier = regime_multipliers[regime]
        adjusted = {k: v * multiplier for k, v in base_weights.items()}
        adjusted['cash'] = 1.0 - sum(adjusted.values())
        
        return adjusted
```

#### F. Pair Trading / Statistical Arbitrage

```python
# ============================================
# PAIRS TRADING FOR CSE
# Good for: Banking sector pairs, conglomerate subsidiaries
# ============================================

from statsmodels.tsa.stattools import coint, adfuller

class PairsTrader:
    """
    Find cointegrated pairs in CSE.
    
    Good candidates in CSE:
    - Commercial Bank vs Hatton National Bank
    - John Keells vs Hemas Holdings
    - Dialog vs SLT (telecom sector)
    - Plantation companies (similar exposure)
    """
    
    def find_cointegrated_pairs(self, price_data, 
                                 significance=0.05):
        """
        Test all pairs for cointegration.
        """
        tickers = price_data.columns.tolist()
        n = len(tickers)
        pairs = []
        
        for i in range(n):
            for j in range(i+1, n):
                p1 = price_data[tickers[i]].dropna()
                p2 = price_data[tickers[j]].dropna()
                
                # Align dates
                common = p1.index.intersection(p2.index)
                if len(common) < 252:  # Need at least 1 year
                    continue
                
                p1, p2 = p1[common], p2[common]
                
                # Engle-Granger cointegration test
                score, pvalue, _ = coint(p1, p2)
                
                if pvalue < significance:
                    # Calculate half-life of mean reversion
                    spread = p1 - p2 * (
                        np.polyfit(p2, p1, 1)[0]
                    )
                    half_life = self._half_life(spread)
                    
                    if 5 < half_life < 60:  # Reasonable mean reversion speed
                        pairs.append({
                            'stock1': tickers[i],
                            'stock2': tickers[j],
                            'pvalue': pvalue,
                            'half_life': half_life,
                        })
        
        return pd.DataFrame(pairs).sort_values('pvalue')
    
    def generate_signals(self, p1, p2, 
                          entry_z=2.0, exit_z=0.5, 
                          stop_z=3.5):
        """Generate trading signals for a pair"""
        # OLS regression
        slope = np.polyfit(p2, p1, 1)[0]
        spread = p1 - slope * p2
        
        # Rolling z-score
        z = (spread - spread.rolling(60).mean()) / spread.rolling(60).std()
        
        signals = pd.Series(0, index=z.index)
        signals[z > entry_z] = -1   # Short spread (short stock1, long stock2)
        signals[z < -entry_z] = 1   # Long spread (long stock1, short stock2)
        signals[abs(z) < exit_z] = 0  # Exit
        signals[abs(z) > stop_z] = 0  # Stop loss
        
        return signals, z
    
    @staticmethod
    def _half_life(spread):
        spread_lag = spread.shift(1).dropna()
        spread_diff = spread.diff().dropna()
        common = spread_lag.index.intersection(spread_diff.index)
        
        beta = np.polyfit(spread_lag[common], spread_diff[common], 1)[0]
        half_life = -np.log(2) / beta if beta < 0 else float('inf')
        return half_life
```

---

## 4. BACKTESTING FRAMEWORK

```python
# ============================================
# PROPER BACKTESTING (AVOID COMMON TRAPS)
# ============================================

class CSEBacktester:
    """
    Backtesting with CSE-specific considerations:
    
    MUST account for:
    1. Transaction costs (brokerage + SEC levy + CSE levy + stamp duty)
    2. Slippage (HUGE for illiquid CSE stocks)
    3. Look-ahead bias (use only data available at decision time)
    4. Survivorship bias (include delisted stocks!)
    5. Market impact (your order moves the price in small stocks)
    """
    
    # CSE transaction costs (approximate)
    BROKERAGE = 0.005      # ~0.5% (negotiable for large accounts)
    SEC_LEVY = 0.00015     # 0.015%
    CSE_LEVY = 0.0004      # 0.04%
    STAMP_DUTY = 0.003     # 0.3% (on seller only, as of recent rules)
    
    # Total round-trip cost estimate
    TOTAL_COST_ROUNDTRIP = 2 * (BROKERAGE + SEC_LEVY + CSE_LEVY) + STAMP_DUTY
    # Approximately 1.4% round trip!
    
    def __init__(self, initial_capital=10_000_000):  # 10M LKR
        self.capital = initial_capital
        self.positions = {}
        self.history = []
    
    def estimate_slippage(self, ticker, order_size_lkr, 
                           avg_daily_turnover):
        """
        Estimate market impact for CSE stocks.
        Rule of thumb: If order > 10% of daily volume,
        expect significant slippage.
        """
        participation_rate = order_size_lkr / avg_daily_turnover
        
        if participation_rate < 0.05:
            slippage = 0.001  # 0.1%
        elif participation_rate < 0.10:
            slippage = 0.003  # 0.3%
        elif participation_rate < 0.25:
            slippage = 0.008  # 0.8%
        else:
            slippage = 0.02   # 2% - too illiquid!
            print(f"WARNING: {ticker} order too large relative to "
                  f"volume. Consider reducing position size.")
        
        return slippage
    
    def run_backtest(self, signals, prices, volumes, 
                      rebalance_freq='monthly'):
        """
        Walk-forward backtest.
        
        signals: dict of date -> {ticker: weight}
        prices: DataFrame of adjusted close prices
        volumes: DataFrame of daily turnover in LKR
        """
        portfolio_value = [self.capital]
        dates = sorted(signals.keys())
        
        for i, date in enumerate(dates):
            target_weights = signals[date]
            current_value = portfolio_value[-1]
            
            # Calculate trades needed
            trades = self._calculate_trades(
                target_weights, current_value, prices.loc[date]
            )
            
            # Apply costs and slippage
            total_cost = 0
            for ticker, trade_value in trades.items():
                if abs(trade_value) > 0:
                    cost = abs(trade_value) * self.TOTAL_COST_ROUNDTRIP
                    slippage_cost = abs(trade_value) * self.estimate_slippage(
                        ticker, abs(trade_value),
                        volumes.loc[date, ticker] if ticker in volumes.columns else 1e6
                    )
                    total_cost += cost + slippage_cost
            
            # Update positions
            self.positions = target_weights
            
            # Calculate return until next rebalance
            if i + 1 < len(dates):
                next_date = dates[i + 1]
                period_prices = prices.loc[date:next_date]
                
                period_return = sum(
                    weight * (
                        period_prices[ticker].iloc[-1] / 
                        period_prices[ticker].iloc[0] - 1
                    )
                    for ticker, weight in target_weights.items()
                    if ticker in period_prices.columns
                )
                
                new_value = current_value * (1 + period_return) - total_cost
                portfolio_value.append(new_value)
        
        return self._compute_metrics(portfolio_value, dates)
    
    def _compute_metrics(self, portfolio_value, dates):
        """Compute comprehensive performance metrics"""
        pv = np.array(portfolio_value)
        returns = np.diff(pv) / pv[:-1]
        
        # Annualized return
        total_return = pv[-1] / pv[0] - 1
        years = len(returns) / 12  # monthly rebalancing
        ann_return = (1 + total_return) ** (1/years) - 1
        
        # Risk metrics
        ann_vol = np.std(returns) * np.sqrt(12)
        sharpe = (ann_return - 0.10) / ann_vol  # 10% risk-free for SL
        
        # Drawdown
        peak = np.maximum.accumulate(pv)
        drawdown = (pv - peak) / peak
        max_drawdown = drawdown.min()
        
        # Calmar ratio
        calmar = ann_return / abs(max_drawdown) if max_drawdown != 0 else 0
        
        # Sortino (downside deviation)
        downside_returns = returns[returns < 0]
        downside_dev = np.std(downside_returns) * np.sqrt(12)
        sortino = (ann_return - 0.10) / downside_dev
        
        metrics = {
            'total_return': f"{total_return:.2%}",
            'annualized_return': f"{ann_return:.2%}",
            'annualized_volatility': f"{ann_vol:.2%}",
            'sharpe_ratio': f"{sharpe:.2f}",
            'sortino_ratio': f"{sortino:.2f}",
            'max_drawdown': f"{max_drawdown:.2%}",
            'calmar_ratio': f"{calmar:.2f}",
            'win_rate': f"{(returns > 0).mean():.2%}",
            
            # CSE-specific benchmarks
            'vs_aspi': "Compare with ASPI total return index",
            'vs_sp20': "Compare with S&P SL 20 index",
            'vs_fixed_deposit': "Compare with bank FD rate (~12%)",
        }
        
        return metrics
```

---

## 5. VOLATILITY MANAGEMENT STRATEGY

```python
# ============================================
# VOLATILITY-BASED POSITION SIZING
# THE SINGLE MOST IMPORTANT THING FOR CSE
# ============================================

class VolatilityManager:
    """
    CSE annual volatility can swing from 10% to 60%+.
    
    Your #1 priority is SURVIVING drawdowns.
    If you lose 50%, you need 100% to get back to even.
    
    Strategy: Target a constant portfolio volatility
    by adjusting equity exposure dynamically.
    """
    
    def __init__(self, target_vol=0.15):  # Target 15% annual vol
        self.target_vol = target_vol
    
    def calculate_exposure(self, current_vol, max_exposure=1.0,
                            min_exposure=0.20):
        """
        Scale equity exposure inversely to current volatility.
        
        If market vol = 15% (target) -> 100% exposure
        If market vol = 30%           -> 50% exposure
        If market vol = 45%           -> 33% exposure
        """
        exposure = self.target_vol / current_vol
        exposure = np.clip(exposure, min_exposure, max_exposure)
        return exposure
    
    def position_size(self, capital, weight, stock_vol,
                       max_loss_per_position=0.02):
        """
        Size each position so max expected loss 
        (2 sigma) doesn't exceed 2% of capital.
        
        This prevents any single stock from blowing up 
        your portfolio.
        """
        # Max position size based on volatility
        # 2-sigma daily move
        daily_2sigma = stock_vol / np.sqrt(252) * 2
        
        # Position size = max_loss / expected_move
        max_position = (capital * max_loss_per_position) / daily_2sigma
        
        # Don't exceed factor model weight
        target_position = capital * weight
        
        return min(max_position, target_position)
    
    def dynamic_stop_loss(self, entry_price, current_vol):
        """
        Volatility-adjusted stop loss.
        Wider stops when vol is high (avoid whipsaws).
        Tighter stops when vol is low.
        """
        atr_multiple = 2.5  # Stop at 2.5x ATR
        daily_vol = current_vol / np.sqrt(252)
        stop_distance = entry_price * daily_vol * atr_multiple * np.sqrt(20)
        # 20-day horizon
        
        stop_price = entry_price - stop_distance
        stop_pct = stop_distance / entry_price
        
        return {
            'stop_price': stop_price,
            'stop_distance_pct': f"{stop_pct:.2%}",
        }
```

---

## 6. THE COMPLETE STRATEGY (Putting It All Together)

```
┌─────────────────────────────────────────────────────────┐
│              CSE QUANT STRATEGY FRAMEWORK                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. UNIVERSE SELECTION                                   │
│     └── Filter to ~50-80 liquid stocks                   │
│         (Min LKR 1M daily turnover)                      │
│                                                          │
│  2. REGIME DETECTION (Monthly)                           │
│     └── HMM on ASPI → Bull / Neutral / Bear             │
│     └── Adjust overall equity exposure                   │
│                                                          │
│  3. STOCK SELECTION (Monthly)                            │
│     ├── Factor Model (60% weight in final signal)        │
│     │   ├── Value (30%)                                  │
│     │   ├── Quality (30%)                                │
│     │   ├── Momentum (20%)                               │
│     │   └── Low Volatility (20%)                         │
│     │                                                    │
│     └── ML Model (40% weight in final signal)            │
│         └── LightGBM with technical + fundamental +      │
│             macro features                               │
│                                                          │
│  4. PORTFOLIO CONSTRUCTION (Monthly)                     │
│     ├── Select top 15-20 stocks                          │
│     ├── Weight by Hierarchical Risk Parity               │
│     ├── Apply sector constraints (max 30% per sector)    │
│     └── Apply single stock cap (max 8%)                  │
│                                                          │
│  5. RISK MANAGEMENT (Daily)                              │
│     ├── Vol-targeting: Scale exposure to 15% target vol  │
│     ├── Position-level vol-adjusted stop losses          │
│     ├── Portfolio drawdown circuit breaker                │
│     │   (If DD > 15%, reduce to 50% exposure)            │
│     │   (If DD > 25%, reduce to 20% exposure)            │
│     └── Cash allocation earns T-bill rate (~10%+)        │
│                                                          │
│  6. EXECUTION                                            │
│     ├── Rebalance monthly (reduce transaction costs)     │
│     ├── Use limit orders (never market orders on CSE!)   │
│     ├── Spread large orders over 2-3 days                │
│     └── Trade in first/last hour (highest liquidity)     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 7. CRITICAL MACRO INDICATORS TO MONITOR

```python
# ============================================
# SRI LANKA SPECIFIC MACRO SIGNALS
# These DOMINATE stock returns in SL
# ============================================

MACRO_DASHBOARD = {
    # PRIMARY SIGNALS
    'USD_LKR': {
        'impact': 'CRITICAL',
        'note': 'Rupee depreciation crushes real returns. '
                'Forex earning companies hedge naturally.',
        'signal': 'If LKR depreciating >5%/month, reduce exposure'
    },
    
    'CBSL_RATES': {
        'impact': 'CRITICAL', 
        'note': 'Policy rate changes directly impact bank stocks '
                '(40%+ of market cap) and all valuations.',
        'signal': 'Rate cuts = bullish. Rate hikes = caution.'
    },
    
    'FOREIGN_FLOWS': {
        'impact': 'HIGH',
        'note': 'Foreign buying/selling drives short-term moves. '
                'Net selling > 5B LKR/month = bearish signal.',
    },
    
    'INFLATION': {
        'impact': 'HIGH',
        'note': 'SL had 70%+ inflation in 2022. '
                'Real returns were deeply negative.',
    },
    
    'FOREX_RESERVES': {
        'impact': 'HIGH',
        'note': 'Reserves < $2B = crisis risk. Monitor monthly.',
    },
    
    'GOVERNMENT_DEBT': {
        'impact': 'MEDIUM-HIGH',
        'note': 'Debt restructuring progress affects '
                'sovereign risk premium and bank NPLs.',
    },
    
    'TOURISM_ARRIVALS': {
        'impact': 'MEDIUM',
        'note': 'Leading indicator for forex inflows '
                'and hotel/leisure sector stocks.',
    },
    
    'TEA_RUBBER_PRICES': {
        'impact': 'SECTOR-SPECIFIC',
        'note': 'Drives plantation company profits.',
    },
}
```

---

## 8. RECOMMENDED IMPLEMENTATION ROADMAP

```
PHASE 1 (Month 1-2): DATA & INFRASTRUCTURE
├── Clean and validate 20 years of price data
├── Build adjusted price series (splits, dividends)
├── Create financial statement database (standardized)
├── Define liquid universe
└── Build backtesting framework with proper costs

PHASE 2 (Month 2-4): FACTOR MODEL (YOUR CORE STRATEGY)
├── Implement multi-factor ranking model
├── Backtest across multiple periods (including 2022 crisis)
├── Test different factor weights
├── Add regime detection overlay
└── Target: Beat ASPI by 5%+ annually after costs

PHASE 3 (Month 4-6): ADD ML LAYER
├── Feature engineering (100+ features)
├── Train LightGBM with proper walk-forward CV
├── Combine ML signal with factor model
├── Analyze feature importance
└── Target: Improve Sharpe by 0.2-0.3 vs pure factor model

PHASE 4 (Month 6-8): RISK MANAGEMENT & REFINEMENT
├── Implement volatility targeting
├── Add drawdown circuit breakers
├── Add macro regime overlay
├── Paper trade for 2-3 months
└── Start with small real capital

PHASE 5 (Month 8+): LIVE TRADING & MONITORING
├── Start with 25% of intended capital
├── Scale up over 6 months if performing
├── Monthly strategy review
├── Quarterly model retraining
└── Annual comprehensive review
```

---

## 9. KEY LIBRARIES

```
pip install pandas numpy scipy statsmodels
pip install scikit-learn lightgbm xgboost
pip install torch  # for LSTM
pip install hmmlearn  # for regime detection
pip install riskfolio-lib  # for portfolio optimization
pip install empyrical  # for performance metrics
pip install matplotlib seaborn plotly  # visualization
```

---

## Final Advice for CSE Specifically

> **1. TRANSACTION COSTS WILL KILL YOU** if you trade frequently. Monthly rebalancing maximum. Quarterly is often better.

> **2. LIQUIDITY IS YOUR BIGGEST CONSTRAINT**, not alpha. A brilliant signal on an illiquid stock is worthless if you can't execute.

> **3. THE MACRO DOMINATES EVERYTHING** in Sri Lanka. Your stock picking can be perfect, but if there's a currency crisis or sovereign default, everything drops together. Regime detection + dynamic allocation is not optional—it's essential.

> **4. T-BILLS ARE YOUR FRIEND.** With rates at 10%+, the opportunity cost of being in cash is LOW. Don't feel pressure to be fully invested. In bear regimes, 70% T-bills + 30% quality stocks can outperform.

> **5. START SIMPLE.** A well-executed factor model with volatility management will likely outperform 90% of CSE participants. Add complexity only if it demonstrably improves risk-adjusted returns.

> **6. SURVIVORSHIP BIAS** - Make sure your 20-year dataset includes companies that were delisted, went bankrupt, or were acquired. Otherwise your backtest will be overly optimistic.

> **7. WATCH FOR DATA SNOOPING.** With 20 years of data, you can find many patterns that don't generalize. Use strict walk-forward validation and out-of-sample testing periods.
