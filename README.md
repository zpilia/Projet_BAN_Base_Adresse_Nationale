# README -- Projet Base Adresse Nationale (BAN)

## 📍 1. Présentation du projet

Ce projet vise à :

-   Importer un fichier réel issu de la **Base Adresse Nationale
    (BAN)**\
-   Analyser sa structure et ses anomalies\
-   Construire un modèle MERISE complet (MCD → MLD → MPD)\
-   Normaliser les données dans un modèle relationnel propre\
-   Construire un pipeline ETL SQL\
-   Produire toutes les **requêtes SQL** imposées par le brief\
-   Créer une **procédure stockée**, des **triggers** et des **index**\
-   Étudier l'impact de la normalisation sur la cohérence et la
    performance
    
------------------------------------------------------------------------

## 📦 2. Installation et import

### 2.1 Prérequis

-   PostgreSQL ≥ 14
-   psql ou DBeaver
-   Fichier BAN départemental (ex : `adresses-59.csv`)

### 2.2 Création de la base & schéma

``` sql
CREATE DATABASE ban;
CREATE SCHEMA ban;
```

### 2.3 Création de la table brute

``` sql
CREATE TABLE ban.ban_raw (
    id TEXT,
    id_fantoir TEXT,
    numero TEXT,
    rep TEXT,
    nom_voie TEXT,
    code_postal TEXT,
    code_insee TEXT,
    nom_commune TEXT,
    code_insee_ancienne_commune TEXT,
    nom_ancienne_commune TEXT,
    x NUMERIC,
    y NUMERIC,
    lon NUMERIC,
    lat NUMERIC,
    type_position TEXT,
    alias TEXT,
    nom_ld TEXT,
    libelle_acheminement TEXT,
    nom_afnor TEXT,
    source_position TEXT,
    source_nom_voie TEXT,
    certification_commune TEXT,
    cad_parcelles TEXT
);
```

### 2.4 Import du CSV

``` sql
\copy ban.ban_raw 
FROM '/path/to/adresses-59.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ';');
```

------------------------------------------------------------------------

## 🧩 3. MERISE --- MCD, MLD, MPD

## 3.1 MCD — Modèle Conceptuel de Données

### ENTITÉS

#### COMMUNE
| Attribut | Description |
|---------|-------------|
| code_insee (PK) | Identifiant unique de la commune |
| nom_commune | Nom officiel |
| libelle_acheminement | Libellé postal |
| code_insee_ancienne_commune (nullable) | Code INSEE avant fusion |
| nom_ancienne_commune (nullable) | Ancien nom de la commune |

#### VOIE
| Attribut | Description |
|---------|-------------|
| id_voie (PK) | Identifiant interne |
| nom_voie | Nom de la voie |
| nom_afnor | Nom normalisé |
| id_fantoir (nullable) | Identifiant FANTOIR |
| source_nom_voie (nullable) | Source de la donnée |

#### ADRESSE
| Attribut | Description |
|---------|-------------|
| id_adresse (PK) | Identifiant interne |
| numero | Numéro dans la voie |
| rep (nullable) | Répétition |
| type_position | Type de position |
| alias (nullable) | Alias |
| nom_ld (nullable) | Lieu-dit |
| source_position | Source de la position |
| certification_commune | Certification |
| cad_parcelles (nullable) | Parcelles cadastrales |

#### POSITION
| Attribut | Description |
|---------|-------------|
| id_position (PK) | Identifiant interne |
| x | Coordonnée X |
| y | Coordonnée Y |
| lon | Longitude |
| lat | Latitude |


### RELATIONS

| Relation            | Description                                  | Cardinalité |
|---------------------|----------------------------------------------|-------------|
| **Commune → Voie**  | Une commune possède plusieurs voies          | **1,n**     |
| **Voie → Adresse**  | Une voie possède plusieurs adresses          | **1,n**     |
| **Adresse → Position** | Une adresse peut avoir plusieurs positions | **1,n**     |

------------------------------------------------------------------------

## 3.2 MLD --- Modèle Logique de Données

### COMMUNE

-   code_insee (PK)
-   nom_commune
-   libelle_acheminement
-   code_insee_ancienne_commune
-   nom_ancienne_commune

### VOIE

