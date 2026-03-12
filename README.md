describe_indicator('Machine Learning: Lorentzian Classification TS Port', 'price', {
    shortName: 'Lorentzian TS',
    decimals: 'by_symbol'
});

const sourceName = input.select('Source', 'close', constants.price_source_options);
const sourceSeries = market[sourceName];
const neighborsCount = input.number('Neighbors Count', 8, { min: 1, max: 100 });
const maxBarsBack = input.number('Max Bars Back', 2000, { min: 100, max: 10000 });
const featureCount = input.number('Feature Count', 5, { min: 2, max: 5 });
const colorCompression = input.number('Color Compression', 1, { min: 1, max: 10 });
const showExits = input.boolean('Show Default Exits', false);
const useDynamicExits = input.boolean('Use Dynamic Exits', false);
const showTradeStats = input.boolean('Show Trade Stats', true);
const useWorstCase = input.boolean('Use Worst Case Estimates', false);

const useVolatilityFilter = input.boolean('Use Volatility Filter', true);
const useRegimeFilter = input.boolean('Use Regime Filter', true);
const useAdxFilter = input.boolean('Use ADX Filter', false);
const regimeThreshold = input.number('Regime Threshold', -0.1, { min: -10, max: 10 });
const adxThreshold = input.number('ADX Threshold', 20, { min: 0, max: 100 });

const feature1Type = input.select('Feature 1 Type', 'RSI', ['RSI', 'WT', 'CCI', 'ADX']);
const feature1ParamA = input.number('Feature 1 Param A', 14, { min: 1, max: 200 });
const feature1ParamB = input.number('Feature 1 Param B', 1, { min: 1, max: 200 });
const feature2Type = input.select('Feature 2 Type', 'WT', ['RSI', 'WT', 'CCI', 'ADX']);
const feature2ParamA = input.number('Feature 2 Param A', 10, { min: 1, max: 200 });
const feature2ParamB = input.number('Feature 2 Param B', 11, { min: 1, max: 200 });
const feature3Type = input.select('Feature 3 Type', 'CCI', ['RSI', 'WT', 'CCI', 'ADX']);
const feature3ParamA = input.number('Feature 3 Param A', 20, { min: 1, max: 200 });
const feature3ParamB = input.number('Feature 3 Param B', 1, { min: 1, max: 200 });
const feature4Type = input.select('Feature 4 Type', 'ADX', ['RSI', 'WT', 'CCI', 'ADX']);
const feature4ParamA = input.number('Feature 4 Param A', 20, { min: 1, max: 200 });
const feature4ParamB = input.number('Feature 4 Param B', 2, { min: 1, max: 200 });
const feature5Type = input.select('Feature 5 Type', 'RSI', ['RSI', 'WT', 'CCI', 'ADX']);
const feature5ParamA = input.number('Feature 5 Param A', 9, { min: 1, max: 200 });
const feature5ParamB = input.number('Feature 5 Param B', 1, { min: 1, max: 200 });

const useEmaFilter = input.boolean('Use EMA Filter', false);
const emaPeriod = input.number('EMA Filter Period', 200, { min: 1, max: 2000 });
const useSmaFilter = input.boolean('Use SMA Filter', false);
const smaPeriod = input.number('SMA Filter Period', 200, { min: 1, max: 2000 });

const useKernelFilter = input.boolean('Trade with Kernel', true);
const showKernelEstimate = input.boolean('Show Kernel Estimate', true);
const useKernelSmoothing = input.boolean('Enhance Kernel Smoothing', false);
const kernelLookback = input.number('Kernel Lookback Window', 8, { min: 3, max: 100 });
const kernelRelativeWeight = input.number('Kernel Relative Weighting', 8, { min: 0.25, max: 100 });
const kernelStartAtBar = input.number('Kernel Regression Level', 25, { min: 1, max: 200 });
const kernelLag = input.number('Kernel Lag', 2, { min: 1, max: 10 });

const showBarColors = input.boolean('Show Bar Colors', true);
const showBarPredictions = input.boolean('Show Bar Prediction Values', true);
const useAtrOffset = input.boolean('Use ATR Offset', false);
const barPredictionsOffset = input.number('Bar Prediction Offset', 0, { min: 0, max: 25, hide_in_legend: true });

