# Flat-drop method validation

Stručný validační report obrazového vyhodnocení ploché kapky pro připravovanou flat-drop tenziometrickou metodu. Produkční program je veden samostatně v repozitáři `drop-analysis`; tento repozitář dokumentuje, **proč byly zvoleny konkrétní algoritmy**, a obsahuje autoritativní [implementační specifikaci pro Codex](implementation-spec.md).

> **Stav:** ověřeno na dry reference a celé sekvenci 23 snímků postupného přilévání vody. Hodnoty v pixelech jsou zatím validační diagnostika obrazového algoritmu, nikoli finální metrologický výsledek povrchového napětí.

## 1. Experimentální záměr

Objem vody se v sérii postupně zvětšuje. Snímky tedy zachycují stav:

```text
ještě nedoplněno
      ↓
bok kapky se blíží vertikální tečně u rim
      ↓
ideální stav
      ↓
již za ideálním stavem
```

Cílem není automaticky vybrat jeden jediný „správný“ snímek. **Každý snímek se má fitnout samostatně** a program má ukázat všechny výsledky a diagnostiku. Operátor si následně sám zvolí, které snímky považuje za vhodné.

Sekvence tak současně umožňuje:

- najít stav nejbližší vertikální tečně;
- ověřit robustnost metody při malém pod-/přeplnění.

To odpovídá strategii draftu článku, kde se má okolí optimálního naplnění vzorkovat více stavy a sledovat stabilita výsledku.

## 2. Dry reference: zdroj pravdy pro geometrii aparatury

Referenční snímek bez kapaliny má určit všechny statické veličiny:

- **horní hranu misky** → fyzická rovina `z = 0`;
- **pravý bok misky** → druhá orientační reference;
- natočení/rektifikaci obrazu;
- statickou ROI pro registraci dalších snímků;
- volitelně měřítko;
- volitelně polohu ručně označeného středu misky jako pomocnou/QC informaci.

![Role dry reference](figures/13-reference-roles.svg)

### Natočení

Horní hrana se fituje jako široká horizontální reference a pravý bok jako široká vertikální reference. Z nich se určí korekce natočení kamery. Pokud jednoduchá rotace nedokáže obě reference současně srovnat v rámci tolerance, program to má hlásit jako QC stav, ne automaticky zavádět projektivní transformaci.

### Rim height

Výšku rim **neodvozovat z jasné spodní hrany na snímku s velkou kapkou**. Osvětlení ji opticky posouvá níže. Výškový počátek musí pocházet z dry reference.

### Měřítko

Vpravo je čtvercový papír s roztečí **5 mm**. Automatická detekce mřížky je vhodná až jako doplněk; první implementace může použít ručně/stored zadané `px/mm`. Fit v pixelech nesmí být kvůli chybějící automatické kalibraci blokován.

### Označený střed

Fixou označený střed misky je vhodný jako pomocný bod nebo QC kontrola. Není považován za přesný metrologický constraint, dokud to nebude výslovně požadováno.

## 3. Registrace liquid frames

I při nehybné kameře ukázala dodaná dávka měřitelný posun mezi dry reference a liquid burstem. Proto se každý snímek nejprve registruje ke statické části reference; liquid-air interface se k registraci nepoužívá.

V prototypové analýze byl nalezen posun přibližně kolem 2 px. To je dostatečné na to, aby nebylo bezpečné prostě kopírovat pixelové souřadnice rim z reference bez registrace.

![Registrace dry reference vůči burstu](figures/09-batch-registration.svg)

## 4. Detekce liquid-air interface: porovnané varianty

Na stejném testovacím snímku byly porovnány čtyři přístupy:

| Metoda | Rθ při `zmax = 100 px` [px] | RMSE [px] | SD Rθ pro `zmax = 70–110 px` [px] | Hodnocení |
|---|---:|---:|---:|---|
| Integer gradient peak | 116.159 | 1.738 | **1.130** | stabilní, ale pouze celé pixely |
| **Quadratic subpixel gradient peak** | **115.902** | **1.695** | 1.204 | **zvolená varianta** |
| Gradient centroid | 115.757 | 1.719 | 1.309 | bez zlepšení |
| Logistic intensity-edge fit | 113.758 | 2.207 | 2.224 | méně stabilní na testovaném snímku |

**Zvolený default:** maximum horizontálního gradientu + tříbodová kvadratická subpixelová interpolace maxima.

![Porovnání metod detekce rozhraní](figures/02-edge-methods.svg)

## 5. Výška `h`

Plateau se vyhodnocuje odděleně od boční křivosti. V široké levé ROI se detekuje horní liquid-air interface a jeho výška se vztáhne k registrované rim plane z dry reference.

