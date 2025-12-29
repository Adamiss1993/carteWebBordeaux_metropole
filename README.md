# carteWebBordeaux_metropole
La question centrale de ce projet est de savoir si la croissance démographique de Bordeaux Métropole s'accompagne d'une offre de services de proximité suffisante. Nous chercherons particulièrement à identifier les disparités d'accès aux pharmacies, écoles maternelles, collèges et aux médecin generalistes en fonction du réseau de transport en commun existant notamment trams et bus .


########Les réquêtes qui m'ont aider à bien construire ma base 


##----Verifier le système de projection des mes tables 
select ST_SRID(geom)
FROM arrets

select ST_SRID(geom)
FROM commune

select ST_SRID(geom)
FROM iris

select ST_SRID(geom)
FROM equipement

select ST_SRID(geom)
FROM lignes
---suppression des clonnes non concerners 
ALTER TABLE carreaux1km
 DROP COLUMN i_est_1km,
 DROP COLUMN fid, 
  DROP COLUMN idcarreaux;
----Ajout des colonnes pour les clés etrangères 
ALTER TABLE arrets ADD COLUMN idcom INT;
ALTER TABLE arrets ADD COLUMN idiris INT;

ALTER TABLE equipement ADD COLUMN idiris INT;
ALTER TABLE equipement ADD COLUMN idcom INT;
------------ Mettre à jour la commune et l'iris pour les arrêts
UPDATE arrets a
SET idcom = c.idcom, idiris = i.idiris
FROM commune c, iris i
WHERE ST_Intersects(a.geom, c.geom)
AND ST_Intersects(a.geom, i.geom);
---pour renommer une colonne 
ALTER TABLE commune
rename column nom_offici to nom_com
---televerser commune idcom dans equipement
UPDATE equipement e
SET idcom = c.idcom
FROM commune c, iris 
WHERE ST_Intersects(e.geom, c.geom)
--Renommer les type d'une colonne
ALTER TABLE commune 
ALTER COLUMN codes_sire TYPE char(9);
ALTER TABLE commune 
ALTER COLUMN nom_com TYPE VARchar(50);
ALTER TABLE commune 
ALTER COLUMN population type integer;

----- superessoion des ppoint de caerrea1km qui ne se trouvent pas dans commune
DELETE FROM carreaux1km c
WHERE NOT EXISTS (
    SELECT 1
    FROM commune com
    WHERE ST_Intersects(c.geom, com.geom)
);

---ajout de colonne 
ALTER TABLE carreaux1 ADD COLUMN idcom INT;
------pour televerser les idcommune de commune vbers  carereau en se basant sur une remation spatiale
UPDATE carreaux1km car
SET idcom = com.idcom
FROM commune com
WHERE ST_Intersects(car.geom, com.geom);
--- decalration de FK idcom dans carreaux1km
ALTER TABLE carreaux1km ALTER COLUMN idcom SET NOT NULL;
ALTER TABLE carreaux1km
ADD CONSTRAINT fk_carreaux1km
FOREIGN KEY (idcom)
REFERENCES commune(idcom);
--- renomer gie en idcarreau dans carreaux1km
ALTER TABLE carreaux1km
RENAME COLUMN gid TO idcarreaux
----ajout de cle primaire de la table carreaux
ALTER TABLE carreaux1km
ADD CONSTRAINT pk_carreaux1km
PRIMARY KEY (idcarreaux);
--- declaration des clés primaires iris
ALTER TABLE iris
ADD CONSTRAINT pk_iris
PRIMARY KEY (idiris);
------ajout des cle primaires idcom 
ALTER TABLE commune
ADD CONSTRAINT pk_commune
PRIMARY KEY (idcom);
---ajout cle primaire equipement idequipe
ALTER TABLE equipement
ADD CONSTRAINT pk_equipement
PRIMARY KEY (idequipe);
--- Déclarfation de FK de equipement
--Vers la table commune
ALTER TABLE equipement ALTER COLUMN idcom SET NOT NULL;
ALTER TABLE equipement
ADD CONSTRAINT fk_equip_com
FOREIGN KEY (idcom)
REFERENCES commune(idcom);
--- vers iris
ALTER TABLE equipement ALTER COLUMN idiris SET NOT NULL;
ALTER TABLE equipement
ADD CONSTRAINT fk_equip_iris
FOREIGN KEY (idcom)
REFERENCES iris(idiris);
---- reverssement de doonnées vers les fk de  equiment
-- FK de comune
UPDATE equipement e
SET idcom = com.idcom
FROM commune com
WHERE ST_Intersects(e.geom, com.geom);
--- FK iris
UPDATE equipement e
SET idiris = i.idiris
FROM iris i
WHERE ST_Intersects(e.geom, i.geom);
-----declareration de cle primaire idarret
ALTER TABLE arrets
ADD CONSTRAINT pk_arret
PRIMARY KEY (idarret);
--televersement de idcom dans arrets
UPDATE arrets a
SET idcom = c.idcom
FROM commune c
WHERE ST_Intersects(a.geom, c.geom);
--- ----- superessoion des ppoint de arret qui ne se trouvent pas dans commune
DELETE FROM arrets a
WHERE NOT EXISTS (
    SELECT 1
    FROM commune com
    WHERE ST_Intersects(a.geom, com.geom)
);
----declaration des clés etrangère arrets
-- vers commune
ALTER TABLE arrets ALTER COLUMN idcom SET NOT NULL;
ALTER TABLE arrets
ADD CONSTRAINT fk_arrets_com
FOREIGN KEY (idcom)
REFERENCES commune(idcom);
----- vers iris
--televersement de idiris dans arret
UPDATE arrets a
SET idiris = i.idiris
FROM iris i
WHERE ST_Intersects(a.geom, i.geom);
--suppression des points hors iris
DELETE FROM arrets a
WHERE NOT EXISTS (
    SELECT 1
    FROM iris i
    WHERE ST_Intersects(a.geom, i.geom)
);
--declarationdes clés etrangères
ALTER TABLE arrets ALTER COLUMN idiris SET NOT NULL;
ALTER TABLE arrets
ADD CONSTRAINT fk_arrets_iris
FOREIGN KEY (idiris)
REFERENCES iris(idiris);
-- declaration de PK de ligne
ALTER TABLE lignes
ADD CONSTRAINT pk_lignes
PRIMARY KEY (idligne)