const candleCount = close.length;
const maxBarsBackIndex = candleCount - 1 >= maxBarsBack ? candleCount - 1 - maxBarsBack : 0;
const transparentColor = 'rgba(0, 0, 0, 0)';
const bullishKernelColor = 'rgba(0, 153, 136, 0.80)';
const bearishKernelColor = 'rgba(204, 51, 17, 0.80)';
const bullishMarkerColor = 'rgba(0, 153, 136, 0.90)';
const bearishMarkerColor = 'rgba(204, 51, 17, 0.90)';
const exitLongColor = 'rgba(58, 255, 23, 0.90)';
const exitShortColor = 'rgba(253, 23, 7, 0.90)';
const neutralColor = '#787b86';
const bearishGradientBase = '#CC3311';
const bullishGradientBase = '#009988';

function newSeries(defaultValue) {
    const out = new Array(candleCount);
    for (let index = 0; index < candleCount; index++) {
        out[index] = defaultValue;
    }
    return out;
}

function isValidNumber(value) {
    return value !== null && value !== undefined && Number.isFinite(value);
}

function clamp(value, minValue, maxValue) {
    return Math.min(Math.max(value, minValue), maxValue);
}

function sumNumbers(values) {
    let total = 0;
    for (let index = 0; index < values.length; index++) {
        total += values[index];
    }
    return total;
}

function hexToRgb(hexColor) {
    const normalized = hexColor.replace('#', '');
    const red = parseInt(normalized.slice(0, 2), 16);
    const green = parseInt(normalized.slice(2, 4), 16);
    const blue = parseInt(normalized.slice(4, 6), 16);
    return { red, green, blue };
}

function mixColors(fromHex, toHex, ratio) {
    const boundedRatio = clamp(ratio, 0, 1);
    const from = hexToRgb(fromHex);
    const to = hexToRgb(toHex);
    const red = Math.round(from.red + (to.red - from.red) * boundedRatio);
    const green = Math.round(from.green + (to.green - from.green) * boundedRatio);
    const blue = Math.round(from.blue + (to.blue - from.blue) * boundedRatio);
    return `rgb(${red}, ${green}, ${blue})`;
}

function withAlpha(colorString, alpha) {
    const boundedAlpha = clamp(alpha, 0, 1);
    if (colorString.indexOf('rgb(') === 0) {
        const numbers = colorString.replace('rgb(', '').replace(')', '').split(',').map(item => parseInt(item.trim(), 10));
        return `rgba(${numbers[0]}, ${numbers[1]}, ${numbers[2]}, ${boundedAlpha})`;
    }
    if (colorString.indexOf('#') === 0) {
        const rgb = hexToRgb(colorString);
        return `rgba(${rgb.red}, ${rgb.green}, ${rgb.blue}, ${boundedAlpha})`;
    }
    return colorString;
}

function normalizeBounded(series, oldMin, oldMax) {
    const out = newSeries(null);
    const denominator = oldMax - oldMin;
    for (let index = 0; index < candleCount; index++) {
        const value = series[index];
        out[index] = isValidNumber(value) ? (((value - oldMin) / denominator) * 2) - 1 : null;
    }
    return out;
}

function normalizeUnbounded(series, scale) {
    const out = newSeries(null);
    for (let index = 0; index < candleCount; index++) {
        const value = series[index];
        out[index] = isValidNumber(value) ? Math.tanh(value / scale) : null;
    }
    return out;
}

function waveTrendClassic(priceSeries, channelLength, averageLength) {
    const esa = ema(priceSeries, channelLength);
    const absDeviation = for_every(priceSeries, esa, function(priceValue, esaValue) {
        if (!isValidNumber(priceValue) || !isValidNumber(esaValue)) {
            return null;
        }
        return Math.abs(priceValue - esaValue);
    });
    const deviationAverage = ema(absDeviation, channelLength);
    const ci = for_every(priceSeries, esa, deviationAverage, function(priceValue, esaValue, deviationValue) {
        if (!isValidNumber(priceValue) || !isValidNumber(esaValue) || !isValidNumber(deviationValue) || deviationValue === 0) {
            return null;
        }
        return (priceValue - esaValue) / (0.015 * deviationValue);
    });
    const wt1 = ema(ci, averageLength);
    const wt2 = sma(wt1, 4);
    const oscillator = for_every(wt1, wt2, function(primary, signal) {
        if (!isValidNumber(primary) || !isValidNumber(signal)) {
            return null;
        }
        return primary - signal;
    });
    return {
        wt1,
        wt2,
        oscillator
    };
}

