'use strict';

function nz(value, fallback = 0) {
  return Number.isFinite(value) ? value : fallback;
}

function clamp(value, minValue, maxValue) {
  return Math.max(minValue, Math.min(maxValue, value));
}

function rescale(value, oldMin, oldMax, newMin, newMax) {
  if (!Number.isFinite(value) || oldMax === oldMin) return NaN;
  const t = (value - oldMin) / (oldMax - oldMin);
  return newMin + t * (newMax - newMin);
}

function sumFinite(values) {
  let acc = 0;
  for (const v of values) {
    if (Number.isFinite(v)) acc += v;
  }
  return acc;
}

function seriesFromBars(bars, key) {
  return bars.map((bar) => Number(bar[key]));
}

function makeArray(length, fill = NaN) {
  return Array.from({ length }, () => fill);
}

function calcSma(src, length) {
  const out = makeArray(src.length, NaN);
  if (length <= 0) return out;

  let runningSum = 0;

  for (let i = 0; i < src.length; i += 1) {
    const currentValue = src[i];
    if (Number.isFinite(currentValue)) runningSum += currentValue;

    if (i >= length) {
      const removedValue = src[i - length];
      if (Number.isFinite(removedValue)) runningSum -= removedValue;
    }

    if (i >= length - 1) out[i] = runningSum / length;
  }

  return out;
}

function calcEma(src, length) {
  const out = makeArray(src.length, NaN);
  if (length <= 0 || src.length === 0) return out;

  const alpha = 2 / (length + 1);
  let prev = NaN;

  for (let i = 0; i < src.length; i += 1) {
    const currentValue = src[i];
    if (!Number.isFinite(currentValue)) {
      out[i] = prev;
      continue;
    }
    prev = Number.isFinite(prev) ? alpha * currentValue + (1 - alpha) * prev : currentValue;
    out[i] = prev;
  }

  return out;
}

function rma(src, length) {
  const out = makeArray(src.length, NaN);
  if (length <= 0 || src.length === 0) return out;

  let prev = NaN;
  let seedTotal = 0;

  for (let i = 0; i < src.length; i += 1) {
    const currentValue = src[i];
    if (!Number.isFinite(currentValue)) continue;

    if (i < length) {
      seedTotal += currentValue;
      if (i === length - 1) {
        prev = seedTotal / length;
        out[i] = prev;
      }
    } else {
      prev = (prev * (length - 1) + currentValue) / length;
      out[i] = prev;
    }
  }

  return out;
}

function rollingMax(src, length) {
  const out = makeArray(src.length, NaN);
  for (let i = 0; i < src.length; i += 1) {
    if (i < length - 1) continue;
    let highestValue = -Infinity;
    for (let j = i - length + 1; j <= i; j += 1) {
      highestValue = Math.max(highestValue, src[j]);
    }
    out[i] = highestValue;
  }
  return out;
}

function rollingMin(src, length) {
  const out = makeArray(src.length, NaN);
  for (let i = 0; i < src.length; i += 1) {
    if (i < length - 1) continue;
    let lowestValue = Infinity;
    for (let j = i - length + 1; j <= i; j += 1) {
      lowestValue = Math.min(lowestValue, src[j]);
    }
    out[i] = lowestValue;
  }
  return out;
}

function trueRange(high, low, close) {
  const out = makeArray(high.length, NaN);
  for (let i = 0; i < high.length; i += 1) {
    if (i === 0) {
      out[i] = high[i] - low[i];
      continue;
    }
    const hl = high[i] - low[i];
    const hc = Math.abs(high[i] - close[i - 1]);
    const lc = Math.abs(low[i] - close[i - 1]);
    out[i] = Math.max(hl, hc, lc);
  }
  return out;
}

function calcAtr(high, low, close, length) {
  return rma(trueRange(high, low, close), length);
}

