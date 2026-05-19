# CLAUDE.md — MoneyMind: Portfolio Challenge
# Projektová dokumentace pro Claude Code

Tento soubor je automaticky načten Claude Code při každém spuštění v tomto adresáři.
Před zápisem jakéhokoliv kódu si přečti celý tento dokument.

---

## PŘEHLED PROJEKTU

**Název:** MoneyMind: Portfolio Challenge
**Typ:** Single-file webová aplikace (index.html)
**Jazyk hry:** Čeština
**Inspirace:** Robert Kiyosaki — Bohatý táta, Chudý táta
**Filosofie:** Aktiva dávají peníze DO kapsy. Pasiva berou peníze Z kapsy.

---

## TECHNICKÝ STACK

| Vrstva | Technologie | Poznámka |
|--------|-------------|----------|
| Markup | HTML5 | Vše v jednom souboru |
| Styl | CSS3 + CSS Variables | Žádný framework |
| Logika | Vanilla JS ES6+ | Žádný framework |
| Písma | Google Fonts via @import | Pouze Cormorant Garamond + DM Sans + JetBrains Mono |
| Audio | Web Audio API | Žádné externí soubory |
| AI | Anthropic API | claude-sonnet-4-20250514 |

**Zakázáno:** npm, webpack, React, Vue, Bootstrap, jQuery, CDN knihovny (kromě Google Fonts).

---

## STRUKTURA SOUBORU index.html

```
index.html
├── <head>
│   ├── SEO meta tagy (viz sekce SEO níže)
│   ├── @import Google Fonts
│   └── <style> — veškerý CSS
└── <body>
    ├── #screen-intro       — úvodní obrazovka
    ├── #screen-game        — herní obrazovka (kola 1–10)
    │   ├── .dashboard      — vždy viditelný panel nahoře
    │   ├── .market-board   — nabídka aktiv kola
    │   ├── .portfolio-panel — vlastněná aktiva + prodej
    │   └── .action-bar     — tlačítka Potvrdit / Přeskočit
    ├── #screen-result      — závěrečná analýza
    ├── #overlay-summary    — animovaný souhrn po každém kole
    ├── #btn-mute           — tlačítko ztlumení (fixed, bottom-right)
    └── <script>            — veškerý JavaScript
```

---

## JAVASCRIPT ARCHITEKTURA

Celá logika hry v jednom objektu `GameState`. Žádné globální proměnné mimo něj.

```javascript
const GameState = {
  // Stav hráče
  cash: 1200000,
  round: 1,
  portfolio: [],        // pole objektů { item, qty, currentValue, roundsRemaining }
  purchasesThisRound: [],
  history: [],          // co bylo koupeno/prodáno každé kolo

  // Vypočítávané hodnoty (gettery nebo metody)
  get netWorth() { ... },
  get monthlyIncome() { ... },
  get monthlyExpenses() { ... },
  get netMonthlyFlow() { ... },

  // Metody
  buyItem(itemId, qty) { ... },
  sellAsset(portfolioIndex) { ... },
  resolveRound() { ... },       // STEP 1–4 ekonomiky
  applyAppreciation() { ... },
  checkBondMaturities() { ... },
  evaluateSurvival() { ... },   // vrací 'svoboda'|'nezavislost'|'hrana'|'nestaci'
  reset() { ... }               // nová hra
};
```

---

## DESIGN SYSTÉM

### Barevná paleta (CSS Variables — použij všude, nikdy hardcode)

```css
:root {
  --bg-deep:        #03030a;
  --bg-primary:     #070710;
  --bg-card:        #0e0e1a;
  --bg-card-hover:  #141426;
  --glass:          rgba(255, 255, 255, 0.03);
  --glass-hover:    rgba(255, 255, 255, 0.06);
  --glass-border:   rgba(255, 255, 255, 0.07);
  --glass-active:   rgba(201, 168, 76, 0.08);
  --border-active:  rgba(201, 168, 76, 0.6);
  --gold:           #C9A84C;
  --gold-light:     #F0C060;
  --gold-dim:       rgba(201, 168, 76, 0.3);
  --green:          #22C55E;
  --green-dim:      rgba(34, 197, 94, 0.15);
  --red:            #EF4444;
  --red-dim:        rgba(239, 68, 68, 0.15);
  --amber:          #F59E0B;
  --blue:           #3B82F6;
  --text-primary:   #F0F0F8;
  --text-secondary: #8888A8;
  --text-muted:     #44445A;
  --text-gold:      #C9A84C;
  --radius-sm:      10px;
  --radius-md:      16px;
  --radius-lg:      24px;
  --shadow-card:    0 8px 40px rgba(0,0,0,0.7);
  --shadow-gold:    0 0 30px rgba(201,168,76,0.2);
  --transition:     all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}
```