function getFeatureSeries(featureType, paramA, paramB) {
    if (featureType === 'RSI') {
        const base = rsi(close, paramA);
        const smoothed = paramB > 1 ? sma(base, paramB) : base;
        return normalizeBounded(smoothed, 0, 100);
    }
    if (featureType === 'WT') {
        const waveTrend = waveTrendClassic(hlc3, paramA, paramB);
        return normalizeUnbounded(waveTrend.wt1, 60);
    }
    if (featureType === 'CCI') {
        const base = cci(close, paramA);
        const smoothed = paramB > 1 ? sma(base, paramB) : base;
        return normalizeUnbounded(smoothed, 100);
    }
    const adxObject = indicators.adx(paramA);
    return normalizeBounded(adxObject.adx, 0, 100);
}

function rationalQuadraticKernel(series, lookback, relativeWeight, startAtBar) {
    const out = newSeries(null);
    for (let barIndex = 0; barIndex < candleCount; barIndex++) {
        if (barIndex < startAtBar) {
            out[barIndex] = null;
            continue;
        }
        let weightedTotal = 0;
        let weightTotal = 0;
        const maxLag = Math.min(lookback - 1, barIndex);
        for (let lagIndex = 0; lagIndex <= maxLag; lagIndex++) {
            const value = series[barIndex - lagIndex];
            if (!isValidNumber(value)) {
                continue;
            }
            const weight = Math.pow(1 + ((lagIndex * lagIndex) / (((lookback * lookback) * 2) * relativeWeight)), -relativeWeight);
            weightedTotal += value * weight;
            weightTotal += weight;
        }
        out[barIndex] = weightTotal === 0 ? null : weightedTotal / weightTotal;
    }
    return out;
}

function gaussianKernel(series, lookback, startAtBar) {
    const out = newSeries(null);
    for (let barIndex = 0; barIndex < candleCount; barIndex++) {
        if (barIndex < startAtBar) {
            out[barIndex] = null;
            continue;
        }
        let weightedTotal = 0;
        let weightTotal = 0;
        const maxLag = Math.min(lookback - 1, barIndex);
        for (let lagIndex = 0; lagIndex <= maxLag; lagIndex++) {
            const value = series[barIndex - lagIndex];
            if (!isValidNumber(value)) {
                continue;
            }
            const weight = Math.exp(-((lagIndex * lagIndex) / (2 * lookback * lookback)));
            weightedTotal += value * weight;
            weightTotal += weight;
        }
        out[barIndex] = weightTotal === 0 ? null : weightedTotal / weightTotal;
    }
    return out;
}

function barsSince(signalSeries) {
    const out = newSeries(candleCount + 1);
    let counter = candleCount + 1;
    for (let index = 0; index < candleCount; index++) {
        if (signalSeries[index]) {
            counter = 0;
        } else if (counter < candleCount + 1) {
            counter += 1;
        }
        out[index] = counter;
    }
    return out;
}

function predictionColor(predictionValue, compressionFactor) {
    if (!isValidNumber(predictionValue)) {
        return neutralColor;
    }
    const boundedCompression = Math.max(1, compressionFactor);
    if (predictionValue > 0) {
        const ratio = clamp(predictionValue / boundedCompression, 0, 1);
        return mixColors(neutralColor, bullishGradientBase, ratio);
    }
    if (predictionValue < 0) {
        const ratio = clamp(Math.abs(predictionValue) / boundedCompression, 0, 1);
        return mixColors(bearishGradientBase, neutralColor, 1 - ratio);
    }
    return neutralColor;
}

function textCell(text, background) {
    return {
        text,
        color: 'var(--text-color)',
        background: background || 'transparent',
        paddingTop: 2,
        paddingBottom: 2,
        paddingLeft: 4,
        paddingRight: 4
    };
}

const feature1Series = getFeatureSeries(feature1Type, feature1ParamA, feature1ParamB);
const feature2Series = getFeatureSeries(feature2Type, feature2ParamA, feature2ParamB);
const feature3Series = getFeatureSeries(feature3Type, feature3ParamA, feature3ParamB);
const feature4Series = getFeatureSeries(feature4Type, feature4ParamA, feature4ParamB);
const feature5Series = getFeatureSeries(feature5Type, feature5ParamA, feature5ParamB);
const featureSeriesList = [feature1Series, feature2Series, feature3Series, feature4Series, feature5Series];