function calcRsi(src, length) {
  const gains = makeArray(src.length, NaN);
  const losses = makeArray(src.length, NaN);
  gains[0] = 0;
  losses[0] = 0;

  for (let i = 1; i < src.length; i += 1) {
    const ch = src[i] - src[i - 1];
    gains[i] = Math.max(ch, 0);
    losses[i] = Math.max(-ch, 0);
  }

  const avgGain = rma(gains, length);
  const avgLoss = rma(losses, length);
  const out = makeArray(src.length, NaN);

  for (let i = 0; i < src.length; i += 1) {
    if (!Number.isFinite(avgGain[i]) || !Number.isFinite(avgLoss[i])) continue;
    if (avgLoss[i] === 0) {
      out[i] = 100;
      continue;
    }
    const rs = avgGain[i] / avgLoss[i];
    out[i] = 100 - 100 / (1 + rs);
  }

  return out;
}

function calcCci(src, length) {
  const ma = calcSma(src, length);
  const out = makeArray(src.length, NaN);

  for (let i = 0; i < src.length; i += 1) {
    if (i < length - 1 || !Number.isFinite(ma[i])) continue;

    let meanDev = 0;
    for (let j = i - length + 1; j <= i; j += 1) {
      meanDev += Math.abs(src[j] - ma[i]);
    }
    meanDev /= length;
    out[i] = meanDev === 0 ? 0 : (src[i] - ma[i]) / (0.015 * meanDev);
  }

  return out;
}

function calcAdx(high, low, close, length) {
  const plusDM = makeArray(high.length, 0);
  const minusDM = makeArray(high.length, 0);
  const tr = trueRange(high, low, close);

  for (let i = 1; i < high.length; i += 1) {
    const upMove = high[i] - high[i - 1];
    const downMove = low[i - 1] - low[i];
    plusDM[i] = upMove > downMove && upMove > 0 ? upMove : 0;
    minusDM[i] = downMove > upMove && downMove > 0 ? downMove : 0;
  }

  const plusRma = rma(plusDM, length);
  const minusRma = rma(minusDM, length);
  const trRma = rma(tr, length);
  const dx = makeArray(high.length, NaN);

  for (let i = 0; i < high.length; i += 1) {
    if (
      !Number.isFinite(plusRma[i]) ||
      !Number.isFinite(minusRma[i]) ||
      !Number.isFinite(trRma[i]) ||
      trRma[i] === 0
    ) continue;

    const plusDI = 100 * plusRma[i] / trRma[i];
    const minusDI = 100 * minusRma[i] / trRma[i];
    const denom = plusDI + minusDI;
    dx[i] = denom === 0 ? 0 : 100 * Math.abs(plusDI - minusDI) / denom;
  }

  return rma(dx, length);
}

function waveTrend(hlc3, n1, n2) {
  const esa = calcEma(hlc3, n1);
  const absDev = makeArray(hlc3.length, NaN);

  for (let i = 0; i < hlc3.length; i += 1) {
    absDev[i] = Number.isFinite(esa[i]) ? Math.abs(hlc3[i] - esa[i]) : NaN;
  }

  const d = calcEma(absDev, n1);
  const ci = makeArray(hlc3.length, NaN);

  for (let i = 0; i < hlc3.length; i += 1) {
    ci[i] = Number.isFinite(d[i]) && d[i] !== 0 ? (hlc3[i] - esa[i]) / (0.015 * d[i]) : NaN;
  }

  const wt1 = calcEma(ci, n2);
  const wt2 = calcSma(wt1, 4);
  return { wt1, wt2 };
}

function normalizeUnbounded(src, lookback = 200, newMin = -1, newMax = 1) {
  const out = makeArray(src.length, NaN);

  for (let i = 0; i < src.length; i += 1) {
    const start = Math.max(0, i - lookback + 1);
    let lo = Infinity;
    let hi = -Infinity;

    for (let j = start; j <= i; j += 1) {
      const currentValue = src[j];
      if (!Number.isFinite(currentValue)) continue;
      lo = Math.min(lo, currentValue);
      hi = Math.max(hi, currentValue);
    }

    if (!Number.isFinite(src[i])) continue;
    out[i] = hi === lo ? 0 : rescale(src[i], lo, hi, newMin, newMax);
  }

  return out;
}

function crossover(a, b) {
  const out = makeArray(a.length, false);

  for (let i = 1; i < a.length; i += 1) {
    out[i] =
      Number.isFinite(a[i - 1]) &&
      Number.isFinite(b[i - 1]) &&
      Number.isFinite(a[i]) &&
      Number.isFinite(b[i])
        ? a[i - 1] < b[i - 1] && a[i] >= b[i]
        : false;
  }

  return out;
}

