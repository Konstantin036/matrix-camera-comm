S obzirom na to da matrica ima **16x16 (256)** dioda, a šifra **100 karaktera** (velika slova i brojevi), ključni problem je što svi podaci ne mogu stati u jedan frejm. Zbog toga moramo uvesti **sekvencijalni, višefrejmski protokol**.


***
### **Postoje 2 nčina tretiranja i obrade lozinke (binarna ON/OFF reprezentacija i binarna RGB reprezentacija - 2 bita po pixel-u), a samim tim i slike tj. frame-a koji detektuje kamera, koji su redom navedeni u nastavku teksta:**

---

### **Rešenje 1: Višefrejmski binarni ON/OFF Protokol za OAK-1**

Cilj: sva obrada se vrši na OAK-1 kameri. `Script` čvor na kameri mora biti "stateful", odnosno mora pamtiti stanje i sakupljati delove poruke pre finalnog dekodiranja.

#### **1. Poboljšanja Protokola i Dizajn Frejma (16x16)**

Poboljšavamo protokol tako da podržava slanje podataka u segmentima. Svaki frejm na 16x16 matrici sada ima definisanu strukturu sa meta-podacima.

* **Kodiranje Karaktera:**
    * Imamo 26 slova + 10 brojeva = 36 jedinstvenih karaktera.
    * Za kodiranje 36 karaktera potrebno nam je **6 bita** po karakteru ($2^5 = 32$, nedovoljno; $2^6 = 64$, dovoljno).
    * Ukupna veličina podataka: 100 karaktera * 6 bita/karakter = **600 bita**.

* **Struktura Frejma:**
    * **Heder (Header) - Prvi red (16 bita):** Prvi red matrice rezervišemo za ključne meta-podatke. 
        * **ID Sesije (8 bita):** Nasumičan broj koji je isti za sve frejmove jedne poruke. Ovo sprečava mešanje podataka ako dve poruke krenu jedna za drugom.
        * **Brojač Sekvence (4 bita):** Koji je ovo frejm po redu (0, 1, 2...). Omogućava do 16 frejmova po poruci.
        * **Ukupan Broj Frejmova (4 bita):** Koliko ukupno frejmova ima u poruci. Kamera odmah zna kada da očekuje kraj.
    * **Payload-Data (Podaci):** Ostatak matrice, isključujući hedere i futere, koristi se za delove podataka.
    * **Futur (Footer) - Poslednji red (16 bita):**
        * **CRC (Cyclic Redundancy Check) (16 bita):** Umesto običnog checksuma, koristimo CRC-16. Ovo je **značajno poboljšanje** jer je daleko otporniji na greške u prenosu, uključujući i višestruke uzastopne greške (burst errors).

* **Kapacitet i Broj Frejmova:**
    * Ukupno dioda: 256.
    * Rezervisano: 16 (heder) + 16 (futur/CRC) = 32 bita.
    * Dostupno za podatke (payload): 256 - 32 = **224 bita po frejmu**.
    * Potrebno frejmova: 600 bita / 224 bita/frejm = 2.68. Dakle, potrebna su **3 frejma** da se prenese cela poruka.

**Vizuelni Prikaz Frejma na 16x16 Matici:**

![Matrica](matrica.png)   

| Simbol | Značenje                     |
|--------|------------------------------|
| ID     | random ID of za svaki frame  |
| CF     | trenutni redni broj frame-a  |
| TF     | ukupan broj frame-ova        |
| D      | podaci                       |
| S      | checksum (CRC), čeksuma      |

---

### **Rešenje 2: Višefrejmski binarni RGB Protokol za OAK-1**
   * **Kodiranje Karaktera:**
    * Imamo 26 slova + 10 brojeva = 36 jedinstvenih karaktera.
    * Za kodiranje 36 karaktera potrebno nam je **6 bita** po karakteru ($2^5 = 32$, nedovoljno; $2^6 = 64$, dovoljno).
    * Ukupna veličina podataka: 100 karaktera * 6 bita/karakter = **600 bita**.
  ### **Kodiranje Pomoću Boja (2 bita po pikselu):**

