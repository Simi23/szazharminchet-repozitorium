---
title: Oetletek
description: 
published: true
date: 2025-09-25T17:02:23.341Z
tags: 
editor: markdown
dateCreated: 2025-09-25T14:29:49.371Z
---

# Timeline

- 2025\. október vége - november eleje hirdetés
- 2025\. nov. 21. ES jelentkezési határidő

- 2025\. nov. 28. (p) ES 1. forduló + JS jelentkezési határidő

- 2025\. dec. 5. (p) JS 1. forduló + ES 2. forduló tájékoztató

- 2026\. jan. 14. (sze) ES 2. forduló határidő

- 2026\. febr. 21. (szo) Várna u. - JS döntő
- 2026\. febr. 28. (szo) Várna u. - ES középdöntő

- 2026\. márc. 27-29. (p-v?) Szakmasztár - ES döntő

# 5letek

## JS

### Első forduló

**Helyszín:** Online - Teams
**Időpont:** dec. 5.

- ua. szokásos Linux-, Windows-, Cisco kvíz és PT feladat.
<br>
### Döntő

**Helyszín:** Várna utca
**Időpont:** febr. 21.

- Cisco -> CML (Szokásos szint, amik eddig PT-ben voltak, de valós virtualizált környezetben menne már itt, lenne tapasztalat)
- Szerver -> normális feladtot írunk;

## ES

### Első forduló

**Helyszín:** Online - Teams
**Időpont:** nov. 28.

- Marad ugyanúgy
<br>
### Második forduló

**Helyszín:** Online
**Időpont:** dec. 5. - jan. 14.
**Versenyzők száma:** 30-35 fő (lehetőséghez és kapacításhoz mérten)
**Megoldhatóság:** 

#### Feladat:
1. CML és abba integrált virtuális gépek (Windows, Linux)
2. Hálózat: Nem kellene túltolni, routing, alap switching, kis security 
3. Szerverek: Windows-, HA-Linux szerver és Linux client
4. A szolgáltatás listát illetve egy CML logint kapnának, amivel hozzáférnek az adott lab-hoz, az IP címzést, VLANozást security szabályokat stb. hogy mennyire és hogyan oldja meg az már a versenyzőre lenne bízva
5. Szolgáltatások
  - Dual Stack
  - Windows
    - DS
    - DHCP
    - DDNS
  - Linux
    - HA kiépítése
    - Web
    - Slave DNS
6. Dokumentáció - Mindent technikailag képekkel ledokumentál
7. Eszközök:
  - Windows:
    - CML-be qcow2
    - Egy háló kártya
    - Semmi konfig nincs rajta
    - CML-ből kezelhető
  - Linux SRV:
    - ESXI install (hogy csak a megadott isokból tudjon dolgozni, ne tudjon a netről leszedni akármilyen csomagot)
    - Egy háló kártya
    - Preconfigured interface -> SSH-n keresztüli elérés
      - Kiemelni, hogy ne nyúljon hozzá, ha elrontaná, akkor tudunk supportot adni, mondjuk egy nappal azutánra beidőzítenénk egy service restartot, hogy jelezte, (vagy csinálunk egy olyan scriptet, ami óránként bemásolja a konfigot és restartolja a network servicet, ha változott a file - meg ssh-val is ugyanez ( egy kis gányolás sosem árt :) ))
  - Linux CLT:
  	- CML-be telepített qcow-ból
    - vezérelhető CML-ből előre telepítjük a szükséges csomagokat

#### Javítás:
1. 
2. 
3. 
4. 
5. 

<br>

### Középdöntő

**Helyszín:** Várna utca
**Időpont:** febr. 28.
**Versenyzők száma:** 12-18 % 3 == 0 (lehetőséghez és kapacításhoz mérten)
**Megoldhatóság:** 80%

1. Modul idők növelése 2 órára
2. US BILLENTYŰZETKIOSZTÁS alapértelmezetten - Tájékoztatóban megemlíteni előre, hogy ne sokként érje a versenyzőket, vagy kinyomtatni a kiosztást, vagy olyan billentyűzeteket vinni
3. Marking scriptek írása a feladatokhoz - Cisco Python script - Szerverek - Windows Estonia framework újrafelhasználása (ezzel a Linuxot is lehetne javítani, mert SSH-n keresztül volt)
4. Cisco -> CML - Kicsit szint növelés, szokni a nehezebb versenyfeladatok megoldását (nem úgy mint Herningben vagy pont hogy olyan szintű feladatot mint Herningben), és nem feltéltlenül azt, hogy ez egy egyszerű modul, ahol pikk pakk, minden követve mindent hibátlanul meg lehet oldani
5. Szerver -> normális feladtot írunk, a Szakma Sztár szintjére hozzuk a feladtsort, ami belefér az időben
6. Beszélgetés -> második fordulós remekművek belecsempészése (pl. milyen volt az a munka, milyen problémákba ütközött, mi volt az erőssége, hogy tetszett a feladat, milyen megoldási módot követett. - egy rövid bemutatás, úgy hogy ne bemutatásnak érezze), és a szokásos, van 10^3 switched hogyan konfigurálnád

<br>

### Döntő

**Helyszín:** Szakmasztár
**Időpont:** kb. március 27-28
**Versenyzők száma:** 6 (ha lehet)
**Megoldhatóság:** 60% 

1. A feladatok lehetnének, olyan nehézségűek, hogy ne az legyen a cél, hogy hibátlanra meg lehessen oldani, hanem hogy rendesen be kelljen osztani az időt és esetleg mérlegelni kelljen, hogy mi az amit ki kellene hagyni.
2. Olyan feladatokat adjunk, amik nem workaroundokra építenek.
3. Küldjünk ki olyan anyagokat, amik tényleg működnek, és leteszteltük, ne egy random cikket, ami teljesen mást vár, mint amit a versenyen kell konfigurálni
4. Ha esetleg lenne elég erőforrás, akkor szerverse troublshootot is belevenni?


## Külön ötletek

1. A leszerelt szerverpark, aminek a rackje Simihez megy, azokat felhasználni versenyek lebonyolításához és készüléshez
2. Esetleg GetCon-hoz betenni ezt a szerverparkot, ott fusson ha elfér 👉👈