function crossunder(a, b) {
  const out = makeArray(a.length, false);

  for (let i = 1; i < a.length; i += 1) {
    out[i] =
      Number.isFinite(a[i - 1]) &&
      Number.isFinite(b[i - 1]) &&
      Number.isFinite(a[i]) &&
      Number.isFinite(b[i])
        ? a[i - 1] > b[i - 1] && a[i] <= b[i]
        : false;
  }

  return out;
}

function barsSince(condition) {
  const out = makeArray(condition.length, Infinity);
  let lastTrue = -Infinity;

  for (let i = 0; i < condition.length; i += 1) {
    if (condition[i]) lastTrue = i;
    out[i] = Number.isFinite(lastTrue) ? i - lastTrue : Infinity;
  }

  return out;
}

function rationalQuadraticKernel(src, lookback, relativeWeight, startAtBar) {
  const out = makeArray(src.length, NaN);
  const denomBase = 2 * relativeWeight * lookback * lookback;

  for (let i = 0; i < src.length; i += 1) {
    if (i < startAtBar) continue;

    let numerator = 0;
    let denominator = 0;
    const last = Math.min(i, lookback - 1);

    for (let j = 0; j <= last; j += 1) {
      const currentValue = src[i - j];
      if (!Number.isFinite(currentValue)) continue;
      const weight = Math.pow(1 + (j * j) / Math.max(1e-12, denomBase), -relativeWeight);
      numerator += currentValue * weight;
      denominator += weight;
    }

    out[i] = denominator === 0 ? NaN : numerator / denominator;
  }

  return out;
}

function gaussianKernel(src, lookback, startAtBar) {
  const out = makeArray(src.length, NaN);
  const sigma2 = lookback * lookback;

  for (let i = 0; i < src.length; i += 1) {
    if (i < startAtBar) continue;

    let numerator = 0;
    let denominator = 0;
    const last = Math.min(i, lookback - 1);

    for (let j = 0; j <= last; j += 1) {
      const currentValue = src[i - j];
      if (!Number.isFinite(currentValue)) continue;
      const weight = Math.exp(-(j * j) / Math.max(1e-12, 2 * sigma2));
      numerator += currentValue * weight;
      denominator += weight;
    }

    out[i] = denominator === 0 ? NaN : numerator / denominator;
  }

  return out;
}

