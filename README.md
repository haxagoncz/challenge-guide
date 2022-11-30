# Technická specifikace požadavků na úlohu spusitelnou v platformě Haxagon

- [Technická specifikace požadavků na úlohu spusitelnou v platformě Haxagon](#technická-specifikace-požadavků-na-úlohu-spusitelnou-v-platformě-haxagon)
    - [Základní principy fungování platformy](#základní-principy-fungování-platformy)
  - [1. Technická specifikace úloh](#1-technická-specifikace-úloh)
    - [`challenge.yaml`](#challengeyaml)
      - [dynamická vlajka](#dynamická-vlajka)
      - [statická vlajka](#statická-vlajka)
      - [vlajka s možností výběru odpovědí](#vlajka-s-možností-výběru-odpovědí)
      - [automaticky se vyhodnocující vlajka](#automaticky-se-vyhodnocující-vlajka)
    - [`docker-compose.yaml`](#docker-composeyaml)
      - [Limity](#limity)
      - [Signalizace úspěšného nastartování scénáře](#signalizace-úspěšného-nastartování-scénáře)
    - [`DESCRIPTION.md`](#descriptionmd)
      - [Strukutra Markdown](#strukutra-markdown)
      - [Další konvence pro tvorbu zadání](#další-konvence-pro-tvorbu-zadání)


### Základní principy fungování platformy

Systém Haxagon zprostředkovává soutěžícím přístup k soutěžním úlohám. Automatizovaně tyto úlohy spouští a vytváří vzdálené připojení pro soutěžící. Aby úloha byla nasaditelná v systému, je nutné aby splňovala určité technícké specifikace.

## 1. Technická specifikace úloh

Formát úlohy je velice jednoduchý - stačí systému poskytnout následující set souborů v gitovém repozitáři:
- `challenge.yaml` - Soubor ve formátu YAML, který definuje základní parametry úlohy
- `docker-compose.yaml` / `docker-compose.yml` - Definice infrastruktury, která se vytvoří pro každou instanci úlohy
- `DESCRIPTION.md` -Markdown soubor obsahující zadání pro plnitele 
- `HANDBOOK.md` - Markdown soubor obsahující obsah, který slouží jako příručka pro učitele

Tyto soubory jsou dále detailněji rozebírány v následujících sekcích

### `challenge.yaml`
| název parametru | popis parametru | příklad |
| - | - | - |
| title | pojmenování úlohy | Uživatelské oprávnění v linuxu
| description | relativní cesta k Markdown souboru, který obsahuje zadání úlohy | `DESCRIPTION.md` |
| handbook | relativní cesta k Markdown souboru, který obsahuje příručku pro učitele | `HANDBOOK.md` |
| flags | pole objektů definující vlajky, které budou součástí úlohy, platforma rozeznává 5 druhů vlajek, definujíse v systému následovně |  |

společné parametry objektů v poli flags:
| název parametru | popis parametru | příklad |
| - | - | - |
| name | pojmenování vlajky | oprávnění souboru `file-1`
| description | bližší info o úkolu | Změň oprávnění souboru `~/file-1` na 742
| points | bodové ohodnocení vlajky | 20
| identifier | unikátní (v rámci souboru challenge.yaml) identifikátor pro vlajku | `file-perms-check1`
| type | číslo označující druh vlajky | "4"

#### **dynamická vlajka**

Každá instance má unikatně vygenerované vlajky, tak aby se zamezilo podvádění. Jejich unikátnost je zaručena vygenerováním náhodného řetězce, kterým jsou nahrazeny všechny výskyty **placeholderu** v repozitáři scénáře. Tak aby nedošlo k nechtěné záměně, jsou všechny místa určená k nahrazení ohraničena znaky `#@{{` a `}}@#`. Právě **placeholder** slouží autorovi úlohy k označení a odlišení jednotlivých míst. Při spuštění se tedy z `#@{{vlajka1}}@#` stane např. `haxagon{897316929176464ebc9ad085f31e7284}` a v jiné instanci s úplně stejným scénářem zase `haxagon{99bd2e29f6b569bb880f601815cd77ef}`. 

> Pozor! K nahrazení řetězců docházi až v runtime instance. Tzn. Nahrazení probíhá těsně předtím, než systém zavolá `docker compose up`. Je potřeba rozdělit `build` fázi kontejnerů a jejich `runtime` fázi. Například tento kod v souboru Dockerfile některého z kontejnerů ulohy: `RUN echo #@{{vlajka1}}@# > /tmp/test`, nesplní tížené očekávání, protože příkaz RUN v Dockerfile je spouštěn při buildění obrazů kontejnerů. Pro zpřístupnění dynamické vlajky v `runtime` fázi kontejneru, je třeba vytvořit složku, v ní vytvořit soubor s placeholderem a tu složku namountovat pomocí docker compose definice `volumes`

specifika pro objekt vlajky tohoto typu
| název parametru | popis parametru | příklad |
| - | - | - |
| type | 1 | 
| placeholder | zástupný řetězec znaků sloužící pro označení místa, do kterého se vloží unikátní vlajka pro instanci (viz. detailní popis níže) | flag2 
| maximumTries | maximální možný počet pokusů o odpověď | 3

#### **statická vlajka**
Tento druh vlajek má ve všech instancích a pro všechny uživatele stejnou hodnotu

specifika pro objekt vlajky tohoto typu
| název parametru | popis parametru | příklad 
| název parametru | popis parametru | příklad 
| - | - | - |
| type | 2 | 
| answer | odpověď na úkol | flag{1234} 
| maximumTries | maximální možný počet pokusů o odpověď | 3

#### **vlajka s možností výběru odpovědí**

specifika pro objekt vlajky tohoto typu
| název parametru | popis parametru | příklad |
| název parametru | popis parametru | příklad |
| - | - | - |
| type | 3 | 
| maximumTries | maximální možný počet pokusů o odpověď | 3
| options | pole objektů možných odpovědí | 
objekt odpovědi má následující strukturu:
```yaml
- value: "chybná odpověď"
  correct: false
```

#### **automaticky se vyhodnocující vlajka**
Pomocí `docker compose exec` se v definovaném **intervalu** spouští **command** a podle **exitCodu** procesu se určí, zda-li byla vlajka splněna. **container** určuje, ve kterým z možných kontejnerů se proces spustí.

specifika pro objekt vlajky tohoto typu
| název parametru | popis parametru | příklad 
| název parametru | popis parametru | příklad 
| - | - | - 
| type | 4 | 
| command | příkaz, který se spustí pro ověření splnění úkolu | `bash -c '[ "$(cat /tmp/test.txt)" == "ahoj" ]'`
| container | cílový kontejner, ve kterém se příkaz bude spouštět | server
| shell | shell, ve kterém je příkaz spouštěn | sh
| user | uživatel, pod kterým je příkaz spoštěn | root
| internval | interval, ve kterém bude docházet ke spuštění příkazu | 2000
| exitCode | v případě, že příkaz bude ukončet s touto hodnotou exit kodu, bude vlajka splněna | 0


ukázkový soubor `challenge.yaml`
```yaml
title: Ukázková úloha

# relativni cesta k souboru obsahujici zadani ulohy
description: ./DESCRIPTION.md

# relativni cesta k souboru obsahujici ucitelskou prirucku k uloze
handbook: ./HANDBOOK.md

# vlajky, ktere jsou s ulohou spojeny
flags:
  - name: Obsah souboru /tmp/file1
    description: 
    points: 10
    type: "1"
    identifier: "file-content-1"
    placeholder: flag1
    maximumTries: 3
  - name: Heslo uživatele adam
    description: Najdi způsob jak získat heslo uživatele adam v plaintextu
    points: 20
    type: "2"
    identifier: "password-dump1"
    answer: flag{adamisbest}
    maximumTries: 2
  - name: Vyber správnou odpověď
    description: 
    points: 30
    type: "3"
    identifier: "choice-flag1"
    maximumTries: 2
    options:
      - value: správná odpověd
        correct: true
      - value: chybná odpověd
        correct: false
  - name: /tmp/test.txt
    description: Do souboru /tmp/test.txt zapiš text "ahoj"
    points: 30
    type: "4"
    identifier: "file-content-check1"
    command: "`bash -c '[ "$(cat /tmp/test.txt)" == "ahoj" ]'`"
    interval: 1000
    container: "server"
    exitCode: 0
```

### `docker-compose.yaml`

Pomocí tohoto souboru jsme schopni definovat, jaká infrastruktura se pro úlohu spustí. 
Dokumentace formátu je dostupná zde: https://docs.docker.com/compose/

#### Limity

V souboru docker-compose není možné:

- eskalovat práva kontejneru:
  - vytvářet privilegované kontejnery
  - přidávat kontejnerům systémové schopnosti (SYStem capabilities)
- mountovat adresáře a soubory (vše potřebné by do kontejneru mělo být předáno v build fázi)

#### Signalizace úspěšného nastartování scénáře

Je nutné systém informovat o tom, že scénář je spuštěný a vše je připraveno. Tuto informaci je možné systému sdělit tím, že do **stdout** entrypointu/commandu libovolného commandu definovaného v docker-compose souboru vypíšete řetězec "SCENARIO_IS_READY". Ukázkový docker-compose.yml:

```yaml
version: "3"

services:
    webserver:
        image: nonbusybox
        container_name: webserver
        command: sh -c '/setup.sh && echo SCENARIO_IS_READY && sleep infinity'
        ports:
            - "80:80"
```

### `DESCRIPTION.md`

Zadání úlohy by mělo splňovat následující konvence:

#### Strukutra Markdown

Obsah zadání se dělí na **teoretickou** část, kde jsou řešiteli předávány teoretické znalosti bez vazby na obsah úlohy a **zadání**. Tato konzistentní struktura napříč úlohami, kde v první řadě v nejdříve v souboru uvedena sekce `# Teorie` a poté až sekce `# Zadání` zlepšuje orientaci řešitelů a zadavatelů v úlohách.

příklad
```markdown
## Teorie
### Enumerace neznámé sitě
informace o tom, jak funguje průzkum sítě, může obsahovat odkaz na asciinema.org, který bude vyrenderovat

## Zadání
Úvodní text zadání
### Přístup do úlohy
Potřebné informace k připojení se do úlohy. Např. port k SSH službě a přístupové údaje uživatele v systému. Může obsahovat placeholder %%INSTANCE_IP%%, ktery bude nahrazen IP adresou instance ulohy
### Vlajka č. 1: Domovský adresář
info k vlajce č. 1
### Vlajka č. 2
info k vlajce č. 2
```

#### Další konvence pro tvorbu zadání
- části textu obsahující nějakou technickou informaci (např. definici subnet - `192.168.40.0/24`, příkazy `find --name file`, parametr - `--service-scan` atp.) zaobalíme do code highlight bloku pomocí znaku **`**
- konzistence termínů napříč zadáním a příručkou k úloze