V dodané filling sequence rostla prototypově detekovaná obrazová výška přibližně z 238.8 px na 281.9 px, což odpovídá pořadí postupného přilévání a je vhodné jako jednoduchá sekvenční diagnostika.

## 6. Tangent diagnostic: kde je vertikální tečna

Krátký lokální fit u rim lze zapsat například jako

```text
q = c1 z + a z²
```

Při zvolené souřadnicové konvenci odpovídá `c1 = 0` vertikální tečně.

V dodané dávce změnil diagnostický člen znaménko mezi `frame_006` a `frame_007`; prototypová lineární interpolace dává průchod nulou přibližně kolem **11.95 s**.

![Průchod diagnostického sklonu nulou](figures/12-tangent-crossing.svg)

Toto je **pomocná informace pro člověka**, nikoli automatický výběr výsledku. Program má fitnout a zobrazit i okolní snímky před a po nule.

## 7. Curvature fit: volný horizontální intercept

Pro křivost se nemá natvrdo vynucovat absolutní image `x0`. Vhodný lokální model je

```text
x(z) = c0 + c2 z²
Rθ = 1 / (2 |c2|)
```

kde `c0` zůstává volný. Tím se křivost neudělá zbytečně citlivou na subpixelovou chybu absolutní radiální reference.

Obecnější diagnostický model

```text
x(z) = c0 + c1 z + c2 z²
```

lze používat ke kontrole sklonu a k porovnání stavů mimo přesnou vertikální tečnu.

## 8. Automatická délka lokálního fitu

Pevná délka profilu se nedoporučuje. **Na každém snímku samostatně** se má procházet řada `zmax` a porovnávat

```text
x = c0 + c2 z²
x = c0 + c2 z² + c4 z⁴
```

Kandidátní interval je z hlediska vyššího členu přijatelný, když

```text
|c4| < 2 sigma_c4
```

což odpovídá kritériu `|b| < 2σb` v draftu článku.

Program může navrhnout nejlepší stabilní interval, ale musí uložit **celý sweep fitovacích oken**, aby šel výběr zpětně zkontrolovat nebo ručně změnit.

![Významnost členu čtvrtého řádu](figures/04-quartic-criterion.svg)

## 9. Contact position a QC

Z liquid profilu lze extrapolovat místo, kde lokální interface protíná registrovanou rim plane. Je vhodné zobrazit jeho průběh přes celou sérii a upozornit na velké změny.

V aktuální prototypové analýze tvořily snímky 001–022 kompaktní oblast a poslední snímek vykázal výrazný skok. Program má tento stav **označit**, ale nemá bez dalšího tvrdit jeho fyzikální příčinu ani výsledek automaticky odstranit.

![Konzistence extrapolované contact position](figures/11-contact-position.svg)

## 10. Doporučená pipeline

```text
DRY REFERENCE
  top rim + right side
  rotation/rectification
  registration ROI
  optional scale / center hint
        ↓
FOR EACH LIQUID FRAME SEPARATELY
        ↓
register to dry reference
        ↓
plateau fit → h
        ↓
side-interface extraction
  gradient maximum
  + 3-point subpixel interpolation
        ↓
contact/tangent diagnostic
        ↓
adaptive curvature-window sweep
  quadratic vs quadratic + z⁴
        ↓
proposed stable window + Rθ
        ↓
SAVE FULL DIAGNOSTICS + OVERLAY
        ↓
SEQUENCE VIEW
  all frames remain visible
  operator chooses suitable results
```

## 11. Co má Codex implementovat

Za závazné technické zadání považuj **[implementation-spec.md](implementation-spec.md)**. README slouží jako stručné zdůvodnění volby metod a report pro kontrolu.

Numerické podklady:

- [batch validation](results/batch-validation.csv)
- [porovnání edge metod](results/edge-method-comparison.csv)
- [diagnostika fitovacích oken](results/frame046-fit-window-diagnostics.csv)

## 12. Co zůstává konfigurovatelné / otevřené

- registrační tolerance;
- přesné ROI;
- krátký interval pro tangent/contact diagnostiku;
- `zmin`;
- krok `zmax`;
- minimum fitovacích bodů;
- numerická definice stability `Rθ`;
- práh pro warning změny contact position;
- automatická 5mm-grid kalibrace.

Tyto hodnoty se nemají schovat jako nezdokumentované konstanty. Operátor musí mít přístup k bodům, fitům, reziduím a QC diagnostice.

---

### Hlavní závěr

Současná nejlepší varianta je **reference-driven geometrie + per-frame nezávislé fitování + gradientová subpixelová extrakce + adaptivní lokální parabolický fit s kontrolou čtvrtého řádu**. Software nemá rozhodovat za uživatele, který snímek je vědecky použitelný; má dodat všechny výsledky a dostatečnou diagnostiku, aby byl tento výběr transparentní.