|  Boja  | Bin. rep. |
|--------|-----------|
| Crvena |    00     |
| Zelena |    01     |      
| Plava  |    10     |
| Bela   |    11     |

 Napomena: Crna boja (ugašen piksel) se može koristiti za popunjavanje (padding) poslednjeg frejma.

 * **Struktura Frejma (Koristi se cela 16x16 matrica):**

    * **Heder (Header) - Prvi red (16 piksela → 32 bita):**
       * **ID Sesije (8 bita):** Nasumičan broj (0-255). Robusnije od 6 bita.
        * **Brojač Sekvence (4 bita):** Koji je ovo frejm po redu (0-15).
        * **Ukupan Broj Frejmova (4 bita):** Koliko frejmova ima u poruci.
        * **CRC-16 za Heder (16 bita):** Izuzetno robustan kontrolni zbir koji štiti samo heder. Ovo omogućava kameri da momentalno odbaci frejm ako je heder oštećen, pre nego što uopšte pokuša da obradi ostatak.

     * **Payload (Podaci) - Redovi 2 do 15 (14 redova = 224 piksela → 448 bita):**
        * **Ovde se nalazi glavni deo poruke.*

     * **Futur (Footer) - Poslednji red (16 piksela → 32 bita):** 
        * **CRC-32 za Ceo Frejm (32 bita):** Standardni i izuzetno jak kontrolni zbir izračunat nad celim frejmom (Heder + Payload). Ovo pruža drugi, sveobuhvatni sloj zaštite od grešaka.

  * **Kapacitet i Broj Frejmova:**
     * **Ukupno dostupnih bitova po frejmu:** 256 piksela * 2 bita/piksel = 512 bita.
     * **Kapacitet za podatke (payload):** 14 redova * 16 piksela/red * 2 bita/piksel = 448 bita po frejmu.
     * **Potrebno frejmova:** 600 bita / 448 bita/frejm = 1.34.
   * **Zaključak: Potrebna su samo 2 frejma!**

---

#### **Flowchart Implementacije za rešenje 1**

**Predajnik (ESP32):**
1.  Uzmi poruku od 100 karaktera.
2.  Pretvori je u niz od 600 bita koristeći 6-bitnu reprezentaciju.
3.  Generiši nasumični 8-bitni ID sesije.
4.  Podeli niz od 600 bita na 3 segmenta (npr. 2x 224 bita i 1x 152 bita).
5.  **Za svaki od 3 segmenta:**
    * Kreiraj frejm: postavi markere, heder (ID sesije, brojač 0/1/2, ukupan broj 3), i payload.
    * Izračunaj CRC-16 nad hederom i payload-om.
    * Postavi CRC u futur.
    * Prikaži kompletan frejm na matrici na `~200ms`.
    * Pošalji sledeći frejm.

**Prijemnik (OAK-1 `Script` čvor):**
Logika unutar `Script` čvora sada mora da upravlja stanjem.

1.  **Inicijalizacija:** Kreiraj globalnu promenljivu (npr. rečnik/dictionary) `sessions = {}` koja će čuvati podatke.
2.  **Detekcija i Dekodiranje Frejma:** Pronađi matricu, ispravi perspektivu i ekstrahuj svih 256 bita kao i pre.
3.  **Validacija Frejma:**
    * Izdvoj heder, payload i primljeni CRC.
    * Izračunaj CRC nad primljenim hederom i payloadom.
    * **Ako se CRC ne poklapa, odbaci frejm.**
4.  **Obrada Sesije:**
    * Pročitaj `session_id`, `frame_index`, i `total_frames` iz hedera.
    * **Ako `session_id` ne postoji u `sessions`:**
        * Kreiraj novi unos: `sessions[session_id] = {'total': total_frames, 'received': 0, 'data': [None]*total_frames}`.
    * **Ako podaci za `frame_index` već postoje, ignoriši (duplikat).**
    * U suprotnom, sačuvaj payload: `sessions[session_id]['data'][frame_index] = payload`.
    * Povećaj brojač primljenih: `sessions[session_id]['received'] += 1`.
5.  **Asambliranje Poruke:**
    * Proveri da li je `sessions[session_id]['received'] == sessions[session_id]['total']`.
    * **Ako DA:**
        * Spoji sve segmente podataka iz `sessions[session_id]['data']`.
        * Pretvori kompletan niz od 600 bita nazad u 100 karaktera.
        * **Pošalji finalnu, kompletnu poruku od 100 karaktera na host računar.**
        * Obriši sesiju: `del sessions[session_id]`.

---

####  Flowchart Implementacije za Rešenje 2

**Predajnik (ESP32):**

1.  Uzmi poruku od 100 karaktera.
2.  Pretvori je u niz od 600 bita koristeći 6-bitnu reprezentaciju.
3.  Generiši nasumični **8-bitni** ID sesije.
4.  Podeli niz od 600 bita na **2 segmenta** (**1x 448 bita** i **1x 152 bita**).
5.  **Za svaki od 2 segmenta:**
    *   Kreiraj **32-bitni heder** (ID sesije, brojač 0/1, ukupan broj 2, i **CRC-16** izračunat samo nad prva tri polja).
    *   Popuni payload od 448 bita. Za poslednji frejm, ostatak od 152 bita se popunjava nulama (pad-ovanje).
    *   Izračunaj **CRC-32** nad *celim frejmom* (Heder + Payload).
    *   Kreiraj futur od 32 bita sa izračunatim CRC-32.
    *   Pretvori finalni niz od 512 bita u **256 piksela koristeći 4 boje** (2 bita po pikselu).
    *   Prikaži kompletan frejm na matrici na `~200ms`.
    *   Pošalji sledeći frejm.