const emaFilterLine = ema(close, emaPeriod);
const smaFilterLine = sma(close, smaPeriod);
const atrFast = atr(1);
const atrSlow = atr(10);
const regimeBaseline = ema(ohlc4, 20);
const adx14 = indicators.adx(14).adx;

const isEmaUptrend = newSeries(true);
const isEmaDowntrend = newSeries(true);
const isSmaUptrend = newSeries(true);
const isSmaDowntrend = newSeries(true);
const volatilityFilter = newSeries(true);
const regimeFilter = newSeries(true);
const adxFilter = newSeries(true);

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    const priceValue = close[barIndex];
    const emaValue = emaFilterLine[barIndex];
    const smaValue = smaFilterLine[barIndex];
    isEmaUptrend[barIndex] = !useEmaFilter || (isValidNumber(priceValue) && isValidNumber(emaValue) && priceValue > emaValue);
    isEmaDowntrend[barIndex] = !useEmaFilter || (isValidNumber(priceValue) && isValidNumber(emaValue) && priceValue < emaValue);
    isSmaUptrend[barIndex] = !useSmaFilter || (isValidNumber(priceValue) && isValidNumber(smaValue) && priceValue > smaValue);
    isSmaDowntrend[barIndex] = !useSmaFilter || (isValidNumber(priceValue) && isValidNumber(smaValue) && priceValue < smaValue);

    const fastAtrValue = atrFast[barIndex];
    const slowAtrValue = atrSlow[barIndex];
    volatilityFilter[barIndex] = !useVolatilityFilter || (isValidNumber(fastAtrValue) && isValidNumber(slowAtrValue) && fastAtrValue > slowAtrValue);

    const baselineValue = regimeBaseline[barIndex];
    const previousBaseline = barIndex > 0 ? regimeBaseline[barIndex - 1] : null;
    const volatilityDenominator = atrSlow[barIndex];
    const regimeValue = isValidNumber(baselineValue) && isValidNumber(previousBaseline) && isValidNumber(volatilityDenominator) && volatilityDenominator !== 0
        ? ((baselineValue - previousBaseline) / volatilityDenominator) * 100
        : 0;
    regimeFilter[barIndex] = !useRegimeFilter || regimeValue > regimeThreshold;

    const adxValue = adx14[barIndex];
    adxFilter[barIndex] = !useAdxFilter || (isValidNumber(adxValue) && adxValue >= adxThreshold);
}

const yTrain = newSeries(0);
for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    if (barIndex + 4 >= candleCount) {
        yTrain[barIndex] = 0;
        continue;
    }
    if (sourceSeries[barIndex + 4] > sourceSeries[barIndex]) {
        yTrain[barIndex] = 1;
    } else if (sourceSeries[barIndex + 4] < sourceSeries[barIndex]) {
        yTrain[barIndex] = -1;
    } else {
        yTrain[barIndex] = 0;
    }
}

const predictionSeries = newSeries(0);
const signalSeries = newSeries(0);

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    const previousSignal = barIndex > 0 ? signalSeries[barIndex - 1] : 0;
    if (barIndex < maxBarsBackIndex) {
        signalSeries[barIndex] = previousSignal;
        continue;
    }

    const trainEnd = barIndex - 4;
    if (trainEnd <= 0) {
        signalSeries[barIndex] = previousSignal;
        continue;
    }

    const trainStart = Math.max(0, trainEnd - maxBarsBack + 1);
    let lastDistance = -1;
    const localDistances = [];
    const localPredictions = [];

    for (let trainIndex = trainStart; trainIndex <= trainEnd; trainIndex++) {
        if ((trainIndex % 4) === 0) {
            continue;
        }
        let distance = 0;
        let validDistance = true;
        for (let featureIndex = 0; featureIndex < featureCount; featureIndex++) {
            const currentFeatureValue = featureSeriesList[featureIndex][barIndex];
            const historicalFeatureValue = featureSeriesList[featureIndex][trainIndex];
            if (!isValidNumber(currentFeatureValue) || !isValidNumber(historicalFeatureValue)) {
                validDistance = false;
                break;
            }
            distance += Math.log(1 + Math.abs(currentFeatureValue - historicalFeatureValue));
        }

        if (!validDistance) {
            continue;
        }

        if (distance >= lastDistance) {
            lastDistance = distance;
            localDistances.push(distance);
            localPredictions.push(yTrain[trainIndex]);
            if (localPredictions.length > neighborsCount) {
                const quartileIndex = Math.min(localDistances.length - 1, Math.round((neighborsCount * 3) / 4));
                lastDistance = localDistances[quartileIndex];
                localDistances.shift();
                localPredictions.shift();
            }
        }
    }

    const predictionValue = localPredictions.length > 0 ? sumNumbers(localPredictions) : 0;
    predictionSeries[barIndex] = predictionValue;

    const filterAll = volatilityFilter[barIndex] && regimeFilter[barIndex] && adxFilter[barIndex];
    if (predictionValue > 0 && filterAll) {
        signalSeries[barIndex] = 1;
    } else if (predictionValue < 0 && filterAll) {
        signalSeries[barIndex] = -1;
    } else {
        signalSeries[barIndex] = previousSignal;
    }
}