### Typografie

| Použití | Font | Weight | Velikost |
|---------|------|--------|----------|
| Hlavní nadpisy | Cormorant Garamond | 300 | clamp(72px, 12vw, 120px) |
| Herní nadpisy | Cormorant Garamond | 400 | 36–48px |
| UI texty, popisy | DM Sans | 400 / 600 | 13–16px |
| Čísla, peníze, stats | JetBrains Mono | 400 / 600 | 13–18px |

### Pozadí — 3 plovoucí orby (CSS keyframes)

```css
/* Orb 1: gold, top-left, 600px, blur 150px, opacity 0.2 */
/* Orb 2: blue #1E40AF, bottom-right, 500px, blur 130px, opacity 0.15 */
/* Orb 3: violet #4C1D95, center, 400px, blur 120px, opacity 0.10 */
/* Všechny: position fixed, pointer-events none, z-index 0 */
/* Animace: keyframes s translateX/Y, 12–18s loop, ease-in-out */
```

### Glassmorphism karty

```css
.card {
  background: var(--glass);
  backdrop-filter: blur(20px) saturate(180%);
  border: 1px solid var(--glass-border);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-card);
}
.card:hover {
  background: var(--glass-hover);
  border-color: var(--gold-dim);
  transform: translateY(-3px);
}
.card.selected {
  background: var(--glass-active);
  border-color: var(--border-active);
  box-shadow: var(--shadow-gold);
}
```

### Tlačítka

```css
/* Primární (gold filled) */
.btn-primary {
  background: linear-gradient(135deg, var(--gold), var(--gold-light));
  color: #070710;
  font-weight: 600;
  border-radius: 50px;
  padding: 16px 44px;
  transition: var(--transition), transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.btn-primary:hover { transform: scale(1.05); box-shadow: var(--shadow-gold); }

/* Ghost */
.btn-ghost {
  border: 1px solid var(--glass-border);
  color: var(--text-secondary);
  border-radius: 50px;
  padding: 12px 28px;
}
.btn-ghost:hover { border-color: var(--gold-dim); color: var(--text-gold); }
```

---

## KATEGORIE AKTIV — BARVY BADGE

| Kategorie | Barva pozadí badge |
|-----------|-------------------|
| Nemovitost | rgba(59, 130, 246, 0.2) — modrá |
| Akcie & ETF | rgba(34, 197, 94, 0.2) — zelená |
| Dluhopisy | rgba(245, 158, 11, 0.2) — amber |
| Podnikání | rgba(139, 92, 246, 0.2) — fialová |
| Spoření | rgba(100, 116, 139, 0.2) — ocelová |
| Životní styl | rgba(239, 68, 68, 0.2) — červená (pasivum!) |

---

## DASHBOARD — ROZLOŽENÍ

```
┌──────────────────────────────────────────────────────────────────┐
│ MoneyMind     [💶 XXX,XXX €]  [📊 X,XXX,XXX €]  [💚 X,XXX €/m]  Kolo 03/10 ████░░░░░ │
└──────────────────────────────────────────────────────────────────┘
```

- Výška: 80px desktop / 60px mobile
- Background: rgba(7, 7, 16, 0.92) + backdrop-filter: blur(20px)
- Border-bottom: 1px solid rgba(255,255,255,0.05)
- Position: fixed top 0, full width, z-index: 100
- Hodnoty: JetBrains Mono — cash=text-primary, NW=blue, income=green
- Progress bar: 4px height, gold fill, animated transition

---

## ANIMACE

### Přechody obrazovek
```css
/* Fade + translateY */
.screen { opacity: 0; transform: translateY(20px); }
.screen.active { opacity: 1; transform: translateY(0); transition: opacity 0.4s ease, transform 0.4s ease; }
```

### Kolo — souhrn overlay
- Plná obrazovka, tmavý podklad + gold orb uprostřed
- Řádky se výsledky "vyskakují" zpola přes translateY + opacity, delay 200ms mezi každým
- Celková délka: 2,5s nebo do kliknutí "Další kolo →"

