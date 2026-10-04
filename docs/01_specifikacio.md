# System Specification & Requirements Analysis
**Projekt:** Eseményvezérelt NoSQL Kártyamozgás-és Anomáliakövető Rendszer  
**Architektúra:** FastAPI (Backend) | React + TypeScript (Frontend) | MongoDB (Database) | Docker  
**Dokumentum verzió:** 1.0.0  

---

## 1. Rendszeráttekintés és Célkitűzés

A projekt célja egy nagy áteresztőképességű, alacsony latenciájú, eseményvezérelt (event-driven) kártyamozgás-és anomáliakövető szoftverarchitektúra tervezése és megvalósítása. A rendszer feladata a különböző beléptetési és logisztikai zónák közötti entitásmozgások (pl. RFID alapú belépőkártyák, eszköz-tag-ek) valós idejű rögzítése, az eseménysorozatok feldolgozása, valamint a tér- és időbeli kontextusból eredő anomáliák (pl. párhuzamos jelenlét, fizikai képtelenségből fakadó áthaladás) azonnali detektálása.

A backend aszinkron I/O modellre épül (FastAPI/Python ASGI), míg az adattárolási réteg a MongoDB dokumentumorientált adatbázisra támaszkodik, kihasználva annak replikációs (High Availability) és horizontális skálázhatósági (Sharding) képességeit.

---

## 2. Érintettek és Rendszerszereplők (Actors)

A rendszerben három fő actorípust különböztetünk meg:

### 2.1. External System / IoT Gateway (`System Actor`)
* **Leírás:** Automatizált kapuk, szkenner terminálok és IoT szenzorok, amelyek magas frekvenciával küldenek eseményrekordokat a REST API végpontokra.
* **Jellemzők:** Nem emberi interfész; API kulccsal vagy mTLS/JWT kliens-hitelesítéssel azonosított, nagy számú egyidejű kérést indító kliens.

### 2.2. Operátor / Security Officer (`User Actor`)
* **Leírás:** A zónák fizikai biztonságát vagy a logisztikai folyamatokat felügyelő személyzet.
* **Jellemzők:** Valós idejű dashboard-ot használ, értesítéseket kap az anomáliákról, és manuális státuszmódosításokat (pl. kártya ideiglenes zárolása) hajthat végre.

### 2.3. Rendszeradminisztrátor (`User Actor`)
* **Leírás:** A rendszer infrastruktúrájáért, a felhasználók és kártyák törzsadataiért, valamint az aggregált riportok elemzéséért felelős szerepkör.
* **Jellemzők:** Teljes körű CRUD hozzáférés a törzsadatokhoz, konfigurációs jogok a szűrési szabályokhoz és az anomáliadetektálási küszöbértékekhez.

---

## 3. Funkcionális Követelmények (Functional Requirements)

A funkcionális követelményeket a rendszer moduljai szerint csoportosítjuk.

### FR-1: Hitelesítés és Hozzáférés-kezelés (Auth & RBAC)
* **FR-1.1:** A rendszer biztosítson állapotmentes (stateless) JWT (JSON Web Token) alapú hitelesítést.
* **FR-1.2:** Támogassa a szerepkör-alapú hozzáférés-vezérlést (Role-Based Access Control - RBAC) legalább három szinten (`ADMIN`, `OPERATOR`, `IOT_DEVICE`).
* **FR-1.3:** A jelszavak tárolása biztonságos, salted hash eljárással (bcrypt / Argon2) történjen.

### FR-2: Entitás- és Kártyakezelés (Asset Management)
* **FR-2.1:** Kártyák törzsadatainak rögzítése, módosítása, törlése és lekérdezése (CRUD).
* **FR-2.2:** Minden kártya rendelkezzen egyedi azonosítóval (`card_number`), állapottal (`ACTIVE`, `INACTIVE`, `BLOCKED`, `LOST`) és aktuális tartózkodási hellyel.
* **FR-2.3:** A kártya jelenlegi státuszának és helyzetének lekérdezése O(1) vagy $O(\log N)$ időkomplexitással történjen, minimalizálva az adatbázis összekapcsolási (`$lookup`) műveleteit.

### FR-3: Eseményfeldolgozás és Anomáliadetektálás (Event Ingestion & Anomaly Detection)
* **FR-3.1:** A rendszer fogadja be a kártya-áthaladási eseményeket (`source_location`, `target_location`, `timestamp`, `card_id`).
* **FR-3.2:** Az esemény rögzítésekor a backend automatikusan értékelje ki az anomália-feltételeket:
  * **Idő-térbeli ütközés (Passback / Impossible Speed):** Ha a kártya előző és aktuális mozgása közötti idő kevesebb, mint az adott zónák közötti minimális fizikai átjutási idő.
  * **Zóna-inkonzisztencia:** Ha a kártya bemeneti zónája nem egyezik meg a legutóbb eltárolt tartózkodási zónájával.
