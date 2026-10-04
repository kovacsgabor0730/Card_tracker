# Adatbázis Terv (Eseményvezérelt NoSQL Kártyamozgás- és Anomáliakövető Rendszer)

## 1. Rendszeráttekintés és Adatbázis-architektúra

A kártyamozgás- és anomáliakövető rendszer adatbázis-rétege teljes egészében **MongoDB** (dokumentumorientált NoSQL) alapokra épül. A specifikációban megfogalmazott nagy írási terhelhetőség (**Write-heavy workload**) és alacsony válaszidő (**Latency < 50ms**) érdekében az adatmodell szigorúan követi a NoSQL tervezési mintákat.

### 1.1. Tervezési Alapelvek
1. **Unbounded Array Anti-Pattern Elkerülése:** A kártyamozgás-események nem a `cards` dokumentumba beágyazott tömbben tárolódnak (mivel ez az idő előrehaladtával a 16 MB-os dokumentumméret-korlát átlépéséhez és memória-fragmentációhoz vezetne), hanem egy különálló, nagy sebességű `card_events` gyűjteményben (*Event Sourcing pattern*).
2. **Denormalizáció és Gyors Olvasás O(1) / O(log N):** A `cards` gyűjtemény mindig az adott kártya legfrissebb állapotát (`current_location`, `status`, `last_seen_at`) tartalmazza denormalizálva. Ezzel az események kiértékelésekor és beléptetésnél nincs szükség drága `$lookup` (JOIN) műveletekre.
3. **Sharding Felkészítés (Horizontális Skálázás):** A leggyorsabban növekvő gyűjtemény (`card_events`) hashed sharding stratégiára van előkészítve a `card_id` kulcs mentén, garantálva az egyenletes adat- és íráseloszlást a MongoDB cluster csomópontjai között.

---

## 2. MongoDB Adatmodell és JSON Schemák

A rendszer 4 fő gyűjteményre épül:
- `users` — Operátorok és adminisztrátorok fiókjai, RBAC szerepkörök.
- `zones` — Fizikai zónák, helyiségek, kapuk és minimális áthaladási időküszöbök.
- `cards` — Kulcskártyák törzsadatai és pillanatnyi állapota.
- `card_events` — Eseményvezérelt, idősoros mozgási és anomália napló.

---

### 2.1. `users` Gyűjtemény (Felhasználók & RBAC)

A felülethez hozzáférő operátorok és adminisztrátorok adatait tárolja.

```json
{
  "_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c10" },
  "zone_code": "ZONE-A2-SERVER",
  "name": "Szerverszoba - A2 Épület",
  "building": "A épület",
  "floor": 2,
  "security_level": 4,
  "min_transit_times": [
    {
      "from_zone_code": "ZONE-A1-LOBBY",
      "min_seconds": 30
    },
    {
      "from_zone_code": "ZONE-A2-CORRIDOR",
      "min_seconds": 5
    }
  ],
  "created_at": { "$date": "2026-04-01T08:00:00Z" }
}
```

### 2.2. `zones` Gyűjtemény (Zónák és Topológia)
A zónák közötti minimális áthaladási időket (min_transit_time_seconds) is tárolja az anomáliák (Impossible Speed / Passback) kiszűréséhez.

```json
{
  "_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c10" },
  "zone_code": "ZONE-DEV-01",
  "name": "Szoftverfejlesztői Iroda + Szerverszoba",
  "security_level": 3,
  "building": "A épület",
  "floor": 2,
  "requires_approval": true,
  "created_at": { "$date": "2026-04-01T08:00:00Z" }
}
```

### 2.3. `cards` Gyűjtemény (Kártya Törzsadatok & Aktuális Állapot)
Megvalósítja az FR-2.2 és FR-2.3 követelményeket. Az aktuális állapot ($O(1)$ elérés) közvetlenül frissül minden belépéskor.

```json
{
  "_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c03" },
  "card_number": "CARD-88492",
  "rfid_uid": "4A:8B:12:F9",
  "card_type": "NFC_DESFire",
  "status": "assigned", 
  "current_holder_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c01" },
  "allowed_zone_ids": [
    { "$oid": "660d1a2b3c4d5e6f7a8b9c10" }
  ],
  "assignment_history": [
    {
      "assigned_to": { "$oid": "660d1a2b3c4d5e6f7a8b9c01" },
      "issued_by": { "$oid": "660d1a2b3c4d5e6f7a8b9c99" },
      "issued_at": { "$date": "2026-04-02T08:00:00Z" },
      "returned_at": null,
      "note": "Ideiglenes kártya kiadás szerviz idejére"
    }
  ],
  "created_at": { "$date": "2026-04-01T09:00:00Z" },
  "updated_at": { "$date": "2026-04-02T08:00:00Z" }
}
```