const kernelEstimate = rationalQuadraticKernel(sourceSeries, kernelLookback, kernelRelativeWeight, kernelStartAtBar);
const gaussianLookback = Math.max(1, kernelLookback - kernelLag);
const kernelEstimateLagged = gaussianKernel(sourceSeries, gaussianLookback, kernelStartAtBar);
const kernelColorSeries = newSeries(transparentColor);
const isBullishRate = newSeries(false);
const isBearishRate = newSeries(false);
const isBullishChange = newSeries(false);
const isBearishChange = newSeries(false);
const isBullishCrossAlert = newSeries(false);
const isBearishCrossAlert = newSeries(false);
const isBullishSmooth = newSeries(false);
const isBearishSmooth = newSeries(false);
const alertBullish = newSeries(false);
const alertBearish = newSeries(false);
const isBullish = newSeries(true);
const isBearish = newSeries(true);

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    const currentKernel = kernelEstimate[barIndex];
    const previousKernel = barIndex > 0 ? kernelEstimate[barIndex - 1] : null;
    const twoBarsBackKernel = barIndex > 1 ? kernelEstimate[barIndex - 2] : null;
    const currentLagged = kernelEstimateLagged[barIndex];
    const previousLagged = barIndex > 0 ? kernelEstimateLagged[barIndex - 1] : null;

    const bullishRateNow = isValidNumber(currentKernel) && isValidNumber(previousKernel) && previousKernel < currentKernel;
    const bearishRateNow = isValidNumber(currentKernel) && isValidNumber(previousKernel) && previousKernel > currentKernel;
    const wasBullishRate = isValidNumber(twoBarsBackKernel) && isValidNumber(previousKernel) && twoBarsBackKernel < previousKernel;
    const wasBearishRate = isValidNumber(twoBarsBackKernel) && isValidNumber(previousKernel) && twoBarsBackKernel > previousKernel;

    isBullishRate[barIndex] = bullishRateNow;
    isBearishRate[barIndex] = bearishRateNow;
    isBullishChange[barIndex] = bullishRateNow && wasBearishRate;
    isBearishChange[barIndex] = bearishRateNow && wasBullishRate;

    const bullishCross = isValidNumber(previousLagged) && isValidNumber(previousKernel) && isValidNumber(currentLagged) && isValidNumber(currentKernel)
        && previousLagged < previousKernel && currentLagged >= currentKernel;
    const bearishCross = isValidNumber(previousLagged) && isValidNumber(previousKernel) && isValidNumber(currentLagged) && isValidNumber(currentKernel)
        && previousLagged > previousKernel && currentLagged <= currentKernel;
    isBullishCrossAlert[barIndex] = bullishCross;
    isBearishCrossAlert[barIndex] = bearishCross;

    const bullishSmooth = isValidNumber(currentLagged) && isValidNumber(currentKernel) && currentLagged >= currentKernel;
    const bearishSmooth = isValidNumber(currentLagged) && isValidNumber(currentKernel) && currentLagged <= currentKernel;
    isBullishSmooth[barIndex] = bullishSmooth;
    isBearishSmooth[barIndex] = bearishSmooth;

    alertBullish[barIndex] = useKernelSmoothing ? bullishCross : isBullishChange[barIndex];
    alertBearish[barIndex] = useKernelSmoothing ? bearishCross : isBearishChange[barIndex];

    isBullish[barIndex] = !useKernelFilter || (useKernelSmoothing ? bullishSmooth : bullishRateNow);
    isBearish[barIndex] = !useKernelFilter || (useKernelSmoothing ? bearishSmooth : bearishRateNow);

    if (!showKernelEstimate) {
        kernelColorSeries[barIndex] = transparentColor;
    } else if (useKernelSmoothing) {
        kernelColorSeries[barIndex] = bullishSmooth ? bullishKernelColor : bearishKernelColor;
    } else {
        kernelColorSeries[barIndex] = bullishRateNow ? bullishKernelColor : bearishKernelColor;
    }
}