----declaration des primaire table desservir
CREATE TABLE desservir(
   idarrets INT,
   idligne INT,
   PRIMARY KEY(idarrets, idligne)
);
----declaration des primaire table recouvrir

CREATE TABLE recouvrir(
   idiris INT,
   idcarreaux INT,
   PRIMARY KEY(idiris, idcarreaux)
);
-- FK recouvrir
-- televerssement les deux iris carrau
INSERT INTO recouvrir (idiris, idcarreaux)
SELECT
    i.idiris,
    ca.idcarreaux
FROM iris i
JOIN carreaux1km ca
ON ST_Contains(i.geom, ca.geom);
---FK vers iris
ALTER TABLE recouvrir ALTER COLUMN idiris SET NOT NULL;
ALTER TABLE recouvrir
ADD CONSTRAINT fk_recouvrir_iris
FOREIGN KEY (idiris)
REFERENCES iris(idiris);
--- FK vers carreau
ALTER TABLE recouvrir ALTER COLUMN idcarreaux SET NOT NULL;
ALTER TABLE recouvrir
ADD CONSTRAINT fk_recouvrir_carreau
FOREIGN KEY (idcarreaux)
REFERENCES carreaux1km(idcarreaux);
--- FK desservir
-- televerssement les deux iris carrau
INSERT INTO desservir (idarrets, idligne)
SELECT
    a.idarret,
    l.idligne
FROM arrets a
JOIN lignes l
ON l.idligne = (
    SELECT l2.idligne
    FROM lignes l2
    ORDER BY ST_Distance(a.geom, l2.geom)
    LIMIT 1
);
---FK vers arrets
ALTER TABLE desservir ALTER COLUMN idarrets SET NOT NULL;
ALTER TABLE desservir
ADD CONSTRAINT fk_desservir_arret
FOREIGN KEY (idarrets)
REFERENCES arrets(idarret);
--- FK vers lignes
ALTER TABLE desservir ALTER COLUMN idligne SET NOT NULL;
ALTER TABLE desservir
ADD CONSTRAINT fk_desservir_ligne
FOREIGN KEY (idligne)
REFERENCES lignes(idligne);

---requêtes pour analyse 
--Nombre d'arrêts de transport par zone IRIS
SELECT i.nom_iris, COUNT(a.stop_id) as densite_arrets
FROM iris i
LEFT JOIN arrets a ON ST_Intersects(a.geom, i.geom)
GROUP BY i.nom_iris
ORDER BY densite_arrets DESC;
----Trouver les équipements à moins de 300m d'un arrêt de bus
SELECT DISTINCT e.libelle, e.typequ, a.stop_id
FROM equipement e
JOIN arrets a ON ST_DWithin(e.geom, a.geom, 300)
WHERE a.typevehic = 'Bus';
----
SELECT 
    i.nom_iris, 
    COUNT(a.stop_id) AS nb_arrets
FROM iris i
LEFT JOIN arrets a ON ST_Intersects(a.geom, i.geom)
WHERE a.typevehic = 'Bus' OR a.typevehic IS NULL
GROUP BY i.nom_iris
ORDER BY nb_arrets DESC;
-------Compter le nombre d'arrêts par type de véhicule
SELECT typevehic, COUNT(*) as nb_arrets
FROM arrets
GROUP BY typevehic;
---- Calcule le nombre d'équipements de santé ou de services par rapport au nombre d'arrêts dans chaque quartier (IRIS).
SELECT i.nom_iris, 
       COUNT(DISTINCT  e.id_equip) AS nb_equipements,
       COUNT(DISTINCT a.stop_id) AS nb_arrets
FROM iris i
LEFT JOIN equipement e ON ST_Intersects(i.geom, e.geom)
LEFT JOIN arrets a ON ST_Intersects(i.geom, a.geom)
GROUP BY i.nom_iris
ORDER BY nb_arrets DESC;
---Trouver quels équipements sont desservis par une ligne de bus 
SELECT DISTINCT e.libelle, l.typevehic
FROM equipement e
JOIN arrets a ON ST_DWithin(e.geom, a.geom, 400)
JOIN desservir d ON a.idarret = d.idarret
JOIN lignes l ON d.idligne = l.idligne
WHERE l.typevehic = 'Tram';
---quelles écoles maternelles sont à moins de 500m d'un arrêt de bus dans une commune donnée 
SELECT 
    e.libelle AS ecole, 
    c.nom_com,
    a.stop_id AS arret_proche
FROM equipement e
JOIN commune c ON ST_Intersects(e.geom, c.geom)
JOIN arrets a ON ST_DWithin(e.geom, a.geom, 500)
WHERE e.libelle ILIKE '%ÉCOLE MATERNELLE%'
AND a.typevehic = 'Tram'
GROUP BY  e.libelle
---- accesibilité au une école maternelle par commune 
SELECT 
    c.nom_com, 
    COUNT(e.idequipe) AS nombre_ECOLE
FROM commune c
JOIN equipement e ON ST_Intersects(e.geom, c.geom)
WHERE e.libelle ILIKE '%ÉCOLE MATERNELLE%'
GROUP BY c.nom_com
ORDER BY nombre_pharmacies DESC;
-----Communes avec population (carreaux) sans pharmacie
SELECT 
    c.nom_com, 
    SUM(car.men) AS pop_sans_pharmacie
FROM commune c
JOIN carreaux1km car ON ST_Intersects(car.geom, c.geom)
WHERE NOT EXISTS (
    SELECT 1 FROM equipement e 
    WHERE e.libelle ILIKE '%ÉCOLE MATERNELLE%' 
    AND ST_DWithin(car.geom, e.geom, 1000)
)
GROUP BY c.nom_com
HAVING SUM(car.men) > 0
-----Cette requête permet de mettre en évidence les quartiers où la population est forte mais l'accès aux services (santé/écoles) est faible.
CREATE VIEW score_accessibilité AS
WITH PopTable AS (
    -- Somme des individus (ind) par IRIS via la table de liaison recouvrir
    SELECT 
        r.idiris, 
        SUM(car.ind) AS population_totale
    FROM recouvrir r
    JOIN carreaux1km car ON r.idcarreaux = car.idcarreaux
    GROUP BY r.idiris
)
SELECT 
    i.idiris,
    i.nom_iris,
    i.nom_commun,
    COALESCE(p.population_totale, 0) AS population_ind,
    COUNT(e.id_equip) AS nb_serv,
    -- Calcul du score pour 1000 individus (habitants)
    CASE 
        WHEN p.population_totale > 0 THEN 
            ROUND((COUNT(e.id_equip)::numeric / p.population_totale) * 1000, 3)
        ELSE 0 
    END AS score_accessibilite,
    i.geom
FROM iris i
-- LEFT JOIN pour conserver tous les IRIS du territoire iris metropole Bordeaux 
LEFT JOIN PopTable p ON i.idiris = p.idiris 
LEFT JOIN equipement e ON i.idiris = e.idiris
GROUP BY i.idiris, i.nom_iris, i.nom_commun, p.population_totale, i.geom;

------ -----Cette requête permet de mettre en évidence les quartiers où la population est forte mais l'accès aux services (santé/écoles) est faible.

CREATE VIEW vue_pression_commune AS
SELECT 
    c.idcom,
    c.nom_com, -- Nom de la colonne dans votre schéma 'communes'
    c.population AS pop_reel, -- Nom exact de la colonne
    SUM(car.ind) AS pop_calculee_carreaux, 
    COUNT(e.id_equip) AS nb_total_equipements,
    -- Calcul du taux pour 10 000 habitants sur la base des individus 'ind'
    CASE 
        WHEN SUM(car.ind) > 0 THEN 
            ROUND((COUNT(e.id_equip)::numeric / SUM(car.ind)) * 10000, 2)
        ELSE 0 
    END AS equipements_pour_10k_hab,
    c.geom
FROM commune c -- Vérifiez si votre table s'appelle 'commmune' ou 'communes'
LEFT JOIN carreaux1km car ON ST_Intersects(car.geom, c.geom)
LEFT JOIN equipement e ON ST_Intersects(e.geom, c.geom)
GROUP BY c.idcom, c.nom_com, c.population, c.geom;
--nb de ligne
SELECT  typevehic, COUNT(*) nb_lignes
FROM lignes
GROUP BY typevehic;
---les lignes qui passe a 100 m d'un cabinet d'un medecin ou pharmacie
SELECT DISTINCT 
    e.libelle AS nom_equipement, 
    l.typevehic AS mode_transport
FROM equipement e
JOIN lignes l ON ST_DWithin(e.geom, l.geom, 100)
WHERE e.libelle ILIKE '%PHARMACIE%' OR e.libelle ILIKE '%MEDECIN%';








