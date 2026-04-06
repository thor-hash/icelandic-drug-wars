# Product Requirements Document (PRD): Krónan Brennur

**Version:** 3.5
**Date:** 6. apríl 2026
**Product Name:** Krónan Brennur (Drug Wars: Ísland)
**Undirtitill:** "30 dagar til að borga lánshákarlinn áður en eldgosið eyðileggur allt"

---

## 1. Yfirlit

Íslensk útgáfa af klassíska Drug Wars (1984). Single HTML file, retro terminal-stíll. Allur texti á íslensku. Þurr kaldhæðni frekar en dramatík.

Þú ert eiturlyfjasali með 30 daga til að borga Bjarna Bróður skuldina þína. Kauptu lágt, seldu hátt, ferðastu á milli bæja. Skuldin hækkar á hverjum degi. Lögreglan fylgist með.

**Sigurskilyrði:** `score = cash + banki + inventoryValue - skuld`
Jákvætt score = sigur. Neikvætt = Bjarni sendir menn.

### Core Loop (per turn)

```
1. Sýna stöðu (dagur, staður, cash, skuld, heat, birgðir)
2. Keyra event engine (random event ef á við)
3. Reikna markaðsverð (base × location × event × random)
4. Leyfa actions (kaupa, selja, ferðast, bjarni, banki, stash, svartamarkaður)
5. Uppfæra state (heat decay, vextir, dagaskipti)
6. Færa dag áfram → endurtaka
```

---

## 2. Player State

```js
{
  day: 1,              // 1–30
  location: 'reykjavik',
  cash: 250000,        // á þér
  debt: 650000,        // skuld við Bjarna
  bank: 0,             // geymt í banka (öruggt)
  heat: 0,             // 0–100, hættuþrep
  inventory: {},       // { kok: 0, smakk: 0, ... }
  stash: {},           // geymt í Reykjavík
  capacity: 100,       // trenchcoat rými
  weapon: 'knife',     // none | knife | pistol | rifle
  coatLevel: 0,        // 0–2
}
```

---

## 3. Staðir (7 bæir)

| Bær | Lýsing | Heat Decay | Location Mult | Sérstakt |
|-----|--------|------------|---------------|----------|
| Reykjavík | Höfuðborgin — dýrt, ferðamenn, lögregla | 3/dag | 1.0 | Banki, Stash |
| Vestmannaeyjar | Eyjabær — ferja, oft seinkun | 7/dag | 1.1 | Ferju-seinkun |
| Hveragerði | Jarðhitabærinn — góð deals á gras | 6/dag | 0.85 | Gras afsláttur |
| Keflavík | Flugvöllurinn — innflutningur, hæst verð | 4/dag | 1.25 | Import hub |
| Seyðisfjörður | Einangrað — ódýrast en hættulegt | 8/dag | 0.75 | Snjóflóðahætta |
| Akureyri | Norðurlandshöfuðborg — stöðugur markaður | 6/dag | 0.95 | Banki |
| Ísafjörður | Vestfirðir — dýrt en örugt | 10/dag | 1.15 | Lægst heat |

**Heat Decay:** Hversu mikið heat lækkar á dag í þeim bæ. Ísafjörður er öruggast. Reykjavík er hættulegust.

---

## 4. Efni og verðkerfi

### 4.1 Efni (6 tegundir)

| Efni | Emoji | Base verð | Lágmark | Hámark |
|------|-------|-----------|---------|--------|
| Kók | ❄️ | 15.000 | 8.000 | 25.000 |
| MDMA | 💜 | 13.000 | 7.000 | 20.000 |
| Sýra | 🌈 | 5.000 | 2.500 | 8.000 |
| Amf | ⚡ | 3.000 | 1.500 | 5.000 |
| Gras | 🌿 | 2.000 | 800 | 3.500 |
| Sveppir | 🍄 | 2.800 | 1.200 | 4.500 |

### 4.2 Verðformúla

```
price = basePrice × locationMultiplier × eventModifier × random(0.8, 1.2)
price = clamp(price, min, max)
```

- **locationMultiplier** — sjá töflu í kafla 3
- **eventModifier** — 1.0 default, breytist ef event er virkt (t.d. eldgos = 2.5× á Kók/MDMA)
- **random** — 0.8 til 1.2, reiknað á hverjum degi

**Hönnunarsjónarmið:** Kók og MDMA eru high-risk/high-reward. Gras og Sveppir eru volume strategy. Keflavík er alltaf dýrt (import). Seyðisfjörður er alltaf ódýrt (einangrun).

---

## 5. Kerfi

### 5.1 Skuld og vextir

Vextir eru compound og stigvaxandi — pressure eykst eftir því sem líður á leikinn.

| Tímabil | Daglegir vextir | Hönnunarástæða |
|---------|----------------|----------------|
| Dagur 1–10 | 8% | Svigrúm til að byggja upp |
| Dagur 11–20 | 12% | Optimization phase |
| Dagur 21–30 | 15% | Survival panic |