* **FR-3.3:** Anomália észlelése esetén a rögzített esemény kapjon `anomaly_flag = true` jelölést, és mentésre kerüljenek a detektálás részletei.

### FR-4: Analytics és Aggregált Riportok (Aggregation Engine)
* **FR-4.1:** A rendszer tegye lehetővé az események idősoros aggregációját (pl. órás/napi áthaladási frekvenciák zónánként).
* **FR-4.2:** Biztosítson felületet és API-t a leggyakrabban riasztott kártyák és zónák lekérdezésére MongoDB Aggregation Pipeline segítségével.

---

## 4. Nem Funkcionális Követelmények (Non-Functional Requirements)

### NFR-1: Teljesítmény és Skálázhatóság (Performance & Scalability)
* **NFR-1.1 (Latency):** Az eseménybejegyző REST API végpont átlagos válaszideje normál terhelés mellett ne haladja meg az **50 ms**-ot.
* **NFR-1.2 (Write Throughput):** Az adatmodellnek támogatnia kell a nagyarányú írási műveleteket (Write-heavy workload) anélkül, hogy a kártya-dokumentumok mérete korlátlanul növekedne (Unbounded Array Anti-Pattern elkerülése).
* **NFR-1.3 (Horizontal Scaling):** Az események gyűjteményének (`card_events`) felkészítettnek kell lennie MongoDB Hashed Sharding architektúrára.

### NFR-2: Architektúra és Portabilitás (Architecture & Deployment)
* **NFR-2.1 (Containerization):** A teljes rendszer (FastAPI backend, React frontend, MongoDB Cluster) deklaratív módon, `docker-compose` segítségével indítható és izolálható legyen.
* **NFR-2.2 (High Availability):** Az adattároló rétegnek tesztfázisban támogatnia kell a MongoDB Replica Set felállást a legalább egycsomópontos kiesések áthidalására.

### NFR-3: Karbantarthatóság és Típusbiztonság (Maintainability & Type Safety)
* **NFR-3.1:** A backend rétegben a Pydantic v2 garantálja a bemeneti adatok szigorú séma-validációját.
* **NFR-3.2:** A frontend komponensek TypeScript alapokon készüljenek a típusbiztonság és a fordításidejű hibaszűrés érdekében.

---

## 5. User Story-k és Elfogadási Kritériumok (Acceptance Criteria)

### US-01: JWT Bejelentkezés
* **As a** Felhasználó (Operátor / Admin)
* **I want to** bejelentkezni a felhasználónevemmel és jelszómmal
* **So that** hozzáférhessek a szerepkörömnek megfelelő funkciókhoz.
  * **AC-1.1:** Helyes adatok esetén a válasz `200 OK` és tartalmazza a JWT access token-t.
  * **AC-1.2:** Helytelen jelszó vagy inaktív userek esetén a válasz `401 Unauthorized`.
  * **AC-1.3:** A token lejárati ideje (TTL) konfigurálható.

### US-02: Kártyamozgás rögzítése és automatikus anomália-szűrés
* **As an** IoT Gateway / Operátor
* **I want to** beküldeni egy kártyaáthaladási eseményt a backend API-nak
* **So that** a rendszer rögzítse a mozgást és azonnal ellenőrizze annak érvényességét.
  * **AC-2.1:** A beküldött adatok alapján új dokumentum keletkezik a `card_events` kollekcióban.
  * **AC-2.2:** A `cards` kollekcióban a kártya `current_location` állapota atomi módon frissül az új zónára.
  * **AC-2.3:** Ha az előző mozgás óta eltelt idő $\Delta t < t_{min}$, a rendszer `anomaly_flag = true` értékkel jelöli meg az eseményt.

### US-03: Zónastatisztikák és Aggregáció
* **As an** Adminisztrátor
* **I want to** lekérdezni a zónák terheltségét egy adott időintervallumban
* **So that** azonosítani tudjam a szűk keresztmetszeteket.
  * **AC-3.1:** A lekérdezés a MongoDB Aggregation Pipeline-t használja (`$match`, `$group`, `$sort`).
  * **AC-3.2:** A válaszidő $10^5$ esemény felett sem haladja meg az **500 ms**-ot a megfelelő indexeltségnek köszönhetően.

---

## 6. Szereplők és Funkciók Mátrixa (Traceability Matrix)

| Funkció / Use Case | IoT Gateway | Operátor | Adminisztrátor |
| :--- | :---: | :---: | :---: |
| **JWT Bejelentkezés** |  | X | X |
| **Kártyák CRUD kezelése** | | | X |
| **Események beküldése** | X | X | |
| **Anomália riasztások megtekintése** | | X | X |
| **Aggregált statisztikák & Riportok** | | | X |
| **Cluster & Sharding Monitoring** | | | X |