-   id_voie (PK)
-   nom_voie
-   nom_afnor
-   id_fantoir
-   source_nom_voie
-   code_insee (FK → COMMUNE)

### ADRESSE

-   id_adresse (PK)
-   numero
-   rep
-   alias
-   nom_ld
-   cad_parcelles
-   type_position
-   source_position
-   certification_commune
-   id_voie (FK → VOIE)

### POSITION

-   id_position (PK)
-   x
-   y
-   lon
-   lat
-   id_adresse (FK → ADRESSE)

### Clés étrangères

| Table    | Colonne FK  | Référence              |
|----------|-------------|------------------------|
| VOIE     | code_insee  | COMMUNE(code_insee)    |
| ADRESSE  | id_voie     | VOIE(id_voie)          |
| POSITION | id_adresse  | ADRESSE(id_adresse)    |

------------------------------------------------------------------------

## 3.3 MPD --- Modèle Physique de Données (SQL)

``` sql
-- TABLE COMMUNE
CREATE TABLE ban.commune (
    code_insee                      TEXT PRIMARY KEY,
    nom_commune                     TEXT NOT NULL,
    libelle_acheminement            TEXT,
    code_insee_ancienne_commune     TEXT,
    nom_ancienne_commune            TEXT
);

-- TABLE VOIE
CREATE TABLE ban.voie (
    id_voie            SERIAL PRIMARY KEY,
    nom_voie           TEXT NOT NULL,
    nom_afnor          TEXT,
    id_fantoir         TEXT,
    source_nom_voie    TEXT,
    code_insee         TEXT NOT NULL,

    CONSTRAINT fk_voie_commune
        FOREIGN KEY (code_insee)
        REFERENCES ban.commune(code_insee)
);

-- TABLE ADRESSE
CREATE TABLE ban.adresse (
    id_adresse              SERIAL PRIMARY KEY,
    numero                  TEXT NOT NULL,
    rep                     TEXT,
    alias                   TEXT,
    nom_ld                  TEXT,
    cad_parcelles           TEXT,
    type_position           TEXT,
    source_position         TEXT,
    certification_commune   TEXT,
    id_voie                 INT NOT NULL,

    CONSTRAINT fk_adresse_voie
        FOREIGN KEY (id_voie)
        REFERENCES ban.voie(id_voie)
);

-- TABLE POSITION
CREATE TABLE ban.position (
    id_position   SERIAL PRIMARY KEY,
    x             NUMERIC,
    y             NUMERIC,
    lon           NUMERIC NOT NULL,
    lat           NUMERIC NOT NULL,
    id_adresse    INT NOT NULL,

    CONSTRAINT fk_position_adresse
        FOREIGN KEY (id_adresse)
        REFERENCES ban.adresse(id_adresse)
);

```
### Index d’optimisation

```sql
CREATE INDEX idx_voie_code_insee
    ON ban.voie(code_insee);

CREATE INDEX idx_adresse_voie
    ON ban.adresse(id_voie);

CREATE INDEX idx_position_adresse
    ON ban.position(id_adresse);

CREATE INDEX idx_commune_nom
    ON ban.commune(nom_commune);

CREATE INDEX idx_voie_nom
    ON ban.voie(nom_voie);

CREATE INDEX idx_adresse_numero
    ON ban.adresse(numero);
```
------------------------------------------------------------------------

## 🛠️ 4. Pipeline SQL complet --- Transformation BAN → Base normalisée

### 4.1 Nettoyage

``` sql
TRUNCATE ban.position RESTART IDENTITY CASCADE;
TRUNCATE ban.adresse RESTART IDENTITY CASCADE;
TRUNCATE ban.voie RESTART IDENTITY CASCADE;
TRUNCATE ban.commune RESTART IDENTITY CASCADE;
```

### 4.2 Insertion COMMUNE

``` sql
INSERT INTO ban.commune (
    code_insee,
    nom_commune,
    libelle_acheminement,
    code_insee_ancienne_commune,
    nom_ancienne_commune
)
SELECT DISTINCT ON (code_insee)
    code_insee,
    nom_commune,
    libelle_acheminement,
    code_insee_ancienne_commune,
    nom_ancienne_commune
FROM ban.ban_raw
WHERE code_insee IS NOT NULL
ORDER BY code_insee, nom_ancienne_commune NULLS FIRST;
```

