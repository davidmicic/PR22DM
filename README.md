
# ANALIZA SLOVENSKEGA TRGA NEPREMIČNIN
### Opis problema
Analiziram in primerjam podatke zadnjih 7 let (2016-2022) slovenskega trga nepremičnin. Za letošnje leto podatkov ne bom analiziral, saj jih je premalo, da bi iz njih lahko črpal kakšne bolj konkretne ugotovitve.

### Opis podatkov
Podatke črpam iz ETN (Evidenca trga nepremičnin), ki je javna,večnamenska zbirka podatkov o kupoprdajnih in najemnih pravnih poslih z nepremičninami. Vodi in vzdržuje jo Geodetska uprava Republike Slovenije. Podatke sem dobil na spletni strani e-Geodetski Podatki (dostopen preko OPSI).

Podatki se nahajajo v 3 različnih datotekah:

- posli.csv,
- delistavb.csv,
- zemljisca.csv

Za vsako leto je približno 35000 vrstic podatkov, razdeljenih v 65 stolpcev. Podatki so v različnih oblikah (numerični, kategorični...).

Do zdaj sem v analizi uporabil naslednje knjižnjice:
- pandas,
- numpy,
- matplotlib,
- sklearn,
- seaborn

### Branje in uvoz podatkov

Za branje .csv datotek sem uporabil metodo `read_csv` iz knjižnjice `Pandas`, s katero sem dobil spremenljivko tipa "DataFrame" (je podobne oblike kot npr. Excelov dokument). 

Primer:
```
posli2022 = pd.read_csv('./podatki/ETN_SLO_CSV_A_KUP/ETN_SLO_KUP_2022_20230304/ETN_SLO_KUP_2022_posli_20230304.csv', delimiter=";")
```
Tako sem uvozil vse datoteke za vsa leta.

Kot že omenjeno, se podatki nahajajo v različnih datotekah, zato jih je bilo potrebno združiti v enoten DataFrame, da bi kasneje bilo lažje manipulirati s podatki.

```
skupni2022 = posli2022.merge(how="inner", right=delistavb2022, on="ID Posla")
skupni2022 = skupni2022.merge(how="inner", right=zemljisca2022, on="ID Posla")
```

V zgornjem primeru sem združil posli2022, delistavb2022 in zemljisca2022 v en skupen DataFrame. 

### Čiščenje podatkov

Na tej točki sem lahko začel s čiščenjem podatkov.

Odstranil sem vse stolpce, katerim manjka več kot 20% podatkov.

```
for col in skupni2022.columns:
    if (skupni2022[col].isnull().mean() > 0.2):
        skupni2022.drop(col, axis="columns")
```

V nadaljevanju sem filtriral posle tako, da sem dobil samo zemljišča na katerih je ali bo mogoče graditi.

```
skupni2022[(skupni2022["Vrsta zemljišča"] == 1) | (skupni2022["Vrsta zemljišča"] == 2) | (skupni2022["Vrsta zemljišča"] == 3)]
```
Številke 1,2 in 3 so šifranti.

Cene nepremičninskih poslov je bilo potrebno zapisati z decimalno piko, da bi jih lahko pretvoril v tip `float` iz tipa `string`, saj so privzeto zapisane z decimalno vejico in brez tega to nebi bilo mogoče. 

```
skupni_filtered2021["Pogodbena cena / Odškodnina"] = skupni_filtered2021["Pogodbena cena / Odškodnina"].str.replace(',', '.')
skupni_filtered2021["Pogodbena cena / Odškodnina"] = skupni_filtered2021["Pogodbena cena / Odškodnina"].astype(float)
```

Ker sem želel ugotoviti, kateri podatki poleg velikosti še najbolj vplivajo na ceno, sem moral nekaj kategoričnih podatkov spremeniti v numerične. To sem storil z metodo `One Hot Encode`. 

Iz kategoričnega tipa sem spreminjal stolpec `Dejanska raba dela stavbe`, ki je v osnovi vseboval šifrante, ki so predstavljali tip nepremičnine (npr.: 2 - Stanovanje, 15 - Garaža, 33 - klet...)

```
value_counts = skupni_filtered2017["Dejanska raba dela stavbe"].value_counts()
top_values = value_counts.head(5).index.tolist()
onehot_df = pd.get_dummies(skupni_filtered2017.loc[skupni_filtered2017["Dejanska raba dela stavbe"].isin(value_counts.nlargest(5).index)], columns=["Dejanska raba dela stavbe"], prefix='onehot_')
result = pd.concat([df, onehot_df], axis=1)
```

Primer korelacijske matrike za leto 2017:
```
ID Posla                                                                0.067088
Vrsta kupoprodajnega posla                                              0.124855
Pogodbena cena / Odškodnina                                             1.000000
Vključenost DDV                                                         0.249640
Stopnja DDV                                                             0.219857
Posredovanje nepremičninske agencije                                    0.011204
Tržnost posla                                                           0.209874
Šifra KO_x                                                              0.139878
Številka stavbe                                                         0.197271
Številka dela stavbe                                                    0.073143
Hišna številka                                                         -0.075876
Številka stanovanja ali poslovnega prostora                             0.107073
Vrsta dela stavbe                                                       0.040055
Leto izgradnje dela stavbe                                              0.084462
Stavba je dokončana                                                     0.004280
Gradbena faza                                                          -0.071948
Novogradnja                                                            -0.249767
Nadstropje dela stavbe                                                  0.094646
Število zunanjih parkirnih mest                                         0.102589
Število sob                                                             0.138234
Šifra KO_y                                                              0.139853
Vrsta zemljišča                                                        -0.114465
Površina parcele                                                        0.003743
onehot__1110001 - Stanovanje v samostoječi stavbi z enim stanovanjem   -0.074631
onehot__1251001 - Industrijski del stavbe                               0.491506
onehot__1271202 - Hlev                                                 -0.070953
onehot__1271301 - Prostor za spravilo pridelka                         -0.092438
onehot__1271401 - Drug kmetijski del stavbe                            -0.081125
```
Primer: Iz matrike lahko razberemo, da če gre za `onehot__1251001 - Industrijski del stavbe` tip nepremičnine, to bolj vpliva na ceno, kot če gre za `onehot__1271202 - Hlev`.

### Ugotovitve
Po izrisu nekaj grafov sem lahko potrdil svoja pričakovanja - da je bilo najmanj poslov sklenjenih v letu 2020. Razlog za to je verjetno Covid. Največ jih je bilo sklenjenih v letu 2022.

Za vsako leto sem primerjal število poslov po najbolj aktivnih regijah, kjer me je presentilo dejstvo, da Ljubljana, Maribor in Koper velikokrat niso na prvih mestih po številu sklenjenih poslov.