### `card_events` Gyűjtemény (Esemény- és Anomálianapló)
Dedikált, nagy sebességű idősoros gyűjtemény a kapuknál/olvasóknál történő lehúzásokról és fizikai átadásokról.

```json
{
  "_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c04" },
  "card_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c03" },
  "rfid_uid": "4A:8B:12:F9",
  "user_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c01" },
  "zone_id": { "$oid": "660d1a2b3c4d5e6f7a8b9c10" },
  "event_type": "access_granted", 
  "reader_id": "READER-A2-01",
  "timestamp": { "$date": "2026-04-02T08:15:30Z" }
}
```

## 3. MongoDB Indexelési Stratégia

A szigorú teljesítménykövetelmények (NFR-1.1) betartásához elengedhetetlen a megfelelő indexeltség.

```javascript
// --- 1. users kollekció ---
db.users.createIndex({ "username": 1 }, { unique: true });
db.users.createIndex({ "email": 1 }, { unique: true });

// --- 2. zones kollekció ---
db.zones.createIndex({ "zone_code": 1 }, { unique: true });

// --- 3. cards kollekció ---
// O(1) szűrés kártyaszám és RFID azonosító alapján
db.cards.createIndex({ "card_number": 1 }, { unique: true });
db.cards.createIndex({ "rfid_uid": 1 }, { unique: true });
db.cards.createIndex({ "status": 1 });
db.cards.createIndex({ "current_location.zone_code": 1 });

// --- 4. card_events kollekció ---
// Összetett indexek az anomália-szűréshez és idősoros lekérdezésekhez
db.card_events.createIndex({ "card_id": 1, "timestamp": -1 });
db.card_events.createIndex({ "anomaly_flag": 1, "timestamp": -1 });
db.card_events.createIndex({ "target_location": 1, "timestamp": -1 });

// Sharding Hashed Index felkészítés
db.card_events.createIndex({ "card_id": "hashed" });
```

## 4. Sharding és Replikációs Stratégia (Production Setup)

### 4.1. Replica Set Konfiguráció (High Availability - NFR-2.2)

A termelési környezetben a MongoDB legalább egy 3 csomópontos Replica Set (rs0) architektúrát használ:

- mongo-primary: Elsődleges író/olvasó csomópont.

- mongo-secondary-1: Másodlagos, szinkronizált olvasási replika.

- mongo-secondary-2: Másodlagos replika / automatikus failover választó (Arbiter/Secondary).

### 4.2. Sharding Konfiguráció (NFR-1.3)

Nagy mennyiségű esemény rögzítésénél a card_events gyűjtemény hálózati skálázása:

```javascript
// Sharding engedélyezése az adatbázison
sh.enableSharding("card_tracker_db");

// Hashed Shard Key beállítása a card_events kollekción
sh.shardCollection("card_tracker_db.card_events", { "card_id": "hashed" });
```

## 5. Aggregációs Lekérdezések (Aggregation Pipelines)

### 5.1. Leggyakrabban Riasztott Kártyák És Zónák (US-03 / FR-4.2)

Az alábbi aggregációs csővezeték kikeresi az elmúlt 24 óra legtöbb anomáliáját generáló kártyákat és a hozzájuk tartozó célzónákat:

```javascript
db.card_events.aggregate([
  {
    $match: {
      anomaly_flag: true,
      timestamp: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
    }
  },
  {
    $group: {
      _id: {
        card_number: "$card_number",
        target_location: "$target_location",
        anomaly_type: "$anomaly_details.type"
      },
      total_anomalies: { $sum: 1 },
      last_detected: { $max: "$timestamp" }
    }
  },
  {
    $sort: { total_anomalies: -1 }
  },
  {
    $limit: 10
  },
  {
    $project: {
      _id: 0,
      card_number: "$_id.card_number",
      target_location: "$_id.target_location",
      anomaly_type: "$_id.anomaly_type",
      total_anomalies: 1,
      last_detected: 1
    }
  }
]);
```

## 6. Atomi Tranzakciókezelés (FastAPI Event Ingestion)

Amikor az IoT Gateway beküld egy eseményt (US-02), a FastAPI az alábbi atomi műveletet hajtja végre:

1. **Kártya ellenőrzése a `cards` gyűjteményből ($O(1)$ lekérdezés).**

2. **Anomália kiszámítása a `current_location.updated_at` és a zóna `min_transit_time_seconds` alapján.**

3. **MongoDB Transaction vagy Atomic Operation keretében:**
    - Új dokumentum beszúrása a `card_events` gyűjteménybe (`insertOne`).
    - A kártya `current_location` állapota frissítése a `cards` gyűjteményben (`updateOne` az `$set` operátorral).