```
interestRate = day <= 10 ? 0.08 : day <= 20 ? 0.12 : 0.15
debt += Math.round(debt × interestRate)
```

- Bjarni Bróðir er fictional lánshákarl. Ekki raunveruleg persóna.
- Hægt að borga skuld hvenær sem er (hvaða upphæð).
- Hægt að taka meira lán (allt að 1M í einu).
- Ef skuld > 2M eftir dag 20: Bjarni sendir menn (event).

### 5.2 Heat (0–100)

Heat mælir hversu mikla athygli þú vekur. Áhrifin eru ekki línuleg — þau vaxa quadratískt.

```
riskMultiplier = 1 + (heat / 100) ** 2
```

| Heat | Risk Multiplier | Merking |
|------|----------------|---------|
| 0 | 1.0× | Enginn sér þig |
| 30 | 1.09× | Smá tortryggnir |
| 50 | 1.25× | Augun á þér |
| 70 | 1.49× | Viðvörun |
| 100 | 2.0× | Fullt eftirlit |

**Heat hækkar:**
- Kaup/sala: `+ceil(transaction / 50.000)`
- Vopnakaup: +5
- Stór viðskipti: tvöföld heat ef transaction > 200K

**Heat lækkar:**
- Daglega eftir bæ (sjá heat decay í kafla 3)

**Heat hefur áhrif á:**
- Líkur á löggu razziu
- Líkur á ráni
- Líkur á Bjarni pressure events
- Almennar "slæmar fréttir" events

### 5.3 Banki

- Geyma peninga öruggt — verndað gegn ránum og Bjarna
- Aðgengilegt í Reykjavík og Akureyri
- Leggja inn / taka út — engin þóknun

### 5.4 Stash

- Geyma efni í Reykjavík — tekur ekki pláss í trenchcoat
- Aðeins aðgengilegt í Reykjavík
- Ótakmarkað rými

### 5.5 Vopn

| Vopn | Verð | Vörn | Lýsing |
|------|------|------|--------|
| Hnífur | Ókeypis (byrjun) | 15% | Betra en ekkert |
| Byssa | 80.000 | 40% | Klassík |
| Riffill | 250.000 | 70% | Alvarlegur búnaður |

Vörn = líkur á að komast undan löggu, ráni eða mönnum Bjarna. Ekki combat — defensive role eingöngu.

### 5.6 Trenchcoat

| Level | Nafn | Rými | Verð |
|-------|------|------|------|
| 0 | Basic Trenchcoat | 100 | Ókeypis |
| 1 | Stór Trenchcoat | 150 | 120.000 |
| 2 | Duffel Bag | 200 | 300.000 |

### 5.7 Ferðalög

- Ferð milli bæja kostar 1 dag (2 í snjóstormi)
- Triggar: dagaskipti, nýtt verð, event roll, heat decay, vexti
- Vera kyrr í sama bæ kostar líka 1 dag — en ný verð

---

## 6. Event Engine

### 6.1 Probability model

```
baseEventChance = 0.25 + (day / 30) × 0.15   // 25% dag 1 → 40% dag 30
```

Events verða tíðari eftir því sem líður á leikinn. Hvert event hefur base weight sem margfaldast við context.

### 6.2 Event flokkar

#### Global events
Hafa áhrif á verð eða leikjaumhverfi.

| Event | Base weight | Áhrif |
|-------|------------|-------|
| 🌋 Eldgos | 0.06 | Kók/MDMA verð ×2–3 |
| 💸 Krónan fellur | 0.04 | Allt verð +20% |
| 🧳 Ferðamannasveifla | 0.08 | Kók/Gras hækkar í Rvk/Kef |
| ♨️ Jarðhiti deal | 0.06 (0.10 í Hveragerði) | Gras verð ×0.3–0.6 |
| 📱 Facebook-deal | 0.05 | Random efni á 30–60% afslætti |
| 🛃 Tollurinn herðir | 0.04 | Innflutt efni (Kók, MDMA) +40% í 1 dag |

#### Local events
Tengjast staðsetningu.

| Event | Base weight | Áhrif |
|-------|------------|-------|
| 🌨️ Snjóstormur | 0.05 | Næsta ferð kostar 2 daga |
| ⛴️ Ferju-seinkun | 0.08 (eyjar/austur) | Festist, missir 1 dag |
| 🚔 Lögreglu razzia | 0.06 × riskMult | Missa hluta af vöru |
| 🚨 Handtaka | 0.02 × riskMult | Game over (nema vopn redda) |

#### Personal events
Tengjast player state.