### 4.3 Insertion VOIE

``` sql
INSERT INTO ban.voie (
    nom_voie,
    nom_afnor,
    id_fantoir,
    source_nom_voie,
    code_insee
)
SELECT DISTINCT ON (code_insee, nom_voie)
    nom_voie,
    nom_afnor,
    id_fantoir,
    source_nom_voie,
    code_insee
FROM ban.ban_raw
WHERE nom_voie IS NOT NULL AND nom_voie <> ''
ORDER BY code_insee, nom_voie;
```

### 4.4 Insertion ADRESSE

``` sql
INSERT INTO ban.adresse (
    numero,
    rep,
    alias,
    nom_ld,
    cad_parcelles,
    type_position,
    source_position,
    certification_commune,
    id_voie
)
SELECT DISTINCT ON (
        r.code_insee,
        r.nom_voie,
        r.numero,
        COALESCE(r.rep, '')
    )
    r.numero,
    r.rep,
    r.alias,
    r.nom_ld,
    r.cad_parcelles,
    r.type_position,
    r.source_position,
    r.certification_commune,
    v.id_voie
FROM ban.ban_raw r
JOIN ban.voie v
      ON v.nom_voie = r.nom_voie
     AND v.code_insee = r.code_insee
WHERE r.numero IS NOT NULL AND r.numero <> ''
ORDER BY
    r.code_insee,
    r.nom_voie,
    r.numero,
    COALESCE(r.rep, '');
```

### 4.5 Insertion POSITION

``` sql
INSERT INTO ban.position (
    x,
    y,
    lon,
    lat,
    id_adresse
)
SELECT
    r.x,
    r.y,
    r.lon,
    r.lat,
    a.id_adresse
FROM ban.ban_raw r
JOIN ban.voie v
      ON v.nom_voie = r.nom_voie
     AND v.code_insee = r.code_insee
JOIN ban.adresse a
      ON a.id_voie = v.id_voie
     AND a.numero = r.numero
     AND COALESCE(a.rep, '') = COALESCE(r.rep, '');
```

------------------------------------------------------------------------

## 📊 5. Requêtes SQL demandées

## 5.1 Requêtes de consultation

### 1. Lister toutes les adresses d'une commune donnée, triées par voie

``` sql
SELECT
    c.nom_commune,
    v.nom_voie,
    a.numero,
    a.rep
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
JOIN ban.commune c ON v.code_insee = c.code_insee
WHERE c.nom_commune = 'Abancourt'
ORDER BY v.nom_voie, a.numero;
```

### 2. Compter le nombre d'adresses par commune

``` sql
SELECT
    c.nom_commune,
    COUNT(a.id_adresse) AS nb_adresses
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
JOIN ban.commune c ON v.code_insee = c.code_insee
GROUP BY c.nom_commune
ORDER BY nb_adresses DESC;
```

### Bonus : par voie

``` sql
SELECT
    c.nom_commune,
    v.nom_voie,
    COUNT(a.id_adresse) AS nb_adresses
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
JOIN ban.commune c ON v.code_insee = c.code_insee
GROUP BY c.nom_commune, v.nom_voie
ORDER BY c.nom_commune, nb_adresses DESC;
```

### 3. Lister toutes les communes distinctes

``` sql
SELECT nom_commune
FROM ban.commune
ORDER BY nom_commune;
```

### 4. Rechercher les adresses contenant un mot-clé dans le nom de voie

``` sql
SELECT
    v.nom_voie,
    a.numero,
    a.rep,
    c.nom_commune
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
JOIN ban.commune c ON v.code_insee = c.code_insee
WHERE v.nom_voie ILIKE '%Boulevard%';
```

### 5. Identifier les adresses où le code postal est vide (dans la table brute)

``` sql
SELECT *
FROM ban.ban_raw
WHERE (code_postal IS NULL OR code_postal = '')
  AND nom_commune IS NOT NULL
  AND nom_commune <> '';
```

------------------------------------------------------------------------

## 5.2 Insertion / Mise à jour / Suppression

### 1. Ajouter une nouvelle adresse

``` sql
INSERT INTO ban.adresse (
    numero, rep, alias, nom_ld, cad_parcelles,
    type_position, source_position, certification_commune,
    id_voie
)
VALUES (
    '10', NULL, NULL, NULL, NULL,
    'entrée', 'manual', '1',
    1234
);
```