**Prijemnik (OAK-1 `Script` čvor):**

Logika unutar `Script` čvora upravlja stanjem i koristi dvoslojnu validaciju.

1.  **Inicijalizacija:** Kreiraj globalnu promenljivu (npr. rečnik/dictionary) `sessions = {}` koja će čuvati podatke.
2.  **Detekcija i Dekodiranje Frejma:**
    *   Pronađi **spoljne markere** i ispravi perspektivu matrice.
    *   Ekstrahuj 16x16 piksela i dekodiraj boje u niz od **512 bita**.
3.  **Validacija Frejma (Dvoslojna):**
    *   **Prvo, proveri CRC-16 hedera:** Izdvoj prvih 16 bita (heder podaci) i drugih 16 bita (heder CRC). Ako se ne poklapaju, **odbaci frejm i čekaj sledeći**.
    *   **Drugo, proveri CRC-32 celog frejma:** Ako je heder validan, uzmi prvih 480 bita (heder+payload) i uporedi njihov CRC-32 sa poslednjih 32 bita (futur). Ako se ne poklapa, **odbaci frejm**.
4.  **Obrada Sesije:**
    *   Pročitaj `session_id`, `frame_index`, i `total_frames` iz (sada već validiranog) hedera.
    *   **Ako `session_id` ne postoji u `sessions`:**
        *   Kreiraj novi unos: `sessions[session_id] = {'total': total_frames, 'received_count': 0, 'data': [None]*total_frames}`.
    *   **Ako podaci za `frame_index` već postoje, ignoriši (duplikat).**
    *   U suprotnom, sačuvaj payload: `sessions[session_id]['data'][frame_index] = payload_data`.
    *   Povećaj brojač primljenih: `sessions[session_id]['received_count'] += 1`.
5.  **Asambliranje Poruke:**
    *   Proveri da li je `sessions[session_id]['received_count'] == sessions[session_id]['total']`.
    *   **Ako DA:**
        *   Spoji sve segmente podataka iz `sessions[session_id]['data']`.
        *   Pretvori kompletan niz od 600 bita nazad u 100 karaktera.
        *   **Pošalji finalnu, kompletnu poruku od 100 karaktera na host računar.**
        *   Obriši sesiju: `del sessions[session_id]`.

---

### **Zaključna Poboljšanja i Prednosti Rešenja 1**

* **✅ Skalabilnost:** Sistem sada može preneti poruke praktično neograničene dužine, a ne samo 100 karaktera. Potrebno je samo više frejmova.
* **✅ Izuzetna Robusnost:** Korišćenje **CRC-16** umesto običnog checksuma drastično smanjuje šansu da neprimećena greška prođe. **ID Sesije** osigurava integritet cele poruke, čak i ako dođe do prekida i ponovnog slanja.
* **✅ Efikasnost:** Iako se šalje više frejmova, OAK-1 i dalje na host šalje samo jednu, finalnu poruku. Opterećenje host računara ostaje minimalno, a mreža (USB) se ne zagušuje nepotrebnim podacima.
* **✅ Održivost Stanja:** Sva logika za upravljanje stanjem i sklapanje poruke je unutar `Script` čvora na kameri, čineći sistem autonomnim i elegantnim.

---

### Zaključna Poboljšanja i Prednosti Rešenja 2 

*   ✅ **Brzina:** Prenos je duplo brži korišćenjem boja (2 bita po pikselu), što smanjuje broj potrebnih frejmova sa 3 na samo **2**. Time se celokupan prenos poruke ubrzava za 33%.
*   ✅ **Izuzetna Pouzdanost:** Dvoslojna CRC provera je ključno poboljšanje. **CRC-16 za heder** omogućava momentalno odbacivanje oštećenih frejmova, dok **CRC-32 za ceo frejm** pruža industrijski standard zaštite za podatke, čineći sistem izuzetno otpornim na greške.
*   ✅ **Robusnost Detekcije:** Korišćenje **spoljnih markera** oslobađa celu matricu za podatke i čini sistem detekcije imunim na sadržaj koji se prikazuje. Sistem će pouzdano raditi bez obzira na boje na matrici ili na pokretne dekoracije u okolini.
*   ✅ **Efikasnost:** Sva složena obrada (detekcija, dekodiranje boja, dvostruka CRC provera, upravljanje stanjem sesije i sklapanje poruke) vrši se direktno na kameri. Host računar dobija samo finalnu, validiranu poruku, što minimalizuje opterećenje i čini sistem potpuno autonomnim.

---
