# Flat-drop method validation

Stručný validační report obrazového vyhodnocení lokální geometrie ploché kapky pro připravovanou flat-drop tenziometrickou metodu. Samotný produkční program je veden samostatně v repozitáři `drop-analysis`; tento repozitář slouží k doložení volby algoritmů, citlivostních testů a výsledné implementační specifikace.

> **Stav:** metodická demonstrace na aktuálních experimentálních snímcích. Uvedené hodnoty v pixelech nejsou finálním metrologickým výsledkem a zatím nejsou přepočteny na fyzikální jednotky.

## 1. Cíl

Z obrazu ploché kapky je potřeba robustně určit zejména:

- výšku kapky `h`,
- lokální meridionální poloměr křivosti `Rθ`,
- geometrickou referenci okraje misky (`rim`).

Pro lokální fit se používají rektifikované souřadnice

```text
q = x0 - x
z = y0 - y
q = a z²
Rθ = 1 / (2a)
```

kde `(x0, y0)` je geometrický referenční bod rim. Šířka lokálního fitu se nemá volit pevně; testuje se rozšířeným modelem s členem čtvrtého řádu a kritériem `|b| < 2σb`.

## 2. Doporučený měřicí workflow

### Referenční snímek bez kapaliny

První snímek série bude pořízen bez kapaliny při stejné, následně nehybné kameře. Z něj se mají určit všechny statické geometrické veličiny nesouvisející přímo s kapkou:

- poloha a sklon roviny rim,
- radiální poloha okraje,
- referenční bod `(x0, y0)`,
- prostorová kalibrace,
- statická ROI misky pro kontrolu registrace dalších snímků.

**Důležitý experimentální poznatek:** při větší kapce osvětlení opticky zvýrazňuje spodní hranu misky a vytváří dojem, že rim leží níže. Výšková reference proto nesmí být odvozena z této světlé spodní hrany na snímku s kapalinou. `y0` odpovídá skutečné úrovni, kde začíná liquid-air interface.

![Interpretace rim reference](figures/01-rim-reference.jpg)

### Snímky s kapalinou

Pro každý další snímek se odděleně určí:

1. **plateau a výška `h`** v široké oblasti mimo zakřivený okraj,
2. **lokální profil liquid-air interface** v pravém ROI,
3. **`Rθ`** lokálním parabolickým fitem,
4. **validita fitovacího intervalu** pomocí vyššího členu a stability výsledku.

## 3. Porovnání detekce liquid-air interface

Na stejném snímku `frame_046` byly porovnány čtyři způsoby určení horizontální polohy rozhraní. Následující čísla jsou výsledkem stejného parabolického fitu pro `zmax = 100 px`.

| Metoda | Rθ [px] | RMSE [px] | SD Rθ pro zmax 70–110 px [px] | Hodnocení |
|---|---:|---:|---:|---|
| Integer gradient peak | 116.159 | 1.738 | **1.130** | stabilní, ale pouze celočíselná poloha |
| **Quadratic subpixel gradient peak** | **115.902** | **1.695** | 1.204 | **zvolená varianta** |
| Gradient centroid | 115.757 | 1.719 | 1.309 | bez zlepšení proti kvadratické interpolaci |
| Logistic intensity-edge fit | 113.758 | 2.207 | 2.224 | na tomto snímku méně stabilní |

![Porovnání extrahovaných kontur](figures/02-edge-methods.jpg)

![Stabilita metod při změně šířky fitu](figures/03-edge-stability.jpg)

### Volba

Pro implementaci je doporučeno **maximum horizontálního gradientu následované tříbodovou kvadratickou subpixelovou interpolací maxima**. Na testovaném snímku dává nejnižší RMSE z porovnávaných subpixelových variant a stabilitu blízkou celočíselnému gradientovému maximu, přitom neomezuje polohu rozhraní na celé pixely.

Logistický fit jasového přechodu se pro současná data nedoporučuje.

## 4. Automatická volba lokálního fitu