| Event | Base weight | Áhrif |
|-------|------------|-------|
| 🍺 Rán | 0.06 × riskMult | Missa 10–35% af cash |
| 🦈 Bjarni sendir menn | 0–0.15 (skuld/dagur) | Cash + vöru loss |
| 🎁 Finnur vöru | 0.05 | 2–10 einingar af random efni |
| 💀 Slæm sending | 0.04 | Hluti af vöru eyðilegst |
| 🤝 Gamall vinur svíkur | 0.03 | Stash í Rvk minnkar um 20% |
| ⚖️ Lögmaður reddar | 0.03 | Heat minnkar um 30 |
| 💼 Fjárfestir | 0.02 | Býðst að tvöfalda lán — en skuld hækkar 50% |

### 6.3 Weighted logic

- Hátt heat: löggu/rán events fá `× riskMultiplier`
- Há skuld + dagur ≥ 20: Bjarni events fá `× 0.15`
- Staðsetning: ferju-seinkun aðeins í Vestmannaeyjum/Seyðisfirði
- Jarðhiti deal líklegri í Hveragerði
- Ferðamannasveifla líklegri í Rvk/Kef

### 6.4 Event texta-dæmi

```
"🌋 ELDGOS Í REYKJANESI! Innflutningur stöðvast."
"📞 Bjarni: 'Skuldin þín er farin að verða vandræðaleg.'"
"🌨️ Snjóstormur — ferðin tekur 2 daga."
"🤝 Gamall vinur svíkur þig. Stash í Rvk minnkaði."
"⚖️ Lögmaður reddar þér. Heat lækkar."
"🛃 Tollurinn herðir eftirlit — innflutningsverð hækkar."
```

---

## 7. Gameplay Balance

### Phases

| Phase | Dagar | Markmið | Stemmning |
|-------|-------|---------|-----------|
| Early | 1–10 | Exploration — læra verð, finna bestu leiðir | Rólegt, 8% vextir |
| Mid | 11–20 | Optimization — stækkun, borga skuld | Pressure vex, 12% vextir |
| Late | 21–30 | Survival — forðast hrun, klára leik | Panik, 15% vextir |

### Stefnur

- **Volume strategy:** Gras og Sveppir — lítil áhætta, lág verð, mikið magn. Krefst stórs trenchcoat.
- **Risk strategy:** Kók og MDMA — há áhætta, há verð, lítið magn. Meira heat.
- **Blönduð:** Best fyrir flesta — byrja á volume, skipta yfir í risk þegar sterkari.

### Win condition

Target profit window: ~500K–2M ISK hagnaður eftir 30 daga. Leikurinn ætti að vera vinnanlegt en erfitt. Um 30–40% leikmanna ættu að ná sigri.

Skuld dag 30 (ef engin borgun): `650.000 × (1.08^10) × (1.12^10) × (1.15^10) ≈ 29M`
Þess vegna er mikilvægt að byrja að borga snemma.

---

## 8. UI og textatón

### Reglur

- Þurr, íslenskur deadpan húmor
- Stuttar setningar — ekki orðmörg
- Ekki glorification eða edgy slang
- Kaldhæðni frekar en drama
- Terminal-viðmót — allt lítur út eins og gamall tölvuskjár

### Dæmi um UI texta

```
"Dagur 12. Þú ert ennþá á lífi."
"Skuldin er farin að verða vandræðaleg."
"Það eru augun á þér alls staðar."
"Þetta leit betur út í símanum."
"Bjarni sendir kveðju. Ekki þá góðu."
"Heat: 73%. Einhver er að fylgjast með."
"Ferjan er seinkuð. Aftur."
```

---

## 9. Tæknileg útfærsla

### Stack

- Single HTML file — allt inline (CSS + JS)
- Vanilla JS — engin framework, engin npm
- HTML5 + CSS3
- Responsive — mobile-first
- Retro green-on-black terminal, scanline overlay

### Architecture

```
┌─────────────────────────┐
│ UI Layer                │
│  ├── Header (stats)     │
│  ├── Event log          │
│  ├── Market table       │
│  └── Action menu        │
├─────────────────────────┤
│ Game Engine             │
│  ├── State manager      │
│  ├── Event engine       │
│  ├── Market generator   │
│  └── Turn processor     │
├─────────────────────────┤
│ Data Layer              │
│  ├── Game state (RAM)   │
│  └── Save (localStorage)│
└─────────────────────────┘
```

### Fonts

- **VT323** — aðal texti
- **Press Start 2P** — titlar og header

### Save system (v4.0)

- `localStorage.setItem('kronan_save', JSON.stringify(state))`
- `localStorage.setItem('kronan_highscore', score)`
- Auto-save á hverjum degi

---

## 10. Hugmyndir fyrir v4.0

- [ ] Save/load (localStorage)
- [ ] High score leaderboard
- [ ] Fleiri efni (Sveppir, MDMA)
- [ ] Mini-games (hlaupa frá löggu, blanda efni)
- [ ] Sound effects / chiptune
- [ ] Achievement system
- [ ] Difficulty levels (Auðvelt / Venjulegt / Erfitt)
- [ ] News ticker (flavor text á hverjum degi)
- [ ] Seasonal price cycles (sumar/vetur)