### Čísla v závěru (odometer efekt)
```javascript
// Animuj číslo od 0 na finalValue za 1,5s
function animateNumber(el, finalValue, duration = 1500) {
  const start = performance.now();
  const update = (now) => {
    const progress = Math.min((now - start) / duration, 1);
    const eased = 1 - Math.pow(1 - progress, 3); // ease-out cubic
    el.textContent = formatCurrency(Math.round(finalValue * eased));
    if (progress < 1) requestAnimationFrame(update);
  };
  requestAnimationFrame(update);
}
```

### Typewriter pro AI text
```javascript
function typewrite(el, text, speed = 20) {
  el.textContent = '';
  let i = 0;
  const tick = () => {
    if (i < text.length) {
      el.textContent += text[i++];
      setTimeout(tick, speed);
    }
  };
  tick();
}
// Mezi odstavci: setTimeout 600ms pauza
```

### prefers-reduced-motion
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 150ms !important;
  }
}
```

---

## WEB AUDIO — AMBIENT ENGINE

Spustit na první klik/tap (browser autoplay policy). Žádné externí soubory.

```javascript
const AudioEngine = {
  ctx: null, master: null, isMuted: false,

  init() {
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    this.master = this.ctx.createGain();
    this.master.gain.value = 0;
    this.master.connect(this.ctx.destination);
    this.master.gain.linearRampToValueAtTime(0.10, this.ctx.currentTime + 4);
    this.buildDrones();
    this.buildPad();
    this.scheduleBells();
  },

  buildDrones() {
    // Sub bass 41.2 Hz (E1) — gain 0.5
    // Fundamental 82.4 Hz (E2) — gain 0.3
    // Harmonic 123.6 Hz (B2) — gain 0.2 + LFO 0.06Hz modulace
  },

  buildPad() {
    // Dva lehce rozladěné oscilátory 164.8 Hz + 165.2 Hz (E3)
    // type: triangle, přes lowpass filter 800Hz, gain 0.12 každý
  },

  scheduleBells() {
    // E minor pentatonic: [164.8, 196.0, 220.0, 261.6, 293.7, 329.6, 392.0, 440.0]
    // Náhodný tón každých 3,5–10,5s
    // Envelope: attack 15ms, exponential decay 3s, max gain 0.15
  },

  toggle() {
    this.isMuted = !this.isMuted;
    const targetGain = this.isMuted ? 0 : 0.10;
    this.master.gain.linearRampToValueAtTime(targetGain, this.ctx.currentTime + 0.5);
    return this.isMuted;
  }
};
```

**Mute tlačítko:** position fixed, bottom: 24px, right: 24px, 48×48px, border-radius: 50%.
Ikona: 🔊 / 🔇. aria-label="Ztlumit / zapnout hudbu".

---

## SEO META TAGY — KOPÍRUJ DO <head>

```html
<html lang="cs">
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>MoneyMind: Portfolio Challenge — Finanční simulace | Bohatý táta, Chudý táta</title>
<meta name="description" content="Spravuj 1 200 000 € za 10 kol a zjisti, jestli jsi finančně svobodný. Kupuj nemovitosti, akcie, dluhopisy. Sleduj pasivní příjem v reálném čase. Inspirováno Robertem Kiyosakim.">
<meta name="keywords" content="finanční simulace, pasivní příjem, nemovitosti, akcie, ETF, dluhopisy, finanční svoboda, Kiyosaki, bohatý táta, investice, portfolio, finanční gramotnost">
<meta name="author" content="MoneyMind">
<meta name="robots" content="index, follow">
<meta property="og:type" content="website">
<meta property="og:title" content="MoneyMind: Portfolio Challenge">
<meta property="og:description" content="Zainvestuj 1 200 000 € za 5 let. Sleduj pasivní příjem. Zjisti, zda přežiješ bez práce.">
<meta property="og:locale" content="cs_CZ">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="MoneyMind — Finanční simulace s pasivním příjmem">
<meta name="twitter:description" content="10 kol. 1 200 000 €. Postavíš si finanční svobodu?">
<link rel="canonical" href="https://moneymind.cz">
```

---

## ACCESSIBILITY CHECKLIST

- [ ] Všechna tlačítka a karty: `tabindex="0"`, Enter + Space spouštějí akci
- [ ] Progress bar: `role="progressbar"` + `aria-valuenow` + `aria-valuemin="0"` + `aria-valuemax="10"`
- [ ] Emoji dekorace: `aria-hidden="true"`
- [ ] Peněžní hodnoty: nezlomitelná mezera mezi skupinami číslic (`\u00A0`)
- [ ] Barva nikdy není jediný indikátor — vždy doplnit textem nebo ikonou
- [ ] Kontrast: minimum WCAG AA pro veškerý text na tmavém pozadí
- [ ] Mute button: `aria-label="Ztlumit hudbu"` / `"Zapnout hudbu"` (dynamicky)

---

## FORMÁTOVÁNÍ MĚNY

```javascript
function formatCurrency(amount) {
  return new Intl.NumberFormat('cs-CZ', {
    style: 'currency',
    currency: 'EUR',
    maximumFractionDigits: 0
  }).format(amount);
}
// Výstup: "1 200 000 €"
```

---

## KIYOSAKI CITÁTY — ROTACE V HERNÍ OBRAZOVCE

Zobrazuj jeden citát za kolo, rotuj v pořadí:

1. "Bohatí kupují aktiva. Chudí kupují pasiva."
2. "Váš dům není aktivum — je to pasivum."
3. "Nedělejte práci pro peníze — nechte peníze pracovat pro vás."
4. "Riziko přichází z toho, že nevíte, co děláte."
5. "Největší aktivum, které máte, je vaše mysl."
6. "Chudí pracují pro peníze — bohatí nechávají peníze pracovat za ně."
7. "Finanční svoboda je dostupná těm, kdo se o ni skutečně zajímají."
8. "Nevysněte si velký život — žijte ho."
9. "Čím více dáváš, tím více dostáváš."
10. "Nebojte se prohrát. Prohry jsou součástí procesu úspěchu."

---

## API KLÍČ

Hardcode placeholder: `YOUR_API_KEY_HERE`

Přidej HTML komentář přímo v souboru:
```html
<!-- DŮLEŽITÉ: Nahraď YOUR_API_KEY_HERE svým Anthropic API klíčem z console.anthropic.com -->
```

Klíč dosaď sem (v JS kódu):
```javascript
headers: {
  'Content-Type': 'application/json',
  'x-api-key': 'YOUR_API_KEY_HERE',
  'anthropic-version': '2023-06-01',
  'anthropic-dangerous-direct-browser-access': 'true'
}
```

---

## FALLBACK TEXTY (pokud API selže)

```javascript
const FALLBACK_RESULTS = {
  svoboda: "Dosáhl/a jsi finanční svobody — peníze pracují za tebe i ve spánku. Tvé portfolio generuje více, než potřebuješ k životu. Kiyosaki by byl hrdý: přesně takto vypadá skutečné bohatství. Každé aktivum, které jsi koupil/a, je malý motor vydělávající bez tvé přítomnosti. Teprve teď jsi skutečně svobodný/á — ne od práce, ale pro výběr.",

  nezavislost: "Vybudoval/a jsi solidní finanční základ, který ti dává volnost volby. Tvůj měsíční příjem pokrývá základní životní náklady bez nutnosti zaměstnání. Jeden až dva další dobré investiční kroky a finanční svoboda je na dosah. Pokračuj stejnou cestou — jsi blíž, než si myslíš.",

  hrana: "Jsi na správné cestě, ale portfolio potřebuje posílit. Měsíční příjem pokryje přežití, ale bez velké rezervy na nečekané výdaje. Přidej jedno nebo dvě výnosná aktiva a situace se výrazně změní. Bohatství se buduje postupně — každý správný krok se počítá.",

  nestaci: "Tentokrát to nevyšlo — ale důležité je, co ses naučil/a. Pasiva a jednorázové výdaje spolykaly kapitál, který mohl generovat příjem. Příště začni s příjmovými aktivy a přidávej pasiva až z jejich výnosů. Kiyosaki říká: neděláš chyby — získáváš zkušenosti."
};
```

---

## RESPONZIVNÍ BREAKPOINTY

| Viewport | Market grid | Dashboard |
|----------|-------------|-----------|
| ≥ 768px | 2 × 3 karty | Jednořádkový |
| 480–767px | 2 × 3 karty | Kompaktní |
| < 480px | 1 sloupec | Dvouřádkový |
| Min. podpora | 375px | — |

---

## CO NESMÍ BÝT V KÓDU

- ❌ žádný `npm install` ani package.json
- ❌ žádný import z CDN (kromě Google Fonts @import v CSS)
- ❌ žádné globální proměnné mimo GameState a AudioEngine
- ❌ žádný inline style v HTML — vše přes CSS třídy a variables
- ❌ žádný `alert()` nebo `confirm()` — vše custom UI
- ❌ žádný localStorage (stateless session)
- ❌ hardcoded barvy mimo CSS variables

---

*MoneyMind: Portfolio Challenge — Postaveno na filosofii Roberta Kiyosakiho.*
*"Bohatí kupují aktiva. Chudí kupují pasiva."*