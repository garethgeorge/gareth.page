+++
title = "Airnow AQI surprises"
date = "2021-08-01"
description = ""
tags = [
  "js",
]
+++


### Some background

During the pandemic, with way too much time on my hands, wildfires blazing across the west coast, and official purple air sensors [costing pushing $269](https://web.archive.org/web/20230209001957/https://www2.purpleair.com/products/purpleair-flex) I decided that this was certainly something I could buid myself! I had a few ESP32 boards laying around and was able to quickly order a PMS5003 laser sensor that could detect a range of particle sizes from 0.3um to 10um.

![PMS5003](/img/pms5003.jpg)

What surprised me was, with everything wired up and my sensor outputting a deluge of data points informing me about the sizes and distributions of particles getting sucked into my lungs, I had no idea what any of it meant! I was used to checking airnow.gov and being presented with a nice colorful visual informing me how bad (or optimistically how good) the air might be on an AQI range. 

### What is AQI?

I quickly realized that, though I knew semantically that AQI values roughly fall into the following ranges of badness:

| AQI | Category | Health Concern |
|---|---|---|
| 0-50 | Good | Air quality is considered satisfactory, and air pollution poses little or no risk. |
| 51-100 | Moderate | Air quality is acceptable; however, for some pollutants there may be a moderate health concern for a very small number of people. For example, people who are unusually sensitive to ozone may experience respiratory symptoms. |
| 101-150 | Unhealthy for Sensitive Groups | Members of sensitive groups may experience health effects that are more serious. |
| 151-200 | Unhealthy | Everyone may begin to experience some adverse health effects, and members of the sensitive groups may experience more serious effects. |
| 201-300 | Very Unhealthy | This would trigger a health alert signifying that everyone may experience more serious health effects. |
| 301+ | Hazardous | This would trigger health warnings of emergency conditions. The entire population is more likely to be affected. |

I didn't have any grasp of how these values were being calculated. It turns out that these values are mapped by a non-linear function from the raw PM2.5 and PM10 values that my sensor was spewing. The best authoritative source I found on this mapping was the official [Technical Assistance Document for the Reporting of Daily Air Quality](https://www.airnow.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf) provided by airnow.gov (rather unconveniently... as a PDF).

### The AQI formula

No nice simple one-liner formulas here. To my surprise, the AQI is a non-linear mapping that is defined largely by lookup table and linear-interpolation between values. The table values are provided in the PDF linked above, but I've additionally copied them above:

| AQI Level | PM2.5 Breakpoint | PM10 Breakpoint |
|---|---|---|
| 0 | 0.0 | 0 |
| 50 | 12.0 | 54 |
| 100 | 35.4 | 154 |
| 150 | 150.4 | 250.4 |
| 200 | 250.4 | 354.0 |
| 300 | 350.4 | 424.0 |
| 400 | 500.4 | 504.0 |

To compute the AQI value, one first finds the pair of values in the table that bracket the PM2.5 and PM10 values. Then, the AQI is computed by linearly interpolating between the two bracketing values. For example, if the PM2.5 value is 20 and the PM10 value is 15 then the AQI is computed as follows:

```txt
AQI_{particulate type} = (AQI_high - AQI_low)/(PM_high - PM_low) * (PM - PM_low) + AQI_low
AQI_2.5 = (50 - 0)/(12.0 - 0.0) * (20 - 0.0) + 0 = 83.3
AQI_10 = (50 - 0)/(54.0 - 0.0) * (15 - 0.0) + 0 = 13.9
```

The raw AQI value is then simply the worse of the two! In this case that is `MAX(AQI_2.5, AQI_10) = 83.3` which is our final AQI.

### As JavaScript

See package on [GitHub](https://github.com/garethgeorge/airnow-aqi-js/tree/master)

```ts

export type ParticulateType = "pm2.5" | "pm10";

export interface Measurement {
  type: ParticulateType;
  ppm: number;
}

const breakpoints: { [pt: string]: number[] } = {
  "pm2.5": [0, 12, 35.4, 150.4, 250.4, 350.4, 500.4],
  pm10: [0, 54, 154, 254, 354, 424, 504, 604],
};

const aqiLevels = [0, 50, 100, 150, 200, 300, 400, 500];

const findIdx = (values, val) => {
  for (let i = 1; i < values.length; ++i) {
    if (values[i - 1] <= val && val < values[i]) {
      return i - 1;
    }
  }
  return -1;
};

const measurementAqi = (measurement: Measurement) => {
  const idx = findIdx(breakpoints[measurement.type], measurement.ppm);
  if (idx == -1) {
    return aqiLevels[aqiLevels.length - 1];
  }

  const ppm = measurement.ppm,
    ilo = aqiLevels[idx],
    ihi = aqiLevels[idx + 1],
    bplo = breakpoints[measurement.type][idx],
    bphi = breakpoints[measurement.type][idx + 1];
  return ((ihi - ilo) / (bphi - bplo)) * (ppm - bplo) + ilo;
};

export const computeAqi = (measurements: Measurement[]) => {
  if (measurements.length === 0) {
    return -1;
  }
  return (
    Math.round(
      measurements.map(measurementAqi).reduce((p, c) => Math.max(p, c)) * 10
    ) / 10
  );
};
```

## Future Reading

In a followup article I'll be sure to document how the air quality sensor itself works and how I made use of this package to visualize AQI values.