const barsHeld = newSeries(0);
const isHeldFourBars = newSeries(false);
const isHeldLessThanFourBars = newSeries(false);
const isDifferentSignalType = newSeries(false);
const isEarlySignalFlip = newSeries(false);
const isBuySignal = newSeries(false);
const isSellSignal = newSeries(false);
const isLastSignalBuy = newSeries(false);
const isLastSignalSell = newSeries(false);
const isNewBuySignal = newSeries(false);
const isNewSellSignal = newSeries(false);

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    const previousSignal = barIndex > 0 ? signalSeries[barIndex - 1] : 0;
    const changed = signalSeries[barIndex] !== previousSignal;
    isDifferentSignalType[barIndex] = changed;
    barsHeld[barIndex] = changed ? 0 : (barIndex > 0 ? barsHeld[barIndex - 1] + 1 : 0);
    isHeldFourBars[barIndex] = barsHeld[barIndex] === 4;
    isHeldLessThanFourBars[barIndex] = barsHeld[barIndex] > 0 && barsHeld[barIndex] < 4;

    const priorChange1 = barIndex > 0 ? isDifferentSignalType[barIndex - 1] : false;
    const priorChange2 = barIndex > 1 ? isDifferentSignalType[barIndex - 2] : false;
    const priorChange3 = barIndex > 2 ? isDifferentSignalType[barIndex - 3] : false;
    isEarlySignalFlip[barIndex] = changed && (priorChange1 || priorChange2 || priorChange3);

    isBuySignal[barIndex] = signalSeries[barIndex] === 1 && isEmaUptrend[barIndex] && isSmaUptrend[barIndex];
    isSellSignal[barIndex] = signalSeries[barIndex] === -1 && isEmaDowntrend[barIndex] && isSmaDowntrend[barIndex];

    const lastIndex = barIndex - 4;
    isLastSignalBuy[barIndex] = lastIndex >= 0 && signalSeries[lastIndex] === 1 && isEmaUptrend[lastIndex] && isSmaUptrend[lastIndex];
    isLastSignalSell[barIndex] = lastIndex >= 0 && signalSeries[lastIndex] === -1 && isEmaDowntrend[lastIndex] && isSmaDowntrend[lastIndex];
    isNewBuySignal[barIndex] = isBuySignal[barIndex] && changed;
    isNewSellSignal[barIndex] = isSellSignal[barIndex] && changed;
}

const startLongTrade = newSeries(false);
const startShortTrade = newSeries(false);
for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    startLongTrade[barIndex] = isNewBuySignal[barIndex] && isBullish[barIndex] && isEmaUptrend[barIndex] && isSmaUptrend[barIndex];
    startShortTrade[barIndex] = isNewSellSignal[barIndex] && isBearish[barIndex] && isEmaDowntrend[barIndex] && isSmaDowntrend[barIndex];
}

const barsSinceShortEntry = barsSince(startShortTrade);
const barsSinceBullishAlert = barsSince(alertBullish);
const barsSinceLongEntry = barsSince(startLongTrade);
const barsSinceBearishAlert = barsSince(alertBearish);
const isValidShortExit = newSeries(false);
const isValidLongExit = newSeries(false);
const endLongTradeDynamic = newSeries(false);
const endShortTradeDynamic = newSeries(false);
const endLongTradeStrict = newSeries(false);
const endShortTradeStrict = newSeries(false);
const endLongTrade = newSeries(false);
const endShortTrade = newSeries(false);
const isDynamicExitValid = !useEmaFilter && !useSmaFilter && !useKernelSmoothing;

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    isValidShortExit[barIndex] = barsSinceBullishAlert[barIndex] > barsSinceShortEntry[barIndex];
    isValidLongExit[barIndex] = barsSinceBearishAlert[barIndex] > barsSinceLongEntry[barIndex];

    const previousLongExitValidity = barIndex > 0 ? isValidLongExit[barIndex - 1] : false;
    const previousShortExitValidity = barIndex > 0 ? isValidShortExit[barIndex - 1] : false;
    endLongTradeDynamic[barIndex] = isBearishChange[barIndex] && previousLongExitValidity;
    endShortTradeDynamic[barIndex] = isBullishChange[barIndex] && previousShortExitValidity;

    const startLongFourBarsAgo = barIndex >= 4 ? startLongTrade[barIndex - 4] : false;
    const startShortFourBarsAgo = barIndex >= 4 ? startShortTrade[barIndex - 4] : false;
    endLongTradeStrict[barIndex] = (((isHeldFourBars[barIndex] && isLastSignalBuy[barIndex]) || (isHeldLessThanFourBars[barIndex] && isNewSellSignal[barIndex] && isLastSignalBuy[barIndex])) && startLongFourBarsAgo);
    endShortTradeStrict[barIndex] = (((isHeldFourBars[barIndex] && isLastSignalSell[barIndex]) || (isHeldLessThanFourBars[barIndex] && isNewBuySignal[barIndex] && isLastSignalSell[barIndex])) && startShortFourBarsAgo);

    endLongTrade[barIndex] = useDynamicExits && isDynamicExitValid ? endLongTradeDynamic[barIndex] : endLongTradeStrict[barIndex];
    endShortTrade[barIndex] = useDynamicExits && isDynamicExitValid ? endShortTradeDynamic[barIndex] : endShortTradeStrict[barIndex];
}

