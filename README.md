# Analýza podvodov zdravotníckych poistných udalostí pomocou metód dátovej analytiky
## Prehľad

Tento repozitár obsahuje implementáciu praktickej časti **bakalárskej práce** vypracovanej na Technickej univerzite v Košiciach, Fakulte elektrotechniky a informatiky, v študijnom programe Hospodárska informatika.

**Autorka:** Alžbeta Jakšová  
**Vedúca práce:** Ing. Anna Biceková, PhD.  
**Školiace pracovisko:** Ústav umelej inteligencie  
**Rok:** 2026

Práca sa zaoberá detekciou podvodov v zdravotnom poistení pomocou metód dátovej analytiky a strojového učenia. Predmetom projektu je návrh a implementácia **hybridného detekčného modelu**, ktorý kombinuje riadené učenie (XGBoost) a neriadené učenie (Isolation Forest) s cieľom identifikovať podozrivých poskytovateľov zdravotnej starostlivosti. Osobitná pozornosť je venovaná riešeniu problému výraznej dátovej nevyváženosti, interpretovateľnosti výsledkov prostredníctvom SHAP analýzy a ekonomickému hodnoteniu efektivity modelu pomocou Lorenzovej krivky a Giniho koeficientu.

---

## Ciele

- Analyzovať problematiku poistných podvodov v zdravotníctve z pohľadu dátovej analytiky a strojového učenia.
- Pripraviť a agregovať viacdimenzionálne dáta o nárokoch, pacientoch a poskytovateľoch do jednotnej analytickej matice na úrovni poskytovateľa.
- Navrhnúť a implementovať príznaky (feature engineering) zachytávajúce typické vzorce podvodného správania.
- Riešiť problém výraznej dátovej nevyváženosti pomocou hybridnej techniky SMOTE-ENN.
- Natrénovať a optimalizovať hybridný model kombinujúci XGBoost a Isolation Forest prostredníctvom Grid Search na validačnej množine.
- Zabezpečiť interpretovateľnosť modelu pomocou SHAP analýzy (globálna aj lokálna úroveň).
- Vyhodnotiť model nielen pomocou klasických klasifikačných metrík, ale aj z hľadiska ekonomickej efektivity auditu (Lorenzova krivka, Giniho koeficient, analýza fraud expozície).
- Produkovať výstupy použiteľné v praxi: rizikové skóre a prehľad najrizikovejších poskytovateľov.

---

## Dataset

Dataset **nie je súčasťou repozitára** z dôvodu jeho veľkosti. Pochádza z verejne dostupného Medicare datasetu na platforme Kaggle:

> **Healthcare Provider Fraud Detection Analysis**  
> [https://www.kaggle.com/datasets/rohitrox/healthcare-provider-fraud-detection-analysis](https://www.kaggle.com/datasets/rohitrox/healthcare-provider-fraud-detection-analysis)

Po stiahnutí je potrebné umiestniť nasledujúce súbory do koreňového adresára projektu:

| Súbor | Popis |
|---|---|
| `Train-1542865627584.csv` | Označenia poskytovateľov zdravotnej starostlivosti — cieľová premenná `PotentialFraud` (Yes / No) |
| `Train_Beneficiarydata-1542865627584.csv` | Demografické a klinické údaje o poistencoch — vek, chronické diagnózy (Alzheimer, diabetes a iné), pohlavie, stav |
| `Train_Inpatientdata-1542865627584.csv` | Nároky hospitalizovaných pacientov (inpatient claims) — dátumy prijatia a prepustenia, kódy diagnóz, výška náhrady, ošetrujúci lekári |
| `Train_Outpatientdata-1542865627584.csv` | Nároky ambulantných pacientov (outpatient claims) — rovnaká štruktúra ako inpatient, bez hospitalizačných dátumov |

Analýza prebieha výhradne na trénovacej časti datasetu. Všetky štyri tabuľky sú v kóde zlúčené do jednej analytickej matice agregovanej na úrovni poskytovateľa.

---

## Obsah repozitára

Repozitár obsahuje hlavný analytický kód s implementáciou celého pipeline a výstupné súbory vygenerované počas analýzy.

| Súbor | Popis |
|---|---|
| `[hlavný skript / notebook]` | Celý analytický pipeline — od prípravy dát po vizualizácie a výstupy |
| `top20_provider_summary.xlsx` | Výstupná tabuľka — Top 20 najrizikovejších poskytovateľov |
| `*.png` | Vizualizácie generované počas analýzy |

---

## Metodika a prístup

Projekt sa riadi metodológiou **CRISP-DM** (Cross Industry Standard Process for Data Mining) a pozostáva z nasledujúcich fáz:

### 1. Príprava a agregácia dát
Štyri vstupné tabuľky sú zlúčené do jednotnej matice na úrovni poskytovateľa. Inpatient a outpatient nároky sú najprv spojené do jednej tabuľky nárokov (s príznakom `IsInpatient`), následne obohatenej o demografické a klinické informácie o poistencoch a o označenie poskytovateľa. Agregáciou na úrovni providera vznikajú príznaky ako celková a priemerná suma nárokov, počet unikátnych pacientov a lekárov, priemerný počet diagnóz na nárok a počet hospitalizovaných prípadov.

### 2. Feature Engineering
Na základe agregovaných dát sú konštruované odvodené príznaky zachytávajúce správanie typické pre podvodných poskytovateľov:

| Príznak | Popis |
|---|---|
| `AvgAmtPerClaim` | Priemerná výška náhrady na jeden nárok |
| `ClaimsPerPatient` | Priemerný počet nárokov na jedného pacienta |
| `Inpatient_Ratio` | Podiel hospitalizovaných nárokov z celkového počtu |
| `HealthRiskScore` | Miera chronickej záťaže pacientov (Alzheimer + diabetes / počet pacientov) |
| `UniquePhysicians` | Počet unikátnych lekárov (ošetrujúci, operujúci, iní) naprieč všetkými nárokmi |

### 3. Rozdelenie dát a vyvažovanie tried (SMOTE-ENN)
Dataset je rozdelený stratifikovaným spôsobom na trénovaciu (70 %), validačnú (10 %) a testovaciu (20 %) množinu. Extrémna triedna nevyváženosť — legitímni poskytovatelia výrazne prevažujú nad podvodnými — je riešená hybridnou technikou **SMOTE-ENN**, ktorá synteticky generuje vzorky menšinovej triedy (SMOTE) a následne odstraňuje šumové a nejednoznačné body z oboch tried (Edited Nearest Neighbours).

### 4. Trénovanie modelov
Paralelne sú trénované dva modely:
- **XGBoost** (riadené učenie) — trénovaný na vyvážených dátach po SMOTE-ENN, 150 stromov, max. hĺbka 5, learning rate 0.05
- **Isolation Forest** (neriadené učenie) — trénovaný na pôvodných trénovacích dátach, skóre normalizované na interval [0, 1]

### 5. Grid Search a hybridná kombinácia
Pomocou Grid Search na validačnej množine sú optimalizované dva parametre hybridného modelu — váha XGBoost skóre (`w_xgb`) a rozhodovací prah (`threshold`) — s cieľom maximalizovať F1 skóre. Výsledné rizikové skóre má tvar:

```
Score = w_xgb × P(XGBoost) + (1 − w_xgb) × Score(IsolationForest)
```

### 6. Vyhodnotenie a interpretovateľnosť
Model je hodnotený pomocou:
- Klasifikačných metrík: Precision, Recall, F1, AUC-ROC, Average Precision (AP)
- **SHAP analýzy** — globálna dôležitosť premenných (bar graf, beeswarm) a lokálne vysvetlenie individuálnych predikcií
- **Kalibračnej analýzy** — súlad predikovaných pravdepodobností s reálnym výskytom podvodov
- **Lorenzovej krivky a Giniho koeficientu** — miera koncentrácie finančnej expozície pri rôznych kapacitách auditu; porovnanie modelu s náhodným výberom a ideálnym scenárom

---

## Použité technológie

| Kategória | Nástroj / Knižnica |
|---|---|
| **Programovací jazyk** | Python |
| **Spracovanie dát** | pandas, NumPy (<2.0.0) |
| **Strojové učenie** | scikit-learn, XGBoost, imbalanced-learn (SMOTE-ENN) |
| **Vizualizácia** | Matplotlib, Seaborn |
| **Metodológia** | CRISP-DM |

---

## Inštalácia a spustenie

### 1. Klonovanie repozitára

```bash
git clone https://github.com/kkuichi/aj529gh.git
cd aj529gh
```

### 2. Inštalácia závislostí

Kód bol vyvíjaný v prostredí **Google Colab / Jupyter Notebook**. Na začiatku notebooku sú uvedené inštalačné príkazy, ktoré je potrebné spustiť pred samotnou analýzou:

```python
!pip install imbalanced-learn
!pip install xgboost imbalanced-learn
!pip install shap
!pip install "numpy<2.0.0"
```

### 3. Príprava datasetu

Stiahnite dataset z Kaggle (odkaz v sekcii Dataset) a umiestnite všetky štyri CSV súbory do koreňového adresára projektu (rovnaká úroveň ako hlavný notebook / skript).

### 4. Spustenie

```bash
jupyter notebook [nazov_notebooku].ipynb
```

Alebo otvorte notebook priamo v Google Colab a nahrajte CSV súbory do pracovného adresára relácie.

---

## Výsledky a výstupy

### Výstupný súbor: `top20_provider_summary.xlsx`

Hlavným praktickým výstupom projektu je tabuľka **Top 20 najrizikovejších poskytovateľov** na testovacej množine, exportovaná do Excelu. Obsahuje nasledujúce stĺpce:

| Stĺpec | Popis |
|---|---|
| `Provider` | Identifikátor poskytovateľa zdravotnej starostlivosti |
| `Score` | Rizikové skóre hybridného modelu v intervale [0, 1] — čím vyššie, tým väčšia podozrivosť |
| `RiskBand` | Kategória rizika: **Vysoké** (≥ 0.7) / **Stredné** (≥ 0.4) / **Nízke** (< 0.4) |
| `PotentialFraud` | Skutočná trieda z datasetu (1 = podvod, 0 = legitímny) — slúži na overenie správnosti |
| `TotalPaid` | Celková fakturovaná suma náhrad daného poskytovateľa |
| `MainSuspiciousFactor` | Hlavný podozrivý faktor identifikovaný SHAP analýzou — príznak s najväčším vplyvom na predikciu, vrátane jeho hodnoty |

Táto tabuľka simuluje výstup, ktorý by v praxi slúžil audítorom poisťovne ako podklad pre prioritizáciu kontroly.

### Vizualizácie

Počas analýzy sú generované nasledujúce grafy (uložené ako PNG súbory):

| Súbor | Obsah |
|---|---|
| `class_distribution_raw.png` | Distribúcia tried (legitímni vs. podvodní poskytovatelia) v surových dátach |
| `smoteenn_balance.png` | Porovnanie rozloženia tried pred a po aplikácii SMOTE-ENN |
| `hybrid_grid_search_f1.png` | Tepelná mapa Grid Search — F1 skóre pre kombinácie váhy `w_xgb` a prahu |
| `hybrid_confusion_matrix.png` | Matica zámen hybridného modelu na testovacej množine |
| `roc_pr_curves.png` | ROC a Precision–Recall krivky — porovnanie čistého XGBoost a hybridného modelu |
| `model_comparison.png` | Súhrnné porovnanie výkonnosti modelov |
| `hybrid_shap_importance_bar.png` | Bar graf globálnej dôležitosti premenných (SHAP) |
| `hybrid_shap_summary_dot.png` | Beeswarm graf — distribúcia SHAP hodnôt pre jednotlivé príznaky |
| `hybrid_calibration_plot.png` | Kalibračná krivka — súlad predikovaných pravdepodobností s realitou |
| `hybrid_fraud_exposure_random_vs_model.png` | Porovnanie zachytenej fraud expozície pri audite 10 % poskytovateľov: model vs. náhodný výber |
| `hybrid_fraud_exposure_curve.png` | Krivka zachytenej fraud expozície v závislosti od kapacity auditu |
| `hybrid_lorenz_curve.png` | Lorenzova krivka koncentrácie finančnej expozície s Giniho koeficientom |
| `hybrid_audit_table.png` | Tabuľka auditno-ekonomického vyhodnotenia pri rôznych kapacitách auditu (5 %, 10 %, 20 %, 30 %) |

Podrobný popis metodiky, experimentov a interpretácie výsledkov je k dispozícii v texte bakalárskej práce.