const ml = {
  nRsi(src, n1, n2) {
    const base = calcRsi(src, n1);
    const smooth = n2 > 1 ? calcEma(base, n2) : base;
    return smooth.map((v) => (Number.isFinite(v) ? rescale(v, 0, 100, -1, 1) : NaN));
  },

  nWt(src, n1, n2) {
    const wt = waveTrend(src, n1, n2);
    return wt.wt1.map((v) => (Number.isFinite(v) ? Math.tanh(v / 100) : NaN));
  },

  nCci(src, n1, n2) {
    const base = calcCci(src, n1);
    const smooth = n2 > 1 ? calcEma(base, n2) : base;
    return smooth.map((v) => (Number.isFinite(v) ? Math.tanh(v / 100) : NaN));
  },

  nAdx(high, low, close, n1) {
    const base = calcAdx(high, low, close, n1);
    return base.map((v) => (Number.isFinite(v) ? rescale(v, 0, 100, -1, 1) : NaN));
  },

  filterVolatility(high, low, close, minLength, maxLength, useVolatilityFilter) {
    if (!useVolatilityFilter) return Array.from({ length: close.length }, () => true);
    const shortAtr = calcAtr(high, low, close, minLength);
    const longAtr = calcAtr(high, low, close, maxLength);
    return shortAtr.map((v, i) =>
      Number.isFinite(v) && Number.isFinite(longAtr[i]) ? v > longAtr[i] : false
    );
  },

  regimeFilter(src, threshold, useRegimeFilter, lookback = 20) {
    if (!useRegimeFilter) return Array.from({ length: src.length }, () => true);

    const out = makeArray(src.length, false);
    for (let i = 0; i < src.length; i += 1) {
      if (i < lookback || !Number.isFinite(src[i - lookback]) || src[i - lookback] === 0) continue;
      const regimeValue = (src[i] - src[i - lookback]) / Math.abs(src[i - lookback]);
      out[i] = regimeValue > threshold;
    }
    return out;
  },

  filterAdx(high, low, close, length, adxThreshold, useAdxFilter) {
    if (!useAdxFilter) return Array.from({ length: close.length }, () => true);
    const adxSeries = calcAdx(high, low, close, length);
    return adxSeries.map((v) => (Number.isFinite(v) ? v > adxThreshold : false));
  },

  backtest(payload) {
    const {
      open,
      startLongTrade,
      endLongTrade,
      startShortTrade,
      endShortTrade,
      isEarlySignalFlip,
      useWorstCase,
    } = payload;

    let inPosition = 0;
    let entryPrice = NaN;
    let totalWins = 0;
    let totalLosses = 0;
    let totalTrades = 0;
    let totalEarlySignalFlips = 0;

    for (let i = 0; i < open.length; i += 1) {
      if (isEarlySignalFlip[i]) totalEarlySignalFlips += 1;

      if (inPosition === 0) {
        if (startLongTrade[i]) {
          inPosition = 1;
          entryPrice = useWorstCase ? closeOr(open, i) : open[i];
          totalTrades += 1;
        } else if (startShortTrade[i]) {
          inPosition = -1;
          entryPrice = useWorstCase ? closeOr(open, i) : open[i];
          totalTrades += 1;
        }
        continue;
      }

      if (inPosition === 1 && endLongTrade[i]) {
        const exitPrice = useWorstCase ? closeOr(open, i) : open[i];
        if (exitPrice >= entryPrice) totalWins += 1;
        else totalLosses += 1;
        inPosition = 0;
        entryPrice = NaN;
      }

      if (inPosition === -1 && endShortTrade[i]) {
        const exitPrice = useWorstCase ? closeOr(open, i) : open[i];
        if (exitPrice <= entryPrice) totalWins += 1;
        else totalLosses += 1;
        inPosition = 0;
        entryPrice = NaN;
      }
    }

    const winRate = totalTrades === 0 ? 0 : totalWins / totalTrades;
    const winLossRatio =
      totalLosses === 0 ? (totalWins > 0 ? Infinity : 0) : totalWins / totalLosses;

    return {
      totalWins,
      totalLosses,
      totalEarlySignalFlips,
      totalTrades,
      tradeStatsHeader: 'Trade Stats',
      winLossRatio,
      winRate,
    };
  },
};

function closeOr(openSeries, index) {
  return openSeries[index];
}

const kernels = {
  rationalQuadratic: rationalQuadraticKernel,
  gaussian: gaussianKernel,
};

function seriesShift(arr, index, offset, fallback = NaN) {
  const k = index - offset;
  return k >= 0 && k < arr.length ? arr[k] : fallback;
}

function change(arr, index) {
  if (index <= 0) return 0;
  const a = arr[index];
  const b = arr[index - 1];
  if (!Number.isFinite(a) || !Number.isFinite(b)) return 0;
  return a - b;
}

function boolAt(arr, idx) {
  return idx >= 0 && idx < arr.length ? Boolean(arr[idx]) : false;
}

const DEFAULT_SETTINGS = {
  sourceKey: 'close',
  neighborsCount: 8,
  maxBarsBack: 2000,
  featureCount: 5,
  colorCompression: 1,
  showExits: false,
  useDynamicExits: false,
  showTradeStats: true,
  useWorstCase: false,
  filterSettings: {
    useVolatilityFilter: true,
    useRegimeFilter: true,
    useAdxFilter: false,
    regimeThreshold: -0.1,
    adxThreshold: 20,
  },
  features: [
    { type: 'RSI', paramA: 14, paramB: 1 },
    { type: 'WT', paramA: 10, paramB: 11 },
    { type: 'CCI', paramA: 20, paramB: 1 },
    { type: 'ADX', paramA: 20, paramB: 2 },
    { type: 'RSI', paramA: 9, paramB: 1 },
  ],
  useEmaFilter: false,
  emaPeriod: 200,
  useSmaFilter: false,
  smaPeriod: 200,
  useKernelFilter: true,
  showKernelEstimate: true,
  useKernelSmoothing: false,
  h: 8,
  r: 8.0,
  x: 25,
  lag: 2,
};

function buildSettings(options) {
  const inputOptions = options || {};
  const inputFilterSettings = inputOptions.filterSettings || {};

  return {
    sourceKey:
      inputOptions.sourceKey !== undefined ? inputOptions.sourceKey : DEFAULT_SETTINGS.sourceKey,
    neighborsCount:
      inputOptions.neighborsCount !== undefined
        ? inputOptions.neighborsCount
        : DEFAULT_SETTINGS.neighborsCount,
    maxBarsBack:
      inputOptions.maxBarsBack !== undefined
        ? inputOptions.maxBarsBack
        : DEFAULT_SETTINGS.maxBarsBack,
    featureCount:
      inputOptions.featureCount !== undefined
        ? inputOptions.featureCount
        : DEFAULT_SETTINGS.featureCount,
    colorCompression:
      inputOptions.colorCompression !== undefined
        ? inputOptions.colorCompression
        : DEFAULT_SETTINGS.colorCompression,
    showExits:
      inputOptions.showExits !== undefined ? inputOptions.showExits : DEFAULT_SETTINGS.showExits,
    useDynamicExits:
      inputOptions.useDynamicExits !== undefined
        ? inputOptions.useDynamicExits
        : DEFAULT_SETTINGS.useDynamicExits,
    showTradeStats:
      inputOptions.showTradeStats !== undefined
        ? inputOptions.showTradeStats
        : DEFAULT_SETTINGS.showTradeStats,
    useWorstCase:
      inputOptions.useWorstCase !== undefined
        ? inputOptions.useWorstCase
        : DEFAULT_SETTINGS.useWorstCase,
    filterSettings: {
      useVolatilityFilter:
        inputFilterSettings.useVolatilityFilter !== undefined
          ? inputFilterSettings.useVolatilityFilter
          : DEFAULT_SETTINGS.filterSettings.useVolatilityFilter,
      useRegimeFilter:
        inputFilterSettings.useRegimeFilter !== undefined
          ? inputFilterSettings.useRegimeFilter
          : DEFAULT_SETTINGS.filterSettings.useRegimeFilter,
      useAdxFilter:
        inputFilterSettings.useAdxFilter !== undefined
          ? inputFilterSettings.useAdxFilter
          : DEFAULT_SETTINGS.filterSettings.useAdxFilter,
      regimeThreshold:
        inputFilterSettings.regimeThreshold !== undefined
          ? inputFilterSettings.regimeThreshold
          : DEFAULT_SETTINGS.filterSettings.regimeThreshold,
      adxThreshold:
        inputFilterSettings.adxThreshold !== undefined
          ? inputFilterSettings.adxThreshold
          : DEFAULT_SETTINGS.filterSettings.adxThreshold,
    },
    features: inputOptions.features !== undefined ? inputOptions.features : DEFAULT_SETTINGS.features,
    useEmaFilter:
      inputOptions.useEmaFilter !== undefined
        ? inputOptions.useEmaFilter
        : DEFAULT_SETTINGS.useEmaFilter,
    emaPeriod:
      inputOptions.emaPeriod !== undefined ? inputOptions.emaPeriod : DEFAULT_SETTINGS.emaPeriod,
    useSmaFilter:
      inputOptions.useSmaFilter !== undefined
        ? inputOptions.useSmaFilter
        : DEFAULT_SETTINGS.useSmaFilter,
    smaPeriod:
      inputOptions.smaPeriod !== undefined ? inputOptions.smaPeriod : DEFAULT_SETTINGS.smaPeriod,
    useKernelFilter:
      inputOptions.useKernelFilter !== undefined
        ? inputOptions.useKernelFilter
        : DEFAULT_SETTINGS.useKernelFilter,
    showKernelEstimate:
      inputOptions.showKernelEstimate !== undefined
        ? inputOptions.showKernelEstimate
        : DEFAULT_SETTINGS.showKernelEstimate,
    useKernelSmoothing:
      inputOptions.useKernelSmoothing !== undefined
        ? inputOptions.useKernelSmoothing
        : DEFAULT_SETTINGS.useKernelSmoothing,
    h: inputOptions.h !== undefined ? inputOptions.h : DEFAULT_SETTINGS.h,
    r: inputOptions.r !== undefined ? inputOptions.r : DEFAULT_SETTINGS.r,
    x: inputOptions.x !== undefined ? inputOptions.x : DEFAULT_SETTINGS.x,
    lag: inputOptions.lag !== undefined ? inputOptions.lag : DEFAULT_SETTINGS.lag,
  };
}