const compressionFactor = neighborsCount / colorCompression;
const predictionTextColors = newSeries(neutralColor);
const candleColors = newSeries(null);
const buyLabels = newSeries(null);
const sellLabels = newSeries(null);
const exitLongLabels = newSeries(null);
const exitShortLabels = newSeries(null);
const predictionLabelsAbove = newSeries(null);
const predictionLabelsBelow = newSeries(null);
const backtestStream = newSeries(0);

for (let barIndex = 0; barIndex < candleCount; barIndex++) {
    const predictionValue = predictionSeries[barIndex];
    const basePredictionColor = predictionColor(predictionValue, compressionFactor);
    predictionTextColors[barIndex] = basePredictionColor;
    candleColors[barIndex] = showBarColors ? withAlpha(basePredictionColor, 0.50) : null;

    if (startLongTrade[barIndex]) {
        buyLabels[barIndex] = 'Buy';
    }
    if (startShortTrade[barIndex]) {
        sellLabels[barIndex] = 'Sell';
    }
    if (showExits && endLongTrade[barIndex]) {
        exitLongLabels[barIndex] = 'Exit L';
    }
    if (showExits && endShortTrade[barIndex]) {
        exitShortLabels[barIndex] = 'Exit S';
    }
    if (showBarPredictions && predictionValue > 0) {
        predictionLabelsAbove[barIndex] = String(predictionValue);
    }
    if (showBarPredictions && predictionValue < 0) {
        predictionLabelsBelow[barIndex] = String(predictionValue);
    }

    if (startLongTrade[barIndex]) {
        backtestStream[barIndex] = 1;
    } else if (endLongTrade[barIndex]) {
        backtestStream[barIndex] = 2;
    } else if (startShortTrade[barIndex]) {
        backtestStream[barIndex] = -1;
    } else if (endShortTrade[barIndex]) {
        backtestStream[barIndex] = -2;
    } else {
        backtestStream[barIndex] = 0;
    }
}

let totalWins = 0;
let totalLosses = 0;
let totalTrades = 0;
let totalEarlySignalFlips = 0;
let currentPosition = 0;
let entryPrice = null;

for (let barIndex = Math.max(maxBarsBackIndex, 0); barIndex < candleCount; barIndex++) {
    if (isEarlySignalFlip[barIndex]) {
        totalEarlySignalFlips += 1;
    }

    if (currentPosition === 0) {
        if (startLongTrade[barIndex]) {
            currentPosition = 1;
            totalTrades += 1;
            entryPrice = useWorstCase ? close[barIndex] : ohlc4[barIndex];
        } else if (startShortTrade[barIndex]) {
            currentPosition = -1;
            totalTrades += 1;
            entryPrice = useWorstCase ? close[barIndex] : ohlc4[barIndex];
        }
        continue;
    }

    if (currentPosition === 1 && endLongTrade[barIndex]) {
        const exitPrice = useWorstCase ? close[barIndex] : ohlc4[barIndex];
        if (isValidNumber(entryPrice) && isValidNumber(exitPrice) && exitPrice > entryPrice) {
            totalWins += 1;
        } else {
            totalLosses += 1;
        }
        currentPosition = 0;
        entryPrice = null;
        continue;
    }

    if (currentPosition === -1 && endShortTrade[barIndex]) {
        const exitPrice = useWorstCase ? close[barIndex] : ohlc4[barIndex];
        if (isValidNumber(entryPrice) && isValidNumber(exitPrice) && exitPrice < entryPrice) {
            totalWins += 1;
        } else {
            totalLosses += 1;
        }
        currentPosition = 0;
        entryPrice = null;
    }
}

