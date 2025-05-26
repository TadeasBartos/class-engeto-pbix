# Power BI projekt - Engeto Akademie

## Autor
- **Jméno:** Tadeáš Bartoš
- **Email:** bartos.tadeas@live.com
- **GitHub:** @TadeasBartos

## Popis
Tento projekt je součástí Engeto akademie.
Cílem je zpracování reportu volebních výsledků tak, aby bylo možné mapovat výsledky v celém území, hledat trendy a hlavní konkurenční strany jak v celém území, tak jednotlivých městských obvodech.

## Vstupní data
Vstupní data tvoří volně dostupná data ze stránky [volby.cz](https://www.volby.cz/).
Jejich scrapování bylo předmětem řešení jednoho z předchozích úkolů - 
[repo](https://github.com/TadeasBartos/class-engeto-python-web-scraper).

Struktura vstupních dat byla následující: 
```
code,location,registered,envelopes,valid,OBČANÉ.CZ,Věci veřejné,Konzervativní strana,Komunistická str.Čech a Moravy,Koruna Česká (monarch.strana),Česká strana národně sociální,Česká str.sociálně demokrat.,Strana Práv Občanů ZEMANOVCI,STOP,TOP 09,EVROPSKÝ STŘED,Křesť.demokr.unie-Čs.str.lid.,Volte Pr.Blok www.cibulka.net,Strana zelených,Suverenita-blok J.Bobošíkové,Humanistická strana,Česká pirátská strana,Dělnic.str.sociální spravedl.,Strana svobodných občanů,Občanská demokratická strana,Klíčové hnutí
500054,Praha 1,24 178,16 869,16 752,37,1710,11,643,39,6,1954,370,5,5263,5,580,67,1265,198,18,120,72,157,4184,48
```

Tyto csv soubory byly načteny přímo do Power BI pomocí Power Query. Pro každý rok bylo provedeno jedno načtení. Tyto načtení jsou následně spojeny vazbou N:1 do tabulky kód-obvod.

Rozdílná situace je v případě roku 2006, kdy neexistovala úvodní stránka, takže obsah byl scrapován upraveným kódem a vznikaly jednotlivé soubory. Z pohledu importu bylo toto zachováno.

Soubor [strany.xlsx](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/data/strany.xlsx) zajišťuje jednotnost pojmenování stran napříč roky. Meziroční párování názvů bylo provedeno manuálně.

### Dopočítané metriky

#### % z platných hlasů
```
% z platných hlasů = 
DIVIDE(
    SUM(master_hlasy[POČET HLASŮ]),
    [Platné hlasy v obvodu]
) * 100
```

#### Platné hlasy v obvodu
```
Platné hlasy v obvodu = 
VAR AktKod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok = SELECTEDVALUE(master_hlasy[ROK])
RETURN
    CALCULATE(
        SUM('master_účast'[PLATNÝCH HLASŮ]),
        'master_účast'[KÓD] = AktKod,
        'master_účast'[ROK]= AktRok
    )
```

#### Pořadí
```
Pořadí = 
VAR AktObvod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok   = SELECTEDVALUE(master_hlasy[ROK])

VAR TabulkaStran =
    FILTER(
        ALL('master_hlasy'[STRANA]),
        NOT ISBLANK(
            CALCULATE(
                SUM(master_hlasy[POČET HLASŮ]),
                'master_hlasy'[KÓD] = AktObvod,
                'master_hlasy'[ROK] = AktRok
            )
        )
    )

RETURN
    RANKX(
        TabulkaStran,
        CALCULATE(
            DIVIDE(
                SUM('master_hlasy'[POČET HLASŮ]),
                CALCULATE(
                    SUM('master_účast'[PLATNÝCH HLASŮ]),
                    'master_účast'[KÓD] = AktObvod,
                    'master_účast'[ROK] = AktRok
                )
            )
        ),
        ,
        DESC,
        DENSE
    )
```

#### Pořadí s tečkou
```
Pořadí s tečkou a celkem = 
VAR AktObvod = SELECTEDVALUE(master_hlasy[KÓD])
VAR AktRok   = SELECTEDVALUE(master_hlasy[ROK])

VAR TabulkaStran =
    FILTER(
        ALL('master_hlasy'[STRANA]),
        NOT ISBLANK(
            CALCULATE(
                SUM(master_hlasy[POČET HLASŮ]),
                'master_hlasy'[KÓD] = AktObvod,
                'master_hlasy'[ROK] = AktRok
            )
        )
    )

VAR Poradi =
    RANKX(
        TabulkaStran,
        CALCULATE(
            DIVIDE(
                SUM('master_hlasy'[POČET HLASŮ]),
                CALCULATE(
                    SUM('master_účast'[PLATNÝCH HLASŮ]),
                    'master_účast'[KÓD] = AktObvod,
                    'master_účast'[ROK] = AktRok
                )
            )
        ),
        ,
        DESC,
        DENSE
    )

VAR PocetStran =
    COUNTROWS(TabulkaStran)

RETURN
    FORMAT(Poradi, "0") & ". z " & FORMAT(PocetStran, "0")
```

#### Volební účast
```
[[Volební účast [%]]] = 
CALCULATE(
    DIVIDE(
        SUM('master_účast'[ODEVZDANÝCH HLASŮ]),
        SUM('master_účast'[REGISTROVANÝCH VOLIČŮ]),
        0
    ) * 100,
    TREATAS(
        VALUES('master_hlasy'[ROK]),
        'master_účast'[ROK]
    ),
    TREATAS(
        VALUES('master_hlasy'[KÓD]),
        'master_účast'[KÓD]
    )
)
```

### Dopočítaná tabulky

#### Vítězné strany
Byla záměrně připravena pro vizuály na druhé stránce.
```
vitezne_strany = 
VAR TabulkaZaklad =
    SUMMARIZE(
        master_hlasy,
        master_hlasy[ROK],
        master_hlasy[KÓD],
        master_hlasy[NÁZEV OBVODU]
    )

RETURN
SELECTCOLUMNS(
    ADDCOLUMNS(
        TabulkaZaklad,
        "ViteznaStrana",
            CALCULATE(
                SELECTCOLUMNS(
                    TOPN(
                        1,
                        FILTER(
                            master_hlasy,
                            master_hlasy[ROK] = [ROK] &&
                            master_hlasy[KÓD] = [KÓD]
                        ),
                        master_hlasy[POČET HLASŮ], DESC,
                        master_hlasy[STRANA], ASC
                    ),
                    "Strana", master_hlasy[STRANA]
                )
            ),
        "PocetHlasu",
            CALCULATE(
                MAXX(
                    TOPN(
                        1,
                        FILTER(
                            master_hlasy,
                            master_hlasy[ROK] = [ROK] &&
                            master_hlasy[KÓD] = [KÓD]
                        ),
                        master_hlasy[POČET HLASŮ], DESC
                    ),
                    master_hlasy[POČET HLASŮ]
                )
            )
    ),
    "ROK", [ROK],
    "KÓD", [KÓD],
    "NÁZEV OBVODU", [NÁZEV OBVODU],
    "VÍTĚZNÁ STRANA", [ViteznaStrana],
    "POČET HLASŮ", [PocetHlasu]
)
```

## Jednotlivé stránky reportu

### Úvodní strana
Úvodní strana reportu slouží pouze pro základní navigaci a seznámení uživatele s obsahem. 
Uživatel má možnost provádět navigaci reportem výběrem šipky, každá stránka reportu pak nabízí tlačítko zpět na hlavní stránku a zpět nebo dopředu o jednu stranu.

![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_base.png)

### První strana - Meziroční skladba hlasů
UŽIVATEL ZADÁVÁ: název obvodu.

VÝSTUP: 
- 100% skládaný sloupcový graf pro každý rok,
- pásový graf změny popularity 10ti nejpopulárnějších stran za celý rozsah období reportu,
- pět karet, kdy každá ukazuje vítěznou stranu a počet hlasů ve vybraném obvodu v daném roce.

KOMENTÁŘ: 
První strana slouží pro úvodní orientaci o situaci na území - uživatel získává základní vhled, které strany se drží na volitelných příčkách v průběhu let a k jaké dochází změně meziročně.
Pásový graf ve spodní části velmi dobře ukazuje, jaké strany se ve sledovaném období drží v popředí a jak upadá nebo stoupá jejich popularita mezi voliči.

![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_1.png)

### Druhá strana - Skladba volebních hlasů
UŽIVATEL ZADÁVÁ: rok, název obvodu.

VÝSTUP: 
- skupinový sloupcový graf pro konkrétní rok, 
- bodový graf zobrazující závislost procenta platných hlasů a volební účasti v daném roce,
- spojnicový graf ukazující meziroční vývoj obyvatel a platných hlasů ve volbách,
- karta s číslem celkového počtu hlasů v daném roce a obvodu,
- volební účast v daném roce a obvodu.

KOMENTÁŘ: 
Druhá strana má za úkol předat uživateli detailní vhled do úspěchu volebních stran v daném roce a období v konkrétnám obvodu, vzhledem k velikosti oblasti a počtu platných hlasů.
Tím pádem lze dát do kontextu například ubývající počet hlasů v dané oblasti, který nemusí být zapříčiněn poklesem popularity, ale poklesem volební účasti nebo počtem obyvatel.

![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_2.png)

### Třetí strana - Úspěch strany v obvodech
UŽIVATEL ZADÁVÁ: stranu.

VÝSTUP:
- tabulka pro každý rok, ukazující obvod-platné hlasy v obvodu-získané hlasy stranou-%-pořadí v obvodu,
- karta s názvem nejlidnatějšího obvodu a karta s počtem voličů v daném obvodu,
- spojnicový graf ukazující vztah počtu voličů meziročně.

KOMENTÁŘ: 
Třetí strana dává uživateli nejkomplexnější vhled do úspěchů vybrané strany napříč obvody. Uživatel si skrze šest záložek může tabulky řadit dle:
1. Počtu hlasů - reflextuje absolutní hodnotu získaných hlasů, nezohledňuje ovšem velikost regionu (lidnatější regiony jsou na předních příčkách).
2. Procent - reflektuje sílu strany v daném obvodu, u malých regionů může být číslo zavádějící a tvořit propastný rozdíl mezi jednotlivými pozicemi.
3. Pořadí - ukazuje regiony, kde se strana umístila na nejlepších a nejhorších příčkách. Slouží pro detailní analýzu, ve kterých obvodech strana uspěla, neuspěla nebo dochází ke zhoršení popularity.

Na ukázce můžeme vidět výběr strany, která v roce 2010, 2013, 2017 získala nejvíce procent v obvodu Královice. Tento obvod má ovšem pouze 164, 171, 194 platných hlasů, takže úspěch v tomto obvodu není pro úspěch strany tak klíčový. 
![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_3.png)

Správnou kombinací filtrů lze tedy určit obvody, kam například více cílit zaměření kampaně a kam ne.

### Čtrvtá strana - Konkurence napříč obvody
UŽIVATEL ZADÁVÁ: název obvodu.

VÝSTUP: 
- pásový graf, který graficky reprezenzuje změnu pořadí stran v daném obvodu.

KOMENTÁŘ: 
Čtvrtá strana dává uživateli vhled, jaké strany se kontinuálně umisťují v horních příčkách a k jaké změně dochází meziročně. Tento vizuál potřebuje další vývoj a vstup lektora.
Smyslem je ukázat sílu v daném obvodu a hlavní konkurenty v meziročním období.

> [!CAUTION]
> Aktuální verze Power BI neumožňuje kontrolu tloušťky pásů. Nutná změna v další verzi reportu.

![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_4.png)

### Pátá strana - Volební účast
UŽIVATEL ZADÁVÁ: rok.

VÝSTUP: 
- sloupcový graf ukazující vztah mezi počtem registrovaných obvodů, odevzdaných hlasů a platných hlasů,
- tabulka s názvem obvodu, počtem voličů, odevzdaných/platných hlasů a volební účast v daném obvodu.

KOMENTÁŘ: 
Pátá, poslední strana dává uživateli obecný vhled nad volební účasti mezi roky a hlavně její výrazný pokles v roce 2013 a zpětný nárust mezi roky 2017 a 2021. 
Průřez roku v horní pravé části ovládá pouze tabulku pod ním. Uživatel v tabulce může sledovat velikost obvodu dle počtu voličů, odevzdaných hlasů nebo platných hlasů a volební účasti.

![ ](https://github.com/TadeasBartos/class-engeto-pbix/blob/main/_pictures/page_5.png)