function seriesFromFeature(featureType, close, high, low, hlc3, paramA, paramB) {
  switch (featureType) {
    case 'RSI':
      return ml.nRsi(close, paramA, paramB);
    case 'WT':
      return ml.nWt(hlc3, paramA, paramB);
    case 'CCI':
      return ml.nCci(close, paramA, paramB);
    case 'ADX':
      return ml.nAdx(high, low, close, paramA);
    default:
      return ml.nRsi(close, paramA, paramB);
  }
}

function getLorentzianDistance(i, featureSeries, settings) {
  let d = 0;
  const featureCount = settings.featureCount;

  for (let k = 0; k < featureCount; k += 1) {
    const currentFeature = featureSeries[k].current;
    const historicalFeature = featureSeries[k].history[i];
    d += Math.log(1 + Math.abs(currentFeature - historicalFeature));
  }

  return d;
}

function runLorentzianClassification(bars, options) {
  const settings = buildSettings(options);
  const n = bars.length;
  if (n === 0) return null;

  const openSeries = seriesFromBars(bars, 'open');
  const highSeries = seriesFromBars(bars, 'high');
  const lowSeries = seriesFromBars(bars, 'low');
  const closeSeries = seriesFromBars(bars, 'close');
  const volumeSeries = seriesFromBars(bars, 'volume');
  const hlc3Series = bars.map((b) => (b.high + b.low + b.close) / 3);
  const ohlc4Series = bars.map((b) => (b.open + b.high + b.low + b.close) / 4);

  const sourceSeries = closeSeries;
  const featureArrays = settings.features.map((f) =>
    seriesFromFeature(
      f.type,
      closeSeries,
      highSeries,
      lowSeries,
      hlc3Series,
      f.paramA,
      f.paramB
    )
  );

  const emaSeries = calcEma(closeSeries, settings.emaPeriod);
  const smaSeries = calcSma(closeSeries, settings.smaPeriod);

  const isEmaUptrend = closeSeries.map((c, i) =>
    settings.useEmaFilter ? c > emaSeries[i] : true
  );
  const isEmaDowntrend = closeSeries.map((c, i) =>
    settings.useEmaFilter ? c < emaSeries[i] : true
  );
  const isSmaUptrend = closeSeries.map((c, i) =>
    settings.useSmaFilter ? c > smaSeries[i] : true
  );
  const isSmaDowntrend = closeSeries.map((c, i) =>
    settings.useSmaFilter ? c < smaSeries[i] : true
  );

  const volatilityFilter = ml.filterVolatility(
    highSeries,
    lowSeries,
    closeSeries,
    1,
    10,
    settings.filterSettings.useVolatilityFilter
  );
  const regimeFilter = ml.regimeFilter(
    ohlc4Series,
    settings.filterSettings.regimeThreshold,
    settings.filterSettings.useRegimeFilter
  );
  const adxFilter = ml.filterAdx(
    highSeries,
    lowSeries,
    closeSeries,
    14,
    settings.filterSettings.adxThreshold,
    settings.filterSettings.useAdxFilter
  );

  const yhat1 = kernels.rationalQuadratic(
    sourceSeries,
    settings.h,
    settings.r,
    settings.x
  );
  const yhat2 = kernels.gaussian(
    sourceSeries,
    settings.h - settings.lag,
    settings.x
  );
  const bullishCross = crossover(yhat2, yhat1);
  const bearishCross = crossunder(yhat2, yhat1);

  const yTrainArray = [];
  const persistentPredictions = [];
  const persistentDistances = [];

  const prediction = makeArray(n, 0);
  const signal = makeArray(n, 0);
  const barsHeld = makeArray(n, 0);
  const isHeldFourBars = makeArray(n, false);
  const isHeldLessThanFourBars = makeArray(n, false);
  const isDifferentSignalType = makeArray(n, false);
  const isEarlySignalFlip = makeArray(n, false);
  const isBuySignal = makeArray(n, false);
  const isSellSignal = makeArray(n, false);
  const isNewBuySignal = makeArray(n, false);
  const isNewSellSignal = makeArray(n, false);
  const wasBearishRate = makeArray(n, false);
  const wasBullishRate = makeArray(n, false);
  const isBearishRate = makeArray(n, false);
  const isBullishRate = makeArray(n, false);
  const isBearishChange = makeArray(n, false);
  const isBullishChange = makeArray(n, false);
  const alertBullish = makeArray(n, false);
  const alertBearish = makeArray(n, false);
  const isBullish = makeArray(n, true);
  const isBearish = makeArray(n, true);
  const startLongTrade = makeArray(n, false);
  const startShortTrade = makeArray(n, false);
  const endLongTradeDynamic = makeArray(n, false);
  const endShortTradeDynamic = makeArray(n, false);
  const endLongTradeStrict = makeArray(n, false);
  const endShortTradeStrict = makeArray(n, false);
  const endLongTrade = makeArray(n, false);
  const endShortTrade = makeArray(n, false);
  const backTestStream = makeArray(n, 0);

  const maxBarsBackIndex = Math.max(
    0,
    n - 1 >= settings.maxBarsBack ? n - 1 - settings.maxBarsBack : 0
  );

  for (let t = 0; t < n; t += 1) {
    const yTrain =
      seriesShift(sourceSeries, t, 4) < sourceSeries[t]
        ? -1
        : seriesShift(sourceSeries, t, 4) > sourceSeries[t]
        ? 1
        : 0;

    yTrainArray.push(yTrain);

    let lastDistance = -1.0;
    const size = Math.min(settings.maxBarsBack - 1, yTrainArray.length - 1);
    const sizeLoop = Math.min(settings.maxBarsBack - 1, size);

    if (t >= maxBarsBackIndex) {
      const featureSeries = featureArrays
        .slice(0, settings.featureCount)
        .map((history) => ({ current: history[t], history }));

      for (let i = 0; i <= sizeLoop; i += 1) {
        const d = getLorentzianDistance(i, featureSeries, settings);
        if (d >= lastDistance && i % 4) {
          lastDistance = d;
          persistentDistances.push(d);
          persistentPredictions.push(Math.round(yTrainArray[i]));

          if (persistentPredictions.length > settings.neighborsCount) {
            const idx = Math.round((settings.neighborsCount * 3) / 4);
            lastDistance =
              persistentDistances[Math.min(idx, persistentDistances.length - 1)];
            persistentDistances.shift();
            persistentPredictions.shift();
          }
        }
      }

      prediction[t] = sumFinite(persistentPredictions);
    }

    const filterAll = volatilityFilter[t] && regimeFilter[t] && adxFilter[t];

    signal[t] =
      prediction[t] > 0 && filterAll
        ? 1
        : prediction[t] < 0 && filterAll
        ? -1
        : t > 0
        ? signal[t - 1]
        : 0;

    isDifferentSignalType[t] = change(signal, t) !== 0;
    barsHeld[t] = isDifferentSignalType[t] ? 0 : t > 0 ? barsHeld[t - 1] + 1 : 0;
    isHeldFourBars[t] = barsHeld[t] === 4;
    isHeldLessThanFourBars[t] = barsHeld[t] > 0 && barsHeld[t] < 4;

    isEarlySignalFlip[t] =
      isDifferentSignalType[t] &&
      (change(signal, t - 1) !== 0 ||
        change(signal, t - 2) !== 0 ||
        change(signal, t - 3) !== 0);

    wasBearishRate[t] = seriesShift(yhat1, t, 2) > seriesShift(yhat1, t, 1);
    wasBullishRate[t] = seriesShift(yhat1, t, 2) < seriesShift(yhat1, t, 1);
    isBearishRate[t] = seriesShift(yhat1, t, 1) > yhat1[t];
    isBullishRate[t] = seriesShift(yhat1, t, 1) < yhat1[t];
    isBearishChange[t] = isBearishRate[t] && wasBullishRate[t];
    isBullishChange[t] = isBullishRate[t] && wasBearishRate[t];

    alertBullish[t] = settings.useKernelSmoothing ? bullishCross[t] : isBullishChange[t];
    alertBearish[t] = settings.useKernelSmoothing ? bearishCross[t] : isBearishChange[t];

    isBullish[t] = settings.useKernelFilter
      ? settings.useKernelSmoothing
        ? yhat2[t] >= yhat1[t]
        : isBullishRate[t]
      : true;

    isBearish[t] = settings.useKernelFilter
      ? settings.useKernelSmoothing
        ? yhat2[t] <= yhat1[t]
        : isBearishRate[t]
      : true;

    isBuySignal[t] = signal[t] === 1 && isEmaUptrend[t] && isSmaUptrend[t];
    isSellSignal[t] = signal[t] === -1 && isEmaDowntrend[t] && isSmaDowntrend[t];
    isNewBuySignal[t] = isBuySignal[t] && isDifferentSignalType[t];
    isNewSellSignal[t] = isSellSignal[t] && isDifferentSignalType[t];

    startLongTrade[t] =
      isNewBuySignal[t] && isBullish[t] && isEmaUptrend[t] && isSmaUptrend[t];
    startShortTrade[t] =
      isNewSellSignal[t] && isBearish[t] && isEmaDowntrend[t] && isSmaDowntrend[t];
  }

  const barsSinceShortEntry = barsSince(startShortTrade);
  const barsSinceBullishAlert = barsSince(alertBullish);
  const barsSinceLongEntry = barsSince(startLongTrade);
  const barsSinceBearishAlert = barsSince(alertBearish);

  const isDynamicExitValid =
    !settings.useEmaFilter &&
    !settings.useSmaFilter &&
    !settings.useKernelSmoothing;

  for (let t = 0; t < n; t += 1) {
    const isLastSignalBuy =
      seriesShift(signal, t, 4, 0) === 1 &&
      boolAt(isEmaUptrend, t - 4) &&
      boolAt(isSmaUptrend, t - 4);

    const isLastSignalSell =
      seriesShift(signal, t, 4, 0) === -1 &&
      boolAt(isEmaDowntrend, t - 4) &&
      boolAt(isSmaDowntrend, t - 4);

    const prevIsValidLongExit =
      t > 0 ? barsSinceBearishAlert[t - 1] > barsSinceLongEntry[t - 1] : false;

    const prevIsValidShortExit =
      t > 0 ? barsSinceBullishAlert[t - 1] > barsSinceShortEntry[t - 1] : false;

    endLongTradeDynamic[t] = isBearishChange[t] && prevIsValidLongExit;
    endShortTradeDynamic[t] = isBullishChange[t] && prevIsValidShortExit;

    endLongTradeStrict[t] =
      ((isHeldFourBars[t] && isLastSignalBuy) ||
        (isHeldLessThanFourBars[t] && isNewSellSignal[t] && isLastSignalBuy)) &&
      boolAt(startLongTrade, t - 4);

    endShortTradeStrict[t] =
      ((isHeldFourBars[t] && isLastSignalSell) ||
        (isHeldLessThanFourBars[t] && isNewBuySignal[t] && isLastSignalSell)) &&
      boolAt(startShortTrade, t - 4);

    endLongTrade[t] =
      settings.useDynamicExits && isDynamicExitValid
        ? endLongTradeDynamic[t]
        : endLongTradeStrict[t];

    endShortTrade[t] =
      settings.useDynamicExits && isDynamicExitValid
        ? endShortTradeDynamic[t]
        : endShortTradeStrict[t];

    backTestStream[t] = startLongTrade[t]
      ? 1
      : endLongTrade[t]
      ? 2
      : startShortTrade[t]
      ? -1
      : endShortTrade[t]
      ? -2
      : 0;
  }

  const stats = settings.showTradeStats
    ? ml.backtest({
        open: openSeries,
        high: highSeries,
        low: lowSeries,
        startLongTrade,
        endLongTrade,
        startShortTrade,
        endShortTrade,
        isEarlySignalFlip,
        useWorstCase: settings.useWorstCase,
      })
    : null;

  return {
    open: openSeries,
    high: highSeries,
    low: lowSeries,
    close: closeSeries,
    volume: volumeSeries,
    hlc3: hlc3Series,
    ohlc4: ohlc4Series,
    features: featureArrays,
    prediction,
    signal,
    kernelEstimate: yhat1,
    kernelAux: yhat2,
    alertBullish,
    alertBearish,
    startLongTrade,
    startShortTrade,
    endLongTrade,
    endShortTrade,
    backTestStream,
    stats,
  };
}