Použití pevně dané délky profilu není vhodné. Pro postupně rostoucí `zmax` se paralelně fitují modely

```text
q = a z²
q = a z² + b z⁴
```

a sleduje se významnost vyššího členu.

![Významnost členu čtvrtého řádu](figures/04-quartic-criterion.jpg)

Pro aktuální `frame_046` vznikl při pracovní referenci souvislý blok splňující `|b| < 2σb` pro:

```text
zmax = 90, 95, 100, 105 px
```

V tomto bloku vyšel:

```text
median Rθ = 118.256 px
mean Rθ   = 118.231 px
SD        = 0.583 px
full range = 1.354 px
relative full range = 1.145 %
```

![Kandidátní blok lokálního fitu](figures/05-fit-window.jpg)

Samotné kritérium `|b| < 2σb` ještě není dostačující pro finální automatickou volbu. Má být kombinováno s explicitním kritériem stability `Rθ` v souvislém bloku přijatelných intervalů. Numerická definice této stability je zatím otevřený bod.

## 5. Citlivost na geometrickou referenci

Nejvýznamnějším praktickým problémem z dosavadních testů není samotná parabola, ale přesná poloha geometrického počátku.

![Citlivost na výšku rim y0](figures/06-y0-sensitivity.jpg)

![Citlivost na radiální referenci x0](figures/07-x0-sensitivity.jpg)

Z toho plyne hlavní návrhové rozhodnutí: **statickou geometrii misky určit z dry reference frame a nepřizpůsobovat ji každému snímku kapky samostatně.**

## 6. Rezidua

Rezidua lokálního parabolického fitu nejsou při libovolném rozšíření intervalu náhodná; při příliš dlouhém intervalu se začíná projevovat systematická odchylka od lokální paraboly. To podporuje použití adaptivní volby fitovacího okna.

![Rezidua parabolického fitu](figures/08-residuals.jpg)

## 7. Současná doporučená pipeline

```text
DRY REFERENCE FRAME
    ↓
static dish geometry
rim line + (x0, y0) + calibration + registration ROI
    ↓
LOCK GEOMETRIC REFERENCE
    ↓
LIQUID FRAME
    ├── plateau detection → h
    └── side-interface extraction
            ↓
       gradient maximum
            ↓
       3-point quadratic subpixel interpolation
            ↓
       expanding local fit windows
            ↓
       q = az² and q = az² + bz⁴
            ↓
       reject |b| ≥ 2σb
            ↓
       select contiguous stable Rθ region
            ↓
       Rθ = 1/(2a)
```

## 8. Co je ověřeno a co ještě ne

**Ověřeno na poskytnutém snímku:**

- praktická extrakce liquid-air interface gradientovými metodami,
- porovnání čtyř variant lokalizace rozhraní,
- citlivost výsledku na šířku lokálního fitu,
- test členu čtvrtého řádu,
- silná citlivost na volbu geometrického počátku.

**Ještě musí být ověřeno na celé experimentální sérii:**

- automatické určení `(x0, y0)` z dry frame,
- přesnost registrace mezi dry frame a liquid frames,
- finální numerické kritérium stability `Rθ`,
- vhodná minimální vzdálenost `zmin` od rim,
- převod px → mm a úplná propagace nejistot,
- opakovatelnost pro více naplnění a více měřicích sérií.

## 9. Data a implementace

- [Numerická diagnostika fitovacích oken](results/frame046-fit-window-diagnostics.csv)
- [Porovnání detekčních metod](results/edge-method-comparison.csv)
- [Implementační specifikace pro `drop-analysis`](implementation-spec.md)

---

### Shrnutí

Dosavadní testy podporují jednoduchou lokální pipeline založenou na gradientové subpixelové detekci rozhraní a parabolickém fitu s automatickou kontrolou vyššího členu. Nejkritičtější částí je přesná a stabilní geometrická reference rim; proto má být oddělena od samotného fitu kapky a určena z referenčního snímku bez kapaliny.