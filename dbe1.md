# Design by Examples #1 - Monolog

## Monolog

Az első alany a [Monolog](https://github.com/Seldaek/monolog) lett. Ez egy általános loggoláshoz létrehozott library, a Symfony2 alapértelmezett loggere.

A leírását olvasva megakadt a szeme az alábbi példa kódsoron:

    $log->pushHandler(new StreamHandler('path/to/your.log', Logger::WARNING));

A kérdés, amely felmerült bennem pedig így szólt: miért kell a `Handler`nek tudnia arról, hogy milyen szinten történik a loggolás? 
A `Logger` fenttart egy konstanst - `Logger::WARNING` - amely átkerül a `Handler`hez, majd az megmondja a bejövő log üzenet alapján, hogy tud-e vele mit kezdeni - [`isHandling()`](https://github.com/blerou/monolog/blob/master/src/Monolog/Handler/HandlerInterface.php#L21). Ez így kicsit büdös, hiszen a `Logger` felelőssége kellene legyen, hogy csak azokat a `Handler`eket értesítse ki, akik a megfelelő szinttel tudnak foglalkozni. Célom azt bemutatni, hogyan tudunk kis lépésekben, teszt vezérelten eljutni oda, hogy elérjük ezt az állapotot.

## Minden kezdet nehéz

A lib egy látszólag mindenre kiterjedő test suite-tal érkezik. Azonban hamar kiderült, hogy ez sajnos csak a látszat. A coverage megvan, csak a tesztek metódusokhoz írodtak, nem pedig funkcionalitást tesztelnek. Ez pedig arra utal, hogy a teszteket utólag írták meg - [és valóban](https://github.com/blerou/monolog/commit/ed6b0e32a262b57ccdacf1145c225350f6e68eb9). A TDD másik neve *Test first development* és nem véletlenül. A tesztek nagyban veszítenek az erejükből, ha utólag kerülnek megírásra, mivel így nem vezetik a designt, pusztán csak a kész implementációt tesztelik.

Sajnos ez ahhoz is hozzájárul, hogy nem tudom egyszerűen elkezdeni a munkát egy failing testtel, kénytelen vagyok megírni pár tesztet, ami a funkcionalitást teszteli. Mindenek előtt [nulladik lépés gyanánt bevezettem egy egyszerű setUp() metódust és ide kiemeltem a Logger példányosítást](https://github.com/blerou/monolog/commit/c9685dfdc274e093c1f75154dbde8aeae934e440#diff-0).

A célom eléréséhez a `addInfo()` és `addInfo()`, ..., `addAlert()` - alias `addRecord()` - metódusokon át vezet az út - ezek adják a Logger alap funkcionalitásának egy részét. A problémám már csak az volt, hogy pont ez az alap funkcionalitás nem volt tesztelve. Volt külön egy-egy metódushoz, de nem a "nagy" egészhez. Sebaj, írjunk egyet. 

<iframe src="https://github.com/blerou/monolog/commit/a3a23ae33acf03856ac4aee132263f7060033a4c#diff-0" style="height:540px;" class="github-embed" scrolling="no"></iframe>

Itt jegyezném meg, hogy az eredeti tesztesetek docblockjában szerepel egy *@covers* annotáció. Ez elég elhibázott dolog, hiszen ez is azt próbálja leírni, hogy az adott teszteset mely metódust hivatott tesztelni, holott nem ez a lényeg. Ez is az utólag megírt tesztek ismérve.

## Egy felesleges static története

A következő lépés pusztán egy kis kódtisztítás, [eltűntettem a `getName()` metódust](https://github.com/blerou/monolog/commit/9931ec85fbce1d3903371637f509456623c0a030#diff-0). Az oka pedig: egy kutyaközönséges getter, a rosszabbik fajtából. Semmi szükség rá, hiszen a `Handler`en kívül mindenki másnak irreleváns a "csatorna" neve, a `Handler` pedig megkapja a a log üzenetben.

Következő három lépésben pedig eltávolítottam a `getLevelName()` metódust. Valójában csak a log üzenetben játszik szerepet, ugyanúgy egyedi, mint a logolás szintje. Ezt ez a tudás rész lehet az `addRecord()` szignatúrájának. Emellett kívülről (majdnem) senki sem használja.

A lépések:<br />
Első lépés gyanánt, miután egyetlen teszt sem foglalkozik a log szint nevével, ezért eltávolítottam a tesztelés során a teszt log üzenet nem ellenőrzött részeit. Ez az egyetlen külső hívása a metódusnak.

<iframe src="https://github.com/blerou/monolog/commit/50fa0d27f7c93b58f9c67ff2b59d9934eaaa084d#diff-0" style="height:220px;" class="github-embed" scrolling="no"></iframe>

A 2. lépésben bevezettem a `$levelName` paramétert az `addRecord()` szignatúrájában és ezt szépen végig is vezettem a megfelelő hívásokon.

<iframe src="https://github.com/blerou/monolog/commit/73e7d4d5931c48f0cb0810832c855281467c90eb#diff-0" style="height:1250px;" class="github-embed" scrolling="no"></iframe>

És a 3. lépésben el lehet tüntetni a `getLevelName()` static metódust. Engedjük is el.

<iframe src="https://github.com/blerou/monolog/commit/013f6a79e37a629dd69467531af847ee6302a2f3#diff-0" style="height:610px;" class="github-embed" scrolling="no"></iframe>

## Áladaptáció

A következő lépés kissé irrelevánsnak tűnhet, bár szerintem nagyon is fontos. [Kidobtam a "compat" metódusokat](https://github.com/blerou/monolog/commit/8c68a33a2f92d94e834eb075762713ca627f921c#diff-0). Ezek nyújtottak "kompatibilitási" felületet a ZF és Symfony2 frameworkök irányába.

Ez azonban több problémát is felvet:

 1. kód duplikáció: ugyanaz a metódus kétszer is szerepel, csak más néven.
 2. egy library nem tudhat semmilyen frameworkök létezéséről. Az adaptációs réteget a frameworknek kell megoldani, csak ők tudhatják mire van szükségük, egy lib soha.

(Ez az a változtatás, ami miatt valószínüleg soha sem fogadnának el egy pull requestet.)

## Tesztek

Ezután elérkeztem oda, hogy most már tényleg csak az a kód szerepel a `Logger`ben és a hozzá tartozó tesztekben, amire szükség van. Ígyhát kitisztítottam kicsit a testsuite-ot:

 * [konstruktort nem tesztelünk külön](https://github.com/blerou/monolog/commit/d957927b9c87e15f4abfa6b8ec28bfab22fc960a#diff-0), hiszen minden egyes tesztesetben példányosítuk az objetumot.
 * a korábban megírt teszteset már lefedett egy régebbit, ezért az eltávolítottam. Itt azért érdemes megnézni és elgondolkozni, hogy miért nem a interface-t mockolta ki a teszt, miért inkább egy (csak ebből a célból készült) implementációt. Ez valószínűleg azért van, mert az `AbstractHandler` alapértelmezett paraméterei és viselkedése pont megfelelt a tesztesethez. Csakhogy ezáltal elveszett a tesztből az az információ, hogy itt azért hívódik meg a `handle()` metódus, mert korábban már az `isHandling()` igaz eredményt adott. Ez pedig egy smell, megint csak azt mutatja, hogy a tesztet utólag írták meg.
 
<iframe src="https://github.com/blerou/monolog/commit/e3b23a2465f269f85d46392ad643ec0d6d6fc814#L0L54" style="height:320px;" class="github-embed" scrolling="no"></iframe>

 * valamint szükségem volt arra a tesztesetre is, ami a "nem kezelés" esetét fedte le. Amit ez megvolt el tudtam távolítani a régi tesztesetet, amely hasonló gyengeségeket tudott felmutatni, mint a fenti eset.

<iframe src="https://github.com/blerou/monolog/commit/eb801af195d95e7438a7346405a62ae28035e861#diff-0" style="height:372px;" class="github-embed" scrolling="no"></iframe>

<iframe src="https://github.com/blerou/monolog/commit/a6dd9d22f2d6c8f6049b668b9baa015796242d58#diff-0" style="height:372px;" class="github-embed" scrolling="no"></iframe>

## Puzzle - aki kapja ...

Ezután már csak egy teszteset hiányzott, több beregisztrált `Handler` esetén, hogyan történik meg a hivások sora, ki kapja el, ki adja tovább a lehetőséget.

<iframe src="https://github.com/blerou/monolog/commit/4b68842d6b24a295d52e91f55e1bddfa8feba332#diff-1" style="height:540px;" class="github-embed" scrolling="no"></iframe>

Ez az a pont, ahol elkezdenek beszélni a tesztek. Az eddigi tesztesetek metódusokhoz írodtak, nem funkcionalitáshoz, és ez az eset éppen nem volt lefedve. Ami viszont ebből a tesztesetből tisztán látszik, hogy pontosan fordított fontossági sorrendben kell beregisztrálni a `Handler`eket, különben egészen más hatást érünk el, mint amire számítunk. Ezen a ponton bukik meg számomra handlerek tárolásának stack modelje, mivel ez egyáltalán nem természetes módja az ilyen dolgok intézésének. Nem hiszem, hogy a Symfony2-ben is fordított sorrendben kellene megadni a handlereket. Sokkal inkább arra gyanakszom, hogy ott - jó szokás szerint - [a `GroupHandler`t használják, ami viszont már soros működésű](https://github.com/Seldaek/monolog/blob/master/src/Monolog/Handler/GroupHandler.php).

És következzen magának az `addRecord()`nak az átrendezése. Ez az a szituáció, ahol a tesztek fogják a kezünket és a diff nem túl sokat segit a különbség megjelenítésében, ezért álljon itt a régi, majd az új metódus.

A régi:

<iframe src="https://github.com/blerou/monolog/blob/5267b03b1e4861c4657ede17a88f13ef479db482/src/Monolog/Logger.php#L151" style="height:788px;" class="github-embed" scrolling="no"></iframe>

és az új:

<iframe src="https://github.com/blerou/monolog/blob/4b68842d6b24a295d52e91f55e1bddfa8feba332/src/Monolog/Logger.php#L135" style="height:570px;" class="github-embed" scrolling="no"></iframe>

Azért egy újabb "csúnya" dolgot csak műveltem, kidobtam egy default `Handler`t. Ennek az volt az oka, hogy az ilyenek csak aknák a kódban. Ha erre van szükség, akkor az jelenjen meg expliciten a konfigurációban.

<iframe src="https://github.com/blerou/monolog/commit/4b68842d6b24a295d52e91f55e1bddfa8feba332#L0L143" style="height:120px;" class="github-embed" scrolling="no"></iframe>



### Forrás

[Teljes forráskód](https://github.com/blerou/monolog) és a [commitok listája](https://github.com/blerou/monolog/commits/master).