const winRate = totalTrades > 0 ? totalWins / totalTrades : 0;
const winLossRatio = totalLosses > 0 ? totalWins / totalLosses : totalWins;
const tradeStatsRows = [];
if (showTradeStats) {
    tradeStatsRows.push({
        cells: [{
            text: 'Trade Stats (calibration only)',
            color: 'var(--text-color)',
            colspan: 2,
            fontWeight: 'bold'
        }]
    });
    tradeStatsRows.push({ cells: [textCell('Win Rate'), textCell(totalTrades > 0 ? `${(winRate * 100).toFixed(1)}%` : 'n/a')] });
    tradeStatsRows.push({ cells: [textCell('Trades'), textCell(`${totalTrades} (${totalWins}|${totalLosses})`)] });
    tradeStatsRows.push({ cells: [textCell('WL Ratio'), textCell(totalLosses > 0 ? winLossRatio.toFixed(2) : (totalWins > 0 ? String(totalWins) : 'n/a'))] });
    tradeStatsRows.push({ cells: [textCell('Early Signal Flips'), textCell(String(totalEarlySignalFlips))] });
    tradeStatsRows.push({ cells: [textCell('Kernel Mode'), textCell(useKernelFilter ? (useKernelSmoothing ? 'Smoothed' : 'Rate') : 'Off')] });
}

paint(kernelEstimate, {
    name: 'Kernel Estimate',
    color: kernelColorSeries,
    thickness: 2
});

paint(buyLabels, {
    name: 'Buy Labels',
    style: 'labels_below',
    backgroundColor: bullishMarkerColor,
    color: 'white',
    verticalOffset: 12
});

paint(sellLabels, {
    name: 'Sell Labels',
    style: 'labels_above',
    backgroundColor: bearishMarkerColor,
    color: 'white',
    verticalOffset: 12
});

paint(exitLongLabels, {
    name: 'Exit Long Labels',
    style: 'labels_above',
    backgroundColor: exitLongColor,
    color: 'black',
    verticalOffset: 8
});

paint(exitShortLabels, {
    name: 'Exit Short Labels',
    style: 'labels_below',
    backgroundColor: exitShortColor,
    color: 'white',
    verticalOffset: 8
});

paint(predictionLabelsAbove, {
    name: 'Prediction Labels Above',
    style: 'labels_above',
    backgroundColor: transparentColor,
    color: predictionTextColors,
    verticalOffset: useAtrOffset ? 18 : Math.round(6 + (barPredictionsOffset * 2)),
    editorHidden: false,
    hideInLegend: true
});

paint(predictionLabelsBelow, {
    name: 'Prediction Labels Below',
    style: 'labels_below',
    backgroundColor: transparentColor,
    color: predictionTextColors,
    verticalOffset: useAtrOffset ? 18 : Math.round(6 + (barPredictionsOffset * 2)),
    editorHidden: false,
    hideInLegend: true
});

paint(backtestStream, {
    name: 'Backtest Stream',
    hidden: true,
    hideInLegend: true,
    hideInScriptEditor: false,
    ignoreWhenScaling: true
});

color_candles(candleColors);

paint_overlay('Trade Stats Overlay', { position: 'top_right', order: 'above_all' }, {
    background: 'var(--background-color)',
    rows: tradeStatsRows
});

register_signal(startLongTrade, 'Open Long');
register_signal(endLongTrade, 'Close Long');
register_signal(startShortTrade, 'Open Short');
register_signal(endShortTrade, 'Close Short');
register_signal(startLongTrade.map(function(value, index) {
    return value || startShortTrade[index];
}), 'Open Position');
register_signal(endLongTrade.map(function(value, index) {
    return value || endShortTrade[index];
}), 'Close Position');
register_signal(alertBullish, 'Kernel Bullish Change');
register_signal(alertBearish, 'Kernel Bearish Change');
register_signal(isEarlySignalFlip, 'Early Signal Flip');
register_signal(predictionSeries.map(function(value) {
    return value > 0;
}), 'Prediction Positive');
register_signal(predictionSeries.map(function(value) {
    return value < 0;
}), 'Prediction Negative');