### 2. Mettre à jour une voie

``` sql
UPDATE ban.voie
SET nom_voie = 'Nouvelle Rue Exemple'
WHERE id_voie = 1234;
```

### 3. Supprimer les adresses avec numéro manquant

``` sql
DELETE FROM ban.adresse
WHERE numero IS NULL OR numero = '';
```

------------------------------------------------------------------------

## 5.3 Détection de problèmes & qualité

### 1. Doublons exacts

``` sql
SELECT
    v.nom_voie,
    a.numero,
    a.rep,
    COUNT(*) AS nb
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
GROUP BY v.nom_voie, a.numero, a.rep
HAVING COUNT(*) > 1;
```

### 2. Adresses sans coordonnées GPS

``` sql
SELECT a.*
FROM ban.adresse a
LEFT JOIN ban.position p ON p.id_adresse = a.id_adresse
WHERE p.id_position IS NULL;
```

### 3. Codes postaux avec plus de 10 000 adresses

``` sql
SELECT code_postal, COUNT(*) AS nb
FROM ban.ban_raw
GROUP BY code_postal
HAVING COUNT(*) > 10000
ORDER BY nb DESC;
```

------------------------------------------------------------------------

## 5.4 Agrégation & analyse

### 1. Nombre moyen d'adresses par commune

``` sql
SELECT AVG(nb) AS moyenne
FROM (
    SELECT c.code_insee, COUNT(a.id_adresse) AS nb
    FROM ban.adresse a
    JOIN ban.voie v ON a.id_voie = v.id_voie
    JOIN ban.commune c ON v.code_insee = c.code_insee
    GROUP BY c.code_insee
) t;
```

### 2. Top 10 communes

``` sql
SELECT
    c.nom_commune,
    COUNT(a.id_adresse) AS nb
FROM ban.adresse a
JOIN ban.voie v ON a.id_voie = v.id_voie
JOIN ban.commune c ON v.code_insee = c.code_insee
GROUP BY c.nom_commune
ORDER BY nb DESC
LIMIT 10;
```

### 3. Vérification complétude

``` sql
SELECT
    COUNT(*) FILTER (WHERE numero IS NULL OR numero = '') AS numero_manquant,
    COUNT(*) FILTER (WHERE id_voie IS NULL) AS voie_manquante
FROM ban.adresse;
```

------------------------------------------------------------------------

## 5.5 Requêtes avancées

### 1. Procédure stockée UPSERT

``` sql
CREATE OR REPLACE FUNCTION ban.upsert_adresse(
    p_numero TEXT,
    p_rep TEXT,
    p_id_voie INT
) RETURNS VOID AS $$
BEGIN
    INSERT INTO ban.adresse (numero, rep, id_voie)
    VALUES (p_numero, p_rep, p_id_voie)
    ON CONFLICT (numero, rep, id_voie)
    DO UPDATE SET rep = EXCLUDED.rep;
END;
$$ LANGUAGE plpgsql;
```

### Index requis

``` sql
CREATE UNIQUE INDEX idx_adresse_unique
ON ban.adresse (numero, COALESCE(rep, ''), id_voie);
```

### 2. Trigger validation GPS

``` sql
CREATE OR REPLACE FUNCTION ban.check_gps()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.lat NOT BETWEEN -90 AND 90 THEN
        RAISE EXCEPTION 'Latitude invalide';
    END IF;
    IF NEW.lon NOT BETWEEN -180 AND 180 THEN
        RAISE EXCEPTION 'Longitude invalide';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_check_gps
BEFORE INSERT OR UPDATE ON ban.position
FOR EACH ROW EXECUTE FUNCTION ban.check_gps();
```

### 3. Trigger dates

``` sql
ALTER TABLE ban.adresse
ADD COLUMN date_creation TIMESTAMP DEFAULT NOW(),
ADD COLUMN date_modification TIMESTAMP;

CREATE OR REPLACE FUNCTION ban.set_dates()
RETURNS TRIGGER AS $$
BEGIN
    NEW.date_modification = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_set_dates
BEFORE UPDATE ON ban.adresse
FOR EACH ROW
EXECUTE FUNCTION ban.set_dates();
```
