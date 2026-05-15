<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Fingerprint Entropy — Critical Demo v3</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; line-height: 1.6; }
    h1 { font-size: 26px; }
    table { border-collapse: collapse; width: 100%; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
    th { background: #f5f5f5; }
    .score { margin-top: 20px; font-size: 18px; }
    .note { margin-top: 10px; font-size: 14px; color: #444; }
    button { margin-top: 15px; padding: 10px 15px; cursor: pointer; margin-right: 10px; }
    .hidden { display: none; margin-top: 15px; padding: 15px; border: 1px solid #ddd; background: #fafafa; }
  </style>
</head>
<body>

<h1>Browser Fingerprint & Entropy (Critical Demo v3)</h1>
<p>This tool estimates entropy — and shows why that estimate can be misleading.</p>

<table id="dataTable">
  <thead>
    <tr>
      <th>Signal</th>
      <th>Value</th>
      <th>Estimated Entropy (bits)</th>
    </tr>
  </thead>
  <tbody></tbody>
</table>

<div class="score" id="totalScore"></div>
<div class="note" id="assumptionNote"></div>
<div class="score" id="confidence"></div>
<div class="note" id="philosophy"></div>

<button onclick="toggleExplanation()">Why is this result misleading?</button>
<button onclick="simulateUser()">Simulate another user</button>

<div id="explanation" class="hidden">
  <strong>Three core problems:</strong>
  <ul>
    <li><strong>False independence:</strong> Signals like User Agent and Platform are correlated.</li>
    <li><strong>No real distribution:</strong> Entropy depends on real-world probabilities, not assumed counts.</li>
    <li><strong>Signal ≠ Evidence:</strong> A unique signal does not guarantee trackability.</li>
  </ul>
</div>

<script>
function log2(x) {
  return Math.log(x) / Math.log(2);
}

function bitsFromProbability(p) {
  return -log2(Math.max(p, 1e-6));
}

function normalizeText(v) {
  return String(v ?? "").trim().toLowerCase();
}

function parseResolution(v) {
  const m = String(v).match(/(\d+)\s*x\s*(\d+)/i);
  if (!m) return null;
  return { w: Number(m[1]), h: Number(m[2]) };
}

function browserFamilyFromUA(ua) {
  const s = normalizeText(ua);
  if (s.includes("edg/") || s.includes("edge/")) return "edge";
  if (s.includes("firefox/")) return "firefox";
  if (s.includes("chrome/") || s.includes("crios/") || s.includes("chromium/")) return "chrome";
  if (s.includes("safari/")) return "safari";
  return "other";
}

const fakeDataOptions = {
  "User Agent": ["Chrome", "Firefox", "Safari", "Edge"],
  "Language": ["en", "ar", "fr", "de"],
  "Platform": ["Win32", "Linux", "MacIntel"],
  "Screen Resolution": ["1920x1080", "1366x768", "1280x720"],
  "Color Depth": [24, 32],
  "Timezone": ["Asia/Baghdad", "Europe/Berlin", "America/New_York"],
  "Cookies Enabled": [true, false]
};

const SIGNAL_MODEL = {
  "User Agent": {
    chrome: 0.64,
    safari: 0.20,
    edge: 0.09,
    firefox: 0.05,
    other: 0.02
  },

  "Language": {
    "en-us": 0.15,
    "en-gb": 0.03,
    en: 0.22,
    ar: 0.05,
    "ar-iq": 0.01,
    "ar-sa": 0.01,
    fr: 0.04,
    de: 0.04,
    es: 0.07,
    "pt-br": 0.02,
    ru: 0.03,
    ja: 0.02,
    ko: 0.02,
    zh: 0.08,
    "zh-cn": 0.03,
    "zh-tw": 0.02,
    other: 0.19
  },

  "Platform": {
    win32: 0.58,
    "win64": 0.05,
    "windows": 0.05,
    macintel: 0.20,
    mac: 0.02,
    linux: 0.05,
    android: 0.03,
    iphone: 0.015,
    ipad: 0.01,
    other: 0.015
  },

  "Screen Resolution": {
    "1920x1080": 0.20,
    "1366x768": 0.16,
    "1536x864": 0.08,
    "1440x900": 0.07,
    "1600x900": 0.05,
    "1280x720": 0.05,
    "2560x1440": 0.04,
    "1680x1050": 0.03,
    "1280x800": 0.03,
    "375x812": 0.015,
    "390x844": 0.015,
    "412x915": 0.015,
    "414x896": 0.015,
    "430x932": 0.01,
    "3440x1440": 0.01,
    other: 0.145
  },

  "Color Depth": {
    24: 0.80,
    32: 0.18,
    other: 0.02
  },

  "Timezone": {
    "america/new_york": 0.08,
    "america/los_angeles": 0.05,
    "america/chicago": 0.04,
    "america/denver": 0.02,
    "america/phoenix": 0.015,
    "europe/london": 0.04,
    "europe/berlin": 0.03,
    "europe/paris": 0.02,
    "europe/madrid": 0.02,
    "europe/rome": 0.02,
    "asia/tokyo": 0.03,
    "asia/shanghai": 0.04,
    "asia/kolkata": 0.03,
    "asia/dubai": 0.02,
    "asia/baghdad": 0.006,
    "asia/seoul": 0.02,
    "australia/sydney": 0.015,
    "etc/utc": 0.02,
    other: 0.47
  },

  "Cookies Enabled": {
    true: 0.97,
    false: 0.03
  }
};

function getRealSignals() {
  return {
    "User Agent": navigator.userAgent,
    "Language": navigator.language,
    "Platform": navigator.platform,
    "Screen Resolution": screen.width + "x" + screen.height,
    "Color Depth": screen.colorDepth,
    "Timezone": Intl.DateTimeFormat().resolvedOptions().timeZone,
    "Cookies Enabled": navigator.cookieEnabled
  };
}

function getFakeSignals() {
  let obj = {};
  for (let key in fakeDataOptions) {
    const arr = fakeDataOptions[key];
    obj[key] = arr[Math.floor(Math.random() * arr.length)];
  }
  return obj;
}

function entropyModel(signal, value) {
  switch (signal) {
    case "User Agent": {
      const family = browserFamilyFromUA(value);
      return bitsFromProbability(SIGNAL_MODEL["User Agent"][family] ?? SIGNAL_MODEL["User Agent"].other);
    }

    case "Language": {
      const raw = normalizeText(value).replace("_", "-");
      const primary = raw.split("-")[0] || "other";
      const exactProb = SIGNAL_MODEL["Language"][raw];
      const baseProb = SIGNAL_MODEL["Language"][primary];
      const p = exactProb ?? baseProb ?? SIGNAL_MODEL["Language"].other;
      return bitsFromProbability(p);
    }

    case "Platform": {
      const s = normalizeText(value);
      let key = "other";
      if (s.includes("win")) key = "win32";
      else if (s.includes("mac")) key = "macintel";
      else if (s.includes("linux")) key = "linux";
      else if (s.includes("android")) key = "android";
      else if (s.includes("iphone")) key = "iphone";
      else if (s.includes("ipad")) key = "ipad";
      return bitsFromProbability(SIGNAL_MODEL["Platform"][key] ?? SIGNAL_MODEL["Platform"].other);
    }

    case "Screen Resolution": {
      const parsed = parseResolution(value);
      if (!parsed) return bitsFromProbability(SIGNAL_MODEL["Screen Resolution"].other);

      const key = `${parsed.w}x${parsed.h}`;
      const exact = SIGNAL_MODEL["Screen Resolution"][key];
      if (exact) return bitsFromProbability(exact);

      let best = SIGNAL_MODEL["Screen Resolution"].other;
      for (const k in SIGNAL_MODEL["Screen Resolution"]) {
        if (k === "other") continue;
        const [tw, th] = k.split("x").map(Number);
        const dw = Math.abs(Math.log2(parsed.w / tw));
        const dh = Math.abs(Math.log2(parsed.h / th));
        const distance = Math.sqrt(dw * dw + dh * dh);
        const p = SIGNAL_MODEL["Screen Resolution"][k] * Math.exp(-3.5 * distance);
        if (p > best) best = p;
      }

      return bitsFromProbability(best);
    }

    case "Color Depth": {
      const depth = Number(value);
      const p = SIGNAL_MODEL["Color Depth"][depth] ?? SIGNAL_MODEL["Color Depth"].other;
      return bitsFromProbability(p);
    }

    case "Timezone": {
      const tz = normalizeText(value);
      const p = SIGNAL_MODEL["Timezone"][tz] ?? SIGNAL_MODEL["Timezone"].other;
      return bitsFromProbability(p);
    }

    case "Cookies Enabled": {
      const p = value === true ? SIGNAL_MODEL["Cookies Enabled"].true : SIGNAL_MODEL["Cookies Enabled"].false;
      return bitsFromProbability(p);
    }

    default:
      return 1;
  }
}

function render(signals) {
  const tbody = document.querySelector("#dataTable tbody");
  tbody.innerHTML = "";

  let totalEntropy = 0;

  for (let key in signals) {
    const value = signals[key];
    const entropy = entropyModel(key, value);
    totalEntropy += entropy;

    const row = document.createElement("tr");
    row.innerHTML = `<td>${key}</td><td>${value}</td><td>${entropy.toFixed(2)}</td>`;
    tbody.appendChild(row);
  }

  document.getElementById("totalScore").innerText =
    "Estimated Entropy: " + totalEntropy.toFixed(2) + " bits";

  document.getElementById("assumptionNote").innerText =
    "Assumption: Each signal is weighted by how common its value is in the population.";

  document.getElementById("confidence").innerText =
    "Confidence: " + (totalEntropy > 20 ? "Higher" : "Lower");

  document.getElementById("philosophy").innerText =
    "Two users can have similar signals — but different rarity profiles.";
}

function toggleExplanation() {
  const el = document.getElementById("explanation");
  el.classList.toggle("hidden");
}

function simulateUser() {
  const fakeSignals = getFakeSignals();
  render(fakeSignals);
}

// initial render
render(getRealSignals());
</script>

</body>
</html>
