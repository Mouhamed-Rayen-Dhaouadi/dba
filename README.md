# 📘 Révision Administration des Bases de Données (DBA Oracle)

Fiche de révision basée sur les TP1 à TP4 et l'Atelier 6 (Data Pump + SQL*Loader).

## Sommaire

1. [Bases : connexion et SQL*Plus](#1-bases--connexion-et-sqlplus)
2. [Le dictionnaire de données (vues)](#2-le-dictionnaire-de-données-vues)
3. [Comment consulter une vue](#3-comment-consulter-une-vue)
4. [Rappels SQL : créer et manipuler une table](#4-rappels-sql--créer-et-manipuler-une-table)
5. [Rappels PL/SQL : bloc, procédure, fonction](#5-rappels-plsql--bloc-procédure-fonction)
6. [Fonctions SQL utiles (UPPER, LOWER...)](#6-fonctions-sql-utiles)
7. [TP1 : dictionnaire, utilisateurs, SGA](#7-tp1--dictionnaire-utilisateurs-sga)
8. [TP2 : tablespaces et fichiers de données](#8-tp2--tablespaces-et-fichiers-de-données)
9. [TP3 : profils et mots de passe](#9-tp3--profils-et-mots-de-passe)
10. [TP4 : rôles et privilèges](#10-tp4--rôles-et-privilèges)
11. [Data Pump (expdp / impdp)](#11-data-pump-expdp--impdp)
12. [SQL*Loader](#12-sqlloader)
13. [Pièges et erreurs fréquentes](#13-pièges-et-erreurs-fréquentes)

---

## 1. Bases : connexion et SQL*Plus

```sql
-- Depuis le terminal
sqlplus / as sysdba                 -- admin (SYS) en local
sqlplus system/motdepasse           -- admin SYSTEM
sqlplus hr/hr                       -- utilisateur HR
sqlplus user1/pwd1@XEPDB1           -- vers une PDB (Oracle XE 18c/21c)

-- Dans SQL*Plus
CONN hr/hr                          -- changer de compte
SHOW USER                           -- qui suis-je ?
ALTER SESSION SET CONTAINER = XEPDB1;   -- se placer dans la PDB (12c+)
```

> **12c et plus (CDB/PDB)** : pour créer un utilisateur sans préfixe `C##` dans le CDB racine :
> `ALTER SESSION SET "_ORACLE_SCRIPT" = TRUE;` (ou se connecter à la PDB).

### Réglages d'affichage SQL*Plus

```sql
SET SERVEROUTPUT ON            -- afficher DBMS_OUTPUT.PUT_LINE
SET LINESIZE 200               -- largeur de ligne
SET PAGESIZE 100               -- lignes par page
COLUMN username FORMAT A20     -- largeur d'une colonne texte
COLUMN bytes FORMAT 999,999,999
SHOW ERRORS                    -- erreurs de compilation PL/SQL
```

---

## 2. Le dictionnaire de données (vues)

Le dictionnaire contient les **métadonnées** de la base. Il est en lecture seule, et on le consulte avec `SELECT`.

### Les 4 familles de préfixes

| Préfixe | Contenu | Qui peut voir |
|---|---|---|
| `USER_` | Objets **dont je suis propriétaire** | Tout le monde |
| `ALL_` | Objets **auxquels j'ai accès** | Tout le monde |
| `DBA_` | **Tous** les objets du serveur | Administrateur (ou privilège `SELECT ANY DICTIONARY`) |
| `V$` / `GV$` | Vues **dynamiques de performance** (état de l'instance en mémoire) | Administrateur |

### Vues principales par thème

#### Dictionnaire lui-même
| Vue | Rôle |
|---|---|
| `DICTIONARY` (ou `DICT`) | Liste de toutes les vues du dictionnaire (`table_name`, `comments`) |
| `DICT_COLUMNS` | Colonnes de chaque vue du dictionnaire |

#### Utilisateurs, profils, sessions
| Vue | Rôle |
|---|---|
| `DBA_USERS` / `ALL_USERS` | Utilisateurs : `username`, `created`, `default_tablespace`, `temporary_tablespace`, `profile`, `account_status`, `lock_date` |
| `USER_USERS` | Infos sur l'utilisateur connecté |
| `V$SESSION` | Sessions connectées : `username`, `osuser`, `machine`, `status`, `logon_time` |
| `DBA_PROFILES` | Ressources et paramètres de mot de passe de chaque profil (`profile`, `resource_name`, `limit`) |

#### Objets et tables
| Vue | Rôle |
|---|---|
| `USER_OBJECTS` / `ALL_OBJECTS` / `DBA_OBJECTS` | `object_name`, `object_type`, `owner`, `created`, `last_ddl_time`, `status` |
| `USER_TABLES` / `ALL_TABLES` / `DBA_TABLES` | Tables : `table_name`, `owner`, `tablespace_name`, `num_rows` |
| `USER_TAB_COLUMNS` | Colonnes des tables : `column_name`, `data_type`, `data_length` |
| `USER_CONSTRAINTS` | Contraintes (PK, FK, CHECK...) |
| `TAB` | Vue de compatibilité : tables/vues de l'utilisateur courant |
| `USER_SOURCE` | Code source des procédures et fonctions |
| `USER_ERRORS` | Erreurs de compilation PL/SQL |

#### Stockage (tablespaces)
| Vue | Rôle |
|---|---|
| `DBA_TABLESPACES` | Tablespaces : `tablespace_name`, `contents` (PERMANENT/TEMPORARY/UNDO), `status` |
| `DBA_DATA_FILES` | Fichiers de données : `file_name`, `tablespace_name`, `bytes`, `autoextensible`, `maxbytes` |
| `DBA_TEMP_FILES` | Fichiers des tablespaces temporaires |
| `DBA_FREE_SPACE` | Espace libre par tablespace |
| `DBA_SEGMENTS` | Segments (espace occupé par objet) |
| `V$DATAFILE`, `V$TABLESPACE`, `V$TEMPFILE` | Équivalents dynamiques |
| `DATABASE_PROPERTIES` | Propriétés de la base : `DEFAULT_PERMANENT_TABLESPACE`, `DEFAULT_TEMP_TABLESPACE` |

#### Privilèges et rôles
| Vue | Rôle |
|---|---|
| `DBA_SYS_PRIVS` / `USER_SYS_PRIVS` | Privilèges **système** accordés (`grantee`, `privilege`, `admin_option`) |
| `DBA_TAB_PRIVS` / `USER_TAB_PRIVS` | Privilèges **objet** (`grantee`, `owner`, `table_name`, `privilege`, `grantable`) |
| `USER_TAB_PRIVS_RECD` | Privilèges objet **reçus** par l'utilisateur |
| `DBA_ROLES` | Liste des rôles |
| `DBA_ROLE_PRIVS` / `USER_ROLE_PRIVS` | Rôles accordés à un utilisateur ou à un rôle |
| `ROLE_SYS_PRIVS` | Privilèges système contenus dans les rôles **accessibles** |
| `ROLE_TAB_PRIVS` | Privilèges objet contenus dans les rôles accessibles |
| `SESSION_PRIVS` | Privilèges **actifs** dans la session courante |
| `SESSION_ROLES` | Rôles **actifs** dans la session courante |

#### Mémoire et instance
| Vue | Rôle |
|---|---|
| `V$SGA` | Composants de la SGA (`name`, `value`) |
| `V$SGAINFO` | Détail de la SGA |
| `V$PARAMETER` | Paramètres d'initialisation (`SHOW PARAMETER nom`) |
| `V$INSTANCE`, `V$DATABASE` | Infos sur l'instance et la base |

#### Data Pump
| Vue | Rôle |
|---|---|
| `DBA_DIRECTORIES` | Directories Oracle (`directory_name`, `directory_path`) |
| `DBA_DATAPUMP_JOBS` | Jobs expdp/impdp en cours |

---

## 3. Comment consulter une vue

### Méthode en 4 étapes

```sql
-- 1) Trouver la vue qui m'intéresse (recherche par mot-clé)
SELECT table_name, comments
FROM   dictionary
WHERE  table_name LIKE '%TABLESPACE%'
ORDER  BY table_name;

-- 2) Voir sa structure (colonnes)
DESC dba_users;
-- ou
SELECT column_name, comments FROM dict_columns WHERE table_name = 'DBA_USERS';

-- 3) Lire son contenu (choisir les colonnes utiles)
SELECT username, created, account_status FROM dba_users ORDER BY created;

-- 4) Filtrer
SELECT username, profile FROM dba_users WHERE username = 'TP3';
```

### Règles importantes

- Les **noms d'objets sont stockés en MAJUSCULES** dans le dictionnaire : `WHERE username = 'HR'` ou `WHERE username = UPPER(p_user)`.
- Une valeur entre apostrophes est **sensible à la casse**.
- Pour compter : `SELECT COUNT(*) FROM dba_tables;`
- Pour regrouper : `SELECT tablespace_name, COUNT(*) FROM dba_data_files GROUP BY tablespace_name;`
- `DESC` sert à voir les colonnes avant d'écrire la requête.

---

## 4. Rappels SQL : créer et manipuler une table

### Créer une table

```sql
CREATE TABLE Etudiants (
    matricule     NUMBER(10)    PRIMARY KEY,
    nom           VARCHAR2(30)  NOT NULL,
    prenom        VARCHAR2(30),
    tel           VARCHAR2(15),
    dateNaissance DATE,
    moyenne       NUMBER(4,2)   DEFAULT 0 CHECK (moyenne BETWEEN 0 AND 20)
);
```

### Types courants

| Type | Usage |
|---|---|
| `NUMBER(p,s)` | Nombre (p chiffres, s décimales) |
| `VARCHAR2(n)` | Texte de longueur variable (max n) |
| `CHAR(n)` | Texte de longueur fixe |
| `DATE` | Date + heure |
| `TIMESTAMP` | Date avec fractions de seconde |
| `CLOB` / `BLOB` | Gros texte / données binaires |

### Contraintes avec nom, clé étrangère

```sql
CREATE TABLE Inscriptions (
    id_insc   NUMBER PRIMARY KEY,
    matricule NUMBER(10),
    CONSTRAINT fk_insc_etud FOREIGN KEY (matricule) REFERENCES Etudiants(matricule)
);
```

### Modifier / supprimer

```sql
ALTER TABLE Etudiants ADD (email VARCHAR2(50));
ALTER TABLE Etudiants MODIFY (nom VARCHAR2(40));
ALTER TABLE Etudiants DROP COLUMN email;
ALTER TABLE Etudiants RENAME TO Etudiant;
DROP TABLE Etudiants;                       -- va dans la corbeille
DROP TABLE Etudiants PURGE;                 -- suppression définitive
```

### Manipuler les données (DML)

```sql
INSERT INTO Etudiants (matricule, nom, prenom) VALUES (1, 'Ben Ali', 'Sami');
UPDATE Etudiants SET nom = 'Trabelsi' WHERE matricule = 1;
DELETE FROM Etudiants WHERE matricule = 1;
COMMIT;      -- valider
ROLLBACK;    -- annuler
```

### Séquence (numéro automatique)

```sql
CREATE SEQUENCE seq_etud START WITH 1 INCREMENT BY 1;
INSERT INTO Etudiants (matricule, nom) VALUES (seq_etud.NEXTVAL, 'Test');
```

---

## 5. Rappels PL/SQL : bloc, procédure, fonction

### Structure d'un bloc anonyme

```sql
SET SERVEROUTPUT ON
DECLARE
    v_nom   VARCHAR2(30);
    v_nb    NUMBER := 0;
BEGIN
    SELECT COUNT(*) INTO v_nb FROM dba_users;
    DBMS_OUTPUT.PUT_LINE('Nombre d''utilisateurs : ' || v_nb);
EXCEPTION
    WHEN NO_DATA_FOUND THEN DBMS_OUTPUT.PUT_LINE('Rien trouvé');
    WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Erreur : ' || SQLERRM);
END;
/
```

> Le `/` seul sur une ligne **exécute** le bloc dans SQL*Plus.
> Une apostrophe dans un texte se double : `''`.

### Parcourir des résultats : boucle FOR (la plus simple)

```sql
BEGIN
    FOR r IN (SELECT username, created FROM dba_users ORDER BY username) LOOP
        DBMS_OUTPUT.PUT_LINE(r.username || ' - ' || TO_CHAR(r.created, 'DD/MM/YYYY'));
    END LOOP;
END;
/
```

### Curseur explicite

```sql
DECLARE
    CURSOR c_tab IS SELECT table_name FROM user_tables;
    v_name user_tables.table_name%TYPE;     -- même type que la colonne
BEGIN
    OPEN c_tab;
    LOOP
        FETCH c_tab INTO v_name;
        EXIT WHEN c_tab%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_name);
    END LOOP;
    CLOSE c_tab;
END;
/
```

### Procédure stockée

```sql
CREATE OR REPLACE PROCEDURE ps_tables_user (p_user IN VARCHAR2) IS
BEGIN
    FOR t IN (SELECT table_name FROM dba_tables WHERE owner = UPPER(p_user)) LOOP
        DBMS_OUTPUT.PUT_LINE(t.table_name);
    END LOOP;
END ps_tables_user;
/

-- Appel
EXEC ps_tables_user('HR');
-- ou
BEGIN ps_tables_user('HR'); END;
/
```

### Fonction stockée

```sql
CREATE OR REPLACE FUNCTION fn_nb_objets (p_user IN VARCHAR2) RETURN NUMBER IS
    v_nb NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_nb FROM dba_objects WHERE owner = UPPER(p_user);
    RETURN v_nb;
END fn_nb_objets;
/

-- Appels
SELECT fn_nb_objets('HR') FROM dual;
EXEC DBMS_OUTPUT.PUT_LINE(fn_nb_objets('HR'));
```

### Différence procédure / fonction

| | Procédure | Fonction |
|---|---|---|
| Retourne une valeur | Non (sauf paramètres `OUT`) | **Oui** (`RETURN`) |
| Appel | `EXEC proc(...)` | Dans un `SELECT` ou une expression |

### Voir, déboguer, supprimer

```sql
SHOW ERRORS;                                         -- après un CREATE avec erreurs
SELECT * FROM user_errors;
SELECT text FROM user_source WHERE name = 'FN_NB_OBJETS' ORDER BY line;
SELECT object_name, status FROM user_objects WHERE object_type IN ('PROCEDURE','FUNCTION');
DROP PROCEDURE ps_tables_user;
DROP FUNCTION fn_nb_objets;
```

### Exécution dynamique (utile pour DDL dans PL/SQL)

```sql
BEGIN
    EXECUTE IMMEDIATE 'CREATE TABLE t_test (id NUMBER)';
END;
/
```

---

## 6. Fonctions SQL utiles

### Chaînes de caractères

| Fonction | Exemple | Résultat |
|---|---|---|
| `UPPER(x)` | `UPPER('tunis')` | `TUNIS` |
| `LOWER(x)` | `LOWER('TUNIS')` | `tunis` |
| `INITCAP(x)` | `INITCAP('ali ben salah')` | `Ali Ben Salah` |
| `LENGTH(x)` | `LENGTH('bonjour')` | `7` |
| `SUBSTR(x, début, long)` | `SUBSTR('ORACLE', 2, 3)` | `RAC` |
| `INSTR(x, y)` | `INSTR('ORACLE', 'A')` | `3` |
| `REPLACE(x, a, b)` | `REPLACE('2024-01', '-', '/')` | `2024/01` |
| `CONCAT(a, b)` ou `a \|\| b` | `'Bon' \|\| 'jour'` | `Bonjour` |
| `TRIM(x)` / `LTRIM` / `RTRIM` | `TRIM('  ab  ')` | `ab` |
| `LPAD(x, n, c)` / `RPAD` | `LPAD('7', 3, '0')` | `007` |
| `REGEXP_LIKE(x, motif)` | `REGEXP_LIKE('a@b', '[?!@]')` | `TRUE` |

### Nombres

| Fonction | Exemple | Résultat |
|---|---|---|
| `ROUND(x, n)` | `ROUND(15.678, 1)` | `15.7` |
| `TRUNC(x, n)` | `TRUNC(15.678, 1)` | `15.6` |
| `MOD(x, y)` | `MOD(10, 3)` | `1` |
| `CEIL(x)` / `FLOOR(x)` | `CEIL(4.1)` | `5` |
| `ABS(x)` | `ABS(-5)` | `5` |

### Dates

| Fonction | Exemple |
|---|---|
| `SYSDATE` | Date et heure courantes |
| `ADD_MONTHS(d, n)` | `ADD_MONTHS(SYSDATE, 3)` |
| `MONTHS_BETWEEN(d1, d2)` | Nombre de mois entre deux dates |
| `LAST_DAY(d)` | Dernier jour du mois |
| `TRUNC(d)` | Date sans l'heure |
| `TO_CHAR(d, 'DD/MM/YYYY HH24:MI:SS')` | Date vers texte |
| `TO_DATE('12/01/1999', 'DD/MM/YYYY')` | Texte vers date |

### Conversion et NULL

| Fonction | Exemple |
|---|---|
| `TO_NUMBER('12.5')` | Texte vers nombre |
| `TO_CHAR(1234.5, '9999.99')` | Nombre vers texte |
| `NVL(x, valeur)` | `NVL(prime, 0)` : remplace NULL |
| `NVL2(x, si_non_null, si_null)` | |
| `COALESCE(a, b, c)` | Premier non NULL |
| `DECODE(x, v1, r1, v2, r2, défaut)` | Équivalent d'un `CASE` |
| `CASE WHEN ... THEN ... ELSE ... END` | Condition |

### Agrégats

`COUNT(*)`, `SUM(x)`, `AVG(x)`, `MIN(x)`, `MAX(x)` avec `GROUP BY` / `HAVING`.

### Table factice

```sql
SELECT UPPER('test'), SYSDATE FROM dual;   -- DUAL = table à une ligne pour tester
```

---

## 7. TP1 : dictionnaire, utilisateurs, SGA

> Connexion administrateur : `sqlplus / as sysdba`

**1. Vues du dictionnaire triées par nom**
```sql
SELECT table_name FROM dictionary ORDER BY table_name;
```

**2. Utilisateurs du serveur + date de création**
```sql
SELECT username, created FROM dba_users ORDER BY created;
```

**3. Utilisateurs connectés à l'instance**
```sql
SELECT username, osuser, machine, status
FROM   v$session
WHERE  username IS NOT NULL;
```

**4. Taille totale de la SGA**
```sql
SELECT SUM(value) AS taille_octets, ROUND(SUM(value)/1024/1024, 2) AS taille_mo
FROM   v$sga;
-- ou simplement : SHOW SGA
```

> Connexion `HR` : `CONN hr/hr`

**5. Objets de HR : type, création, dernière modification**
```sql
SELECT object_name, object_type, created, last_ddl_time
FROM   user_objects
ORDER  BY object_type, object_name;
```

**6. Tables sur lesquelles HR a des droits**
```sql
SELECT DISTINCT owner, table_name FROM all_tab_privs_recd;   -- droits reçus
-- ou toutes les tables accessibles :
SELECT owner, table_name FROM all_tables;
```

**7. Procédure : tables dont HR est propriétaire**
```sql
CREATE OR REPLACE PROCEDURE ps_mes_tables IS
BEGIN
    FOR t IN (SELECT table_name FROM user_tables ORDER BY table_name) LOOP
        DBMS_OUTPUT.PUT_LINE(t.table_name);
    END LOOP;
END;
/
SET SERVEROUTPUT ON
EXEC ps_mes_tables;
```

> Retour au compte administrateur.

**8. Nombre total de tables du serveur**
```sql
SELECT COUNT(*) FROM dba_tables;
```

**9. Nombre de tables créées par HR**
```sql
SELECT COUNT(*) FROM dba_tables WHERE owner = 'HR';
```

**10. Fonction : nombre d'objets d'un utilisateur**
```sql
CREATE OR REPLACE FUNCTION fn_nb_objets (p_user VARCHAR2) RETURN NUMBER IS
    v_nb NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_nb FROM dba_objects WHERE owner = UPPER(p_user);
    RETURN v_nb;
END;
/
SELECT fn_nb_objets('HR') FROM dual;
```

**11. Procédure : lister les tables d'un utilisateur (test avec HR)**
```sql
CREATE OR REPLACE PROCEDURE ps_lister_tables (p_user VARCHAR2) IS
BEGIN
    FOR t IN (SELECT table_name FROM dba_tables
              WHERE owner = UPPER(p_user) ORDER BY table_name) LOOP
        DBMS_OUTPUT.PUT_LINE(t.table_name);
    END LOOP;
END;
/
SET SERVEROUTPUT ON
EXEC ps_lister_tables('HR');
```

---

## 8. TP2 : tablespaces et fichiers de données

### Notions

- **Tablespace** : espace logique de stockage. Il est composé d'un ou plusieurs **fichiers de données** (`.dbf`).
- **Permanent** : stocke les tables et index. **Temporaire** : stocke les tris. **Undo** : annulation des transactions.
- Chaque utilisateur possède un tablespace **par défaut** et un tablespace **temporaire**.

> Remplace le chemin par celui de ta base, par exemple `C:\oraclexe\app\oracle\oradata\XE\`.
> Pour le connaître : `SELECT file_name FROM dba_data_files;`

**1. TBL01 (10 Mo en 2 fichiers)**
```sql
CREATE TABLESPACE TBL01
DATAFILE 'C:\oraclexe\app\oracle\oradata\XE\fd01tbl01.dbf' SIZE 6M,
         'C:\oraclexe\app\oracle\oradata\XE\fd02tbl01.dbf' SIZE 4M;
```

**2. TBL01 tablespace par défaut du serveur**
```sql
ALTER DATABASE DEFAULT TABLESPACE TBL01;
-- vérifier :
SELECT property_value FROM database_properties
WHERE  property_name = 'DEFAULT_PERMANENT_TABLESPACE';
```

**3. TBL02 (25 Mo en 3 fichiers)**
```sql
CREATE TABLESPACE TBL02
DATAFILE 'C:\oraclexe\app\oracle\oradata\XE\fd01tbl02.dbf' SIZE 10M,
         'C:\oraclexe\app\oracle\oradata\XE\fd02tbl02.dbf' SIZE 10M,
         'C:\oraclexe\app\oracle\oradata\XE\fd0xbl02.dbf'  SIZE 5M;
```

**4. Ajouter un fichier de 20 Mo à TBL01**
```sql
ALTER TABLESPACE TBL01
ADD DATAFILE 'C:\oraclexe\app\oracle\oradata\XE\fd02tbl01.dbf' SIZE 20M;
```
> ⚠️ Ce nom existe déjà (créé à la question 1) : Oracle renvoie `ORA-01537`. Utilise un autre nom, par exemple `fd03tbl01.dbf`.

**5. Renommer `fd0xbl02.dbf` en `fd03bl02.dbf`**
```sql
ALTER TABLESPACE TBL02 OFFLINE;
-- Renommer le fichier physiquement dans l'explorateur Windows / avec la commande ren
ALTER TABLESPACE TBL02 RENAME DATAFILE
    'C:\oraclexe\app\oracle\oradata\XE\fd0xbl02.dbf'
 TO 'C:\oraclexe\app\oracle\oradata\XE\fd03bl02.dbf';
ALTER TABLESPACE TBL02 ONLINE;
```

**6. Liste des tablespaces**
```sql
SELECT tablespace_name FROM dba_tablespaces;
```

**7. Bloc PL/SQL : nom de chaque tablespace + nombre de fichiers**
```sql
SET SERVEROUTPUT ON
BEGIN
    FOR r IN (SELECT tablespace_name, COUNT(*) nb FROM dba_data_files GROUP BY tablespace_name
              UNION ALL
              SELECT tablespace_name, COUNT(*) FROM dba_temp_files GROUP BY tablespace_name) LOOP
        DBMS_OUTPUT.PUT_LINE(r.tablespace_name || ' : ' || r.nb || ' fichier(s)');
    END LOOP;
END;
/
```

**8. Fichier extensible (2 Mo, +1 Mo, max 4 Mo)**
```sql
ALTER TABLESPACE TBL01
ADD DATAFILE 'C:\oraclexe\app\oracle\oradata\XE\fd04tlb01.dbf' SIZE 2M
AUTOEXTEND ON NEXT 1M MAXSIZE 4M;
```

**9. Tablespace temporaire MonTemp, par défaut de la base**
```sql
CREATE TEMPORARY TABLESPACE MonTemp
TEMPFILE 'C:\oraclexe\app\oracle\oradata\XE\montemp01.dbf' SIZE 5M;

ALTER DATABASE DEFAULT TEMPORARY TABLESPACE MonTemp;
```
> Un tablespace temporaire utilise `TEMPFILE` (et non `DATAFILE`).

**10. Fonction FN_NBR_TAB_TEMP**
```sql
CREATE OR REPLACE FUNCTION FN_NBR_TAB_TEMP RETURN NUMBER IS
    v_nb NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_nb FROM dba_tablespaces WHERE contents = 'TEMPORARY';
    RETURN v_nb;
END;
/
SELECT FN_NBR_TAB_TEMP FROM dual;
```

**11. Utilisateur TD3 sur TBL01 + vérifier son tablespace temporaire**
```sql
CREATE USER TD3 IDENTIFIED BY td3
DEFAULT TABLESPACE TBL01
QUOTA UNLIMITED ON TBL01;

GRANT CREATE SESSION, CREATE TABLE TO TD3;

SELECT username, default_tablespace, temporary_tablespace
FROM   dba_users WHERE username = 'TD3';
-- temporary_tablespace = MonTemp (le défaut de la base)
```

**12. Supprimer TBL01 et vérifier le tablespace de TD3**
```sql
-- Impossible de supprimer le tablespace par défaut de la base (ORA-12919) :
ALTER DATABASE DEFAULT TABLESPACE USERS;

DROP TABLESPACE TBL01 INCLUDING CONTENTS AND DATAFILES;

SELECT username, default_tablespace FROM dba_users WHERE username = 'TD3';
```
> `INCLUDING CONTENTS` supprime les objets, `AND DATAFILES` supprime aussi les fichiers sur le disque. Après la suppression, TD3 n'a plus de quota sur un tablespace existant : il faut lui en donner un (`ALTER USER TD3 DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;`).

**13. Procédure PS_DETAILS_TAB : nom, taille totale, taille occupée**
```sql
CREATE OR REPLACE PROCEDURE PS_DETAILS_TAB IS
BEGIN
    FOR t IN (
        SELECT d.tablespace_name,
               d.total_mb,
               d.total_mb - NVL(f.free_mb, 0) AS used_mb
        FROM  (SELECT tablespace_name, SUM(bytes)/1024/1024 total_mb
               FROM dba_data_files GROUP BY tablespace_name) d
        LEFT JOIN
              (SELECT tablespace_name, SUM(bytes)/1024/1024 free_mb
               FROM dba_free_space GROUP BY tablespace_name) f
        ON d.tablespace_name = f.tablespace_name
        ORDER BY d.tablespace_name)
    LOOP
        DBMS_OUTPUT.PUT_LINE(RPAD(t.tablespace_name, 15)
            || ' total = '  || ROUND(t.total_mb, 2) || ' Mo'
            || ' | occupé = ' || ROUND(t.used_mb, 2) || ' Mo');
    END LOOP;
END;
/
SET SERVEROUTPUT ON
EXEC PS_DETAILS_TAB;
```

**14. Connexion TD3, création de la table et insertion de 10000 lignes**
```sql
CONN td3/td3
CREATE TABLE Etudiants (
    num_etud     NUMBER(10) PRIMARY KEY,
    nom_etud     VARCHAR2(30),
    moyenne_etud NUMBER(4,2)
);

BEGIN
    FOR i IN 1 .. 10000 LOOP
        INSERT INTO Etudiants (num_etud, nom_etud, moyenne_etud)
        VALUES (i, 'Etudiant' || i, 10);
    END LOOP;
    COMMIT;
END;
/
```
> ⚠️ L'énoncé écrit `insert into Etudiant` : il manque le **s**, sinon `ORA-00942`. Pense aussi au `COMMIT`.

**15. Réexécuter PS_DETAILS_TAB (en administrateur)**
```sql
CONN / as sysdba
EXEC PS_DETAILS_TAB;
```
La taille **occupée** du tablespace de TD3 a augmenté (les 10000 lignes + l'index de la clé primaire), alors que la taille **totale** reste identique.

### Commandes mémo tablespaces

```sql
ALTER TABLESPACE TBL01 OFFLINE;  /  ONLINE;
ALTER DATABASE DATAFILE 'chemin\f.dbf' RESIZE 30M;
ALTER DATABASE DATAFILE 'chemin\f.dbf' AUTOEXTEND ON NEXT 1M MAXSIZE 100M;
ALTER TABLESPACE TBL01 RENAME TO TBL_NEW;
DROP TABLESPACE TBL01;                                        -- doit être vide
DROP TABLESPACE TBL01 INCLUDING CONTENTS AND DATAFILES;
SELECT * FROM v$tablespace;
```

---

## 9. TP3 : profils et mots de passe

### Notions

Un **profil** limite les ressources d'un utilisateur et gère la politique de mot de passe.
Un utilisateur sans profil explicite a le profil `DEFAULT`.

| Paramètre | Rôle | Unité |
|---|---|---|
| `CONNECT_TIME` | Durée max d'une session | minutes |
| `IDLE_TIME` | Inactivité max | minutes |
| `SESSIONS_PER_USER` | Sessions simultanées max | nombre |
| `FAILED_LOGIN_ATTEMPTS` | Échecs avant verrouillage | nombre |
| `PASSWORD_LOCK_TIME` | Durée du verrouillage | **jours** (`5/1440` = 5 min) |
| `PASSWORD_LIFE_TIME` | Durée de vie du mot de passe | jours |
| `PASSWORD_REUSE_TIME` / `PASSWORD_REUSE_MAX` | Réutilisation | jours / nombre |
| `PASSWORD_VERIFY_FUNCTION` | Fonction de contrôle du mot de passe | nom de fonction ou `NULL` |

> Les limites de ressources (`CONNECT_TIME`, `IDLE_TIME`, `SESSIONS_PER_USER`) nécessitent `RESOURCE_LIMIT = TRUE` :
> `ALTER SYSTEM SET resource_limit = TRUE;`

**1. Créer profil_tp3**
```sql
CREATE PROFILE profil_tp3 LIMIT
    CONNECT_TIME          120
    IDLE_TIME             3
    SESSIONS_PER_USER     2
    FAILED_LOGIN_ATTEMPTS 1          -- verrouillage après 1 erreur
    PASSWORD_LOCK_TIME    5/1440;    -- 5 minutes (la valeur est en jours)
```

**2. Utilisateur TP3**
```sql
CREATE USER TP3 IDENTIFIED BY tp3
PROFILE profil_tp3
DEFAULT TABLESPACE USERS
QUOTA 10M ON USERS
TEMPORARY TABLESPACE TEMP;

GRANT CREATE SESSION TO TP3;
```

**3. Vérifier l'attribution du profil**
```sql
SELECT username, profile, default_tablespace, temporary_tablespace
FROM   dba_users WHERE username = 'TP3';

SELECT resource_name, limit FROM dba_profiles WHERE profile = 'PROFIL_TP3';
```

**4. Utilisateurs du serveur**
```sql
SELECT username FROM dba_users ORDER BY username;
```

**5. Procédure : lister les profils**
```sql
CREATE OR REPLACE PROCEDURE ps_lister_profils IS
BEGIN
    FOR p IN (SELECT DISTINCT profile FROM dba_profiles ORDER BY profile) LOOP
        DBMS_OUTPUT.PUT_LINE(p.profile);
    END LOOP;
END;
/
EXEC ps_lister_profils;
```

**6. Fonction verif_password**

Version demandée par l'énoncé (2 paramètres) :
```sql
CREATE OR REPLACE FUNCTION verif_password (p_username VARCHAR2, p_password VARCHAR2)
RETURN BOOLEAN IS
BEGIN
    IF LENGTH(p_password) <= 6 THEN
        RETURN FALSE;                                   -- longueur > 6
    END IF;
    IF NOT REGEXP_LIKE(p_password, '[?!@]') THEN
        RETURN FALSE;                                   -- au moins un de ? ! @
    END IF;
    IF UPPER(p_username) = UPPER(p_password) THEN
        RETURN FALSE;                                   -- nom différent du mot de passe
    END IF;
    RETURN TRUE;
END;
/
```

> ⚠️ **Oracle exige 3 paramètres** pour une fonction de vérification attachée à un profil :
> `(username, password, old_password)`. Avec 2 paramètres, la fonction se compile, mais le `ALTER PROFILE` ou le `CREATE USER` échoue. Pour qu'elle fonctionne réellement, utilise cette version, **créée sous SYS** :

```sql
CREATE OR REPLACE FUNCTION verif_password
    (p_username VARCHAR2, p_password VARCHAR2, p_old_password VARCHAR2)
RETURN BOOLEAN IS
BEGIN
    IF LENGTH(p_password) <= 6 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Mot de passe trop court (> 6 caracteres)');
    END IF;
    IF NOT REGEXP_LIKE(p_password, '[?!@]') THEN
        RAISE_APPLICATION_ERROR(-20002, 'Il faut au moins un caractere parmi ? ! @');
    END IF;
    IF UPPER(p_username) = UPPER(p_password) THEN
        RAISE_APPLICATION_ERROR(-20003, 'Le mot de passe doit etre different du nom');
    END IF;
    RETURN TRUE;
END;
/
```

**7. Attacher la fonction au profil**
```sql
ALTER PROFILE profil_tp3 LIMIT PASSWORD_VERIFY_FUNCTION verif_password;
```

**8. Tester**
```sql
CREATE USER testtp31 IDENTIFIED BY testtp31 PROFILE profil_tp3;   -- refusé
CREATE USER testtp32 IDENTIFIED BY system   PROFILE profil_tp3;   -- refusé
CREATE USER testtp33 IDENTIFIED BY managers PROFILE profil_tp3;   -- refusé

-- Les 3 échouent, car aucun ne contient ? ! @. Un cas qui réussit :
CREATE USER testtp34 IDENTIFIED BY "Pass@word1" PROFILE profil_tp3;
```

| Utilisateur | Mot de passe | Pourquoi refusé |
|---|---|---|
| testtp31 | testtp31 | Aucun `? ! @` et identique au nom |
| testtp32 | system | 6 caractères (doit être > 6), aucun `? ! @` |
| testtp33 | managers | Aucun `? ! @` |

**9. Supprimer le profil et vérifier TP3**
```sql
DROP PROFILE profil_tp3 CASCADE;     -- CASCADE obligatoire si des utilisateurs l'utilisent

SELECT username, profile FROM dba_users WHERE username = 'TP3';
-- profile = DEFAULT
```

### Commandes mémo

```sql
ALTER USER TP3 PROFILE profil_tp3;
ALTER USER TP3 ACCOUNT LOCK;   /  ACCOUNT UNLOCK;
ALTER USER TP3 IDENTIFIED BY nouveau_mdp;
ALTER USER TP3 PASSWORD EXPIRE;
ALTER USER TP3 QUOTA 20M ON USERS;
DROP USER TP3 CASCADE;                       -- supprime aussi ses objets
ALTER PROFILE profil_tp3 LIMIT PASSWORD_VERIFY_FUNCTION NULL;   -- retirer la fonction
```

---

## 10. TP4 : rôles et privilèges

### Notions

| Type | Description | Exemples |
|---|---|---|
| **Privilège système** | Droit d'effectuer une action | `CREATE SESSION`, `CREATE TABLE`, `CREATE ROLE`, `CREATE PROCEDURE` |
| **Privilège objet** | Droit sur un objet précis | `SELECT ON hr.employees`, `INSERT ON t` |
| **Rôle** | Ensemble de privilèges nommé, attribuable à des utilisateurs | `CONNECT`, `RESOURCE`, `DBA` |

- `WITH ADMIN OPTION` : (privilège **système**/rôle) le bénéficiaire peut le retransmettre.
- `WITH GRANT OPTION` : (privilège **objet**) le bénéficiaire peut le retransmettre.
- `CREATE PROCEDURE` couvre aussi les **fonctions** (et packages, triggers).

**1. Créer ROLE_TP3**
```sql
CREATE ROLE ROLE_TP3;

GRANT CREATE SESSION   TO ROLE_TP3;                      -- se connecter
GRANT CREATE TABLE     TO ROLE_TP3;                      -- créer des tables
GRANT CREATE ROLE      TO ROLE_TP3;                      -- créer des rôles
GRANT CREATE PROCEDURE TO ROLE_TP3 WITH ADMIN OPTION;    -- procédures + fonctions, transmissible

GRANT SELECT ON hr.employees   TO ROLE_TP3;
GRANT SELECT ON hr.departments TO ROLE_TP3;
```

**2. Attribuer le rôle à TP3**
```sql
GRANT ROLE_TP3 TO TP3;
```

**3. Connexion TP3 et création de la table**
```sql
CONN tp3/tp3
CREATE TABLE Etudiants (
    Matricule     NUMBER PRIMARY KEY,
    nom           VARCHAR2(30),
    prenom        VARCHAR2(30),
    tel           VARCHAR2(15),
    dateNaissance DATE
);
```
> Il faut un quota sur USERS (déjà donné au TP3). Un privilège obtenu par rôle ne remplace pas le quota.

**4. Utilisateur TP3_2 (quota 5M sur USERS, profil_tp3)**
```sql
CONN / as sysdba
CREATE USER TP3_2 IDENTIFIED BY "Pass@word1"
DEFAULT TABLESPACE USERS
QUOTA 5M ON USERS
PROFILE profil_tp3;
```
> Si `profil_tp3` a été supprimé au TP3 (question 9), recrée-le d'abord. Si la fonction `verif_password` est attachée, le mot de passe doit la respecter.

**5. Rôle pour TP3_2**
```sql
GRANT ROLE_TP3 TO TP3_2;
```

**6. TP3_2 : SELECT et INSERT sur Etudiants de TP3**
```sql
-- Connecté en TP3 (propriétaire de la table) :
CONN tp3/tp3
GRANT SELECT, INSERT ON Etudiants TO TP3_2;

-- Vérifier (en administrateur) :
SELECT grantee, owner, table_name, privilege
FROM   dba_tab_privs
WHERE  grantee = 'TP3_2';

-- Tester (en TP3_2) :
CONN tp3_2/"Pass@word1"
INSERT INTO tp3.Etudiants (Matricule, nom, prenom) VALUES (1, 'Ben Ali', 'Sami');
COMMIT;
SELECT * FROM tp3.Etudiants;
```
> Il faut préfixer par le propriétaire (`tp3.Etudiants`), ou créer un synonyme.

**7. Procédure : privilèges système d'un utilisateur (test avec TP3)**
```sql
CREATE OR REPLACE PROCEDURE ps_privs_sys (p_user VARCHAR2) IS
BEGIN
    -- privilèges directs + privilèges venant des rôles
    FOR p IN (SELECT privilege, 'DIRECT' AS origine
              FROM   dba_sys_privs WHERE grantee = UPPER(p_user)
              UNION
              SELECT s.privilege, r.granted_role
              FROM   dba_role_privs r
              JOIN   dba_sys_privs  s ON s.grantee = r.granted_role
              WHERE  r.grantee = UPPER(p_user)
              ORDER  BY 1) LOOP
        DBMS_OUTPUT.PUT_LINE(p.privilege || '   (' || p.origine || ')');
    END LOOP;
END;
/
SET SERVEROUTPUT ON
EXEC ps_privs_sys('TP3');
```
> Version minimale : `SELECT privilege FROM dba_sys_privs WHERE grantee = UPPER(p_user)`. Pour TP3, elle affiche presque seulement `CREATE SESSION`, puisque les autres privilèges viennent du **rôle**.

**8. Procédure : privilèges objet d'un rôle (test avec ROLE_TP3)**
```sql
CREATE OR REPLACE PROCEDURE ps_privs_obj_role (p_role VARCHAR2) IS
BEGIN
    FOR p IN (SELECT owner, table_name, privilege
              FROM   dba_tab_privs
              WHERE  grantee = UPPER(p_role)
              ORDER  BY owner, table_name) LOOP
        DBMS_OUTPUT.PUT_LINE(p.privilege || ' ON ' || p.owner || '.' || p.table_name);
    END LOOP;
END;
/
EXEC ps_privs_obj_role('ROLE_TP3');
-- Résultat : SELECT ON HR.DEPARTMENTS, SELECT ON HR.EMPLOYEES
```

### Commandes mémo

```sql
GRANT <priv> TO <user|role> [WITH ADMIN OPTION];
GRANT <priv> ON <objet> TO <user|role> [WITH GRANT OPTION];
REVOKE <priv> FROM <user|role>;
REVOKE SELECT ON hr.employees FROM ROLE_TP3;
DROP ROLE ROLE_TP3;

SELECT * FROM session_privs;                -- mes privilèges actifs
SELECT * FROM session_roles;                -- mes rôles actifs
SELECT * FROM user_role_privs;              -- mes rôles accordés
SELECT * FROM role_sys_privs  WHERE role = 'ROLE_TP3';
SELECT * FROM role_tab_privs  WHERE role = 'ROLE_TP3';
```

---

## 11. Data Pump (expdp / impdp)

### Notions

- **Data Pump** exporte/importe des données et des métadonnées **côté serveur**, dans des fichiers `.dmp` stockés dans un **Directory Oracle**.
- `expdp` et `impdp` sont des **outils en ligne de commande** : on les lance depuis le terminal Windows (`cmd`), **pas** depuis SQL*Plus.
- Remplace les anciens `exp` / `imp`.
- Privilèges : `EXP_FULL_DATABASE` / `IMP_FULL_DATABASE` pour exporter ou importer d'autres schémas, et `READ`/`WRITE` sur le directory.

### Atelier 6, partie 1 : solution

**1 et 2. Utilisateurs et privilèges** (en administrateur)
```sql
CREATE USER user1 IDENTIFIED BY pwd1 DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
CREATE USER user2 IDENTIFIED BY pwd2 DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;

GRANT CREATE SESSION, CREATE TABLE, EXP_FULL_DATABASE, IMP_FULL_DATABASE TO user1, user2;
```

**3 et 4. Tables et données** (sous `user1`)
```sql
CONN user1/pwd1
CREATE TABLE Test1 (a1 NUMBER, b1 NUMBER);
CREATE TABLE Test2 (a2 NUMBER, b2 NUMBER);

INSERT INTO Test1 VALUES (1, 10);
INSERT INTO Test1 VALUES (2, 20);
INSERT INTO Test1 VALUES (3, 30);
INSERT INTO Test2 VALUES (1, 100);
INSERT INTO Test2 VALUES (2, 200);
INSERT INTO Test2 VALUES (3, 300);
COMMIT;
```

**5. Directory Oracle** (en administrateur, le dossier doit exister sur le disque)
```sql
CREATE OR REPLACE DIRECTORY dir_dump AS 'C:\oraclexe\app\oracle';
SELECT directory_name, directory_path FROM dba_directories;
```

**6. Droits sur le directory**
```sql
GRANT READ, WRITE ON DIRECTORY dir_dump TO user1, user2;
```

**7. Export des 2 tables dans `user1.dump`** (depuis `cmd`)
```bat
expdp user1/pwd1 DIRECTORY=dir_dump DUMPFILE=user1.dump TABLES=Test1,Test2 LOGFILE=exp_user1.log
```
> Avec une PDB : `expdp user1/pwd1@XEPDB1 ...`

**8. Import de Test1, métadonnées seulement, vers user2**
```bat
impdp user2/pwd2 DIRECTORY=dir_dump DUMPFILE=user1.dump TABLES=user1.Test1 REMAP_SCHEMA=user1:user2 CONTENT=METADATA_ONLY LOGFILE=imp1.log
```
> `REMAP_SCHEMA=user1:user2` est indispensable : sans lui, Oracle tente de recréer la table dans `user1`.

**9. Requêtes sous `user2` et explication**
```sql
SELECT * FROM tab;       -- affiche TEST1 (la table existe)
SELECT * FROM Test1;     -- « aucune ligne sélectionnée » (0 ligne)
```
**Pourquoi ?** Avec `CONTENT=METADATA_ONLY`, seule la **structure** (le `CREATE TABLE`) est importée, et aucune donnée n'est copiée. La table existe donc, mais elle est vide.

**10. Import de Test2, métadonnées et données**
```bat
impdp user2/pwd2 DIRECTORY=dir_dump DUMPFILE=user1.dump TABLES=user1.Test2 REMAP_SCHEMA=user1:user2 CONTENT=ALL LOGFILE=imp2.log
```
> `CONTENT=ALL` est la valeur par défaut.

**11. Requêtes sous `user2`**
```sql
SELECT * FROM tab;       -- affiche TEST1 et TEST2
SELECT * FROM Test2;     -- 3 lignes : (1,100) (2,200) (3,300)
```
**Pourquoi ?** Cette fois la structure **et** les données sont importées. Test1 reste vide.

### Paramètres importants

| Paramètre | Rôle |
|---|---|
| `DIRECTORY=` | Directory Oracle utilisé |
| `DUMPFILE=` | Nom du fichier dump |
| `LOGFILE=` | Fichier journal |
| `TABLES=` | Liste de tables (`schema.table`) |
| `SCHEMAS=` | Exporter/importer un ou plusieurs schémas entiers |
| `FULL=Y` | Base entière |
| `CONTENT=` | `ALL` (défaut), `DATA_ONLY`, `METADATA_ONLY` |
| `REMAP_SCHEMA=a:b` | Importer les objets du schéma `a` dans le schéma `b` |
| `REMAP_TABLESPACE=a:b` | Changer de tablespace à l'import |
| `TABLE_EXISTS_ACTION=` | Si la table existe : `SKIP` (défaut), `APPEND`, `TRUNCATE`, `REPLACE` |
| `EXCLUDE=` / `INCLUDE=` | Filtrer des types d'objets (`EXCLUDE=INDEX`) |
| `QUERY=` | Exporter un sous-ensemble de lignes (`QUERY=Test1:"WHERE a1>1"`) |
| `PARALLEL=` | Nombre de processus parallèles |
| `SQLFILE=` | Générer le DDL dans un fichier au lieu d'importer |
| `JOB_NAME=` | Nom du job |

### Exemples utiles

```bat
:: Exporter un schéma complet
expdp system/pwd DIRECTORY=dir_dump DUMPFILE=hr.dmp SCHEMAS=hr LOGFILE=hr.log

:: Exporter toute la base
expdp system/pwd DIRECTORY=dir_dump DUMPFILE=full.dmp FULL=Y

:: Importer en ajoutant les lignes dans une table existante
impdp user2/pwd2 DIRECTORY=dir_dump DUMPFILE=user1.dump TABLES=user1.Test1 REMAP_SCHEMA=user1:user2 CONTENT=DATA_ONLY TABLE_EXISTS_ACTION=APPEND

:: Voir le DDL sans importer
impdp user2/pwd2 DIRECTORY=dir_dump DUMPFILE=user1.dump SQLFILE=ddl.sql
```

### Surveiller et dépanner

```sql
SELECT job_name, operation, state FROM dba_datapump_jobs;
SELECT directory_name, directory_path FROM dba_directories;
```

| Erreur | Cause |
|---|---|
| `ORA-39002` / `ORA-39087` | Directory inexistant ou mal nommé |
| `ORA-39070` / `ORA-39145` | Pas de droit `READ/WRITE` sur le directory, ou chemin inexistant |
| `ORA-31655` | Aucun objet à exporter, mauvais nom de table |
| `ORA-39151` | La table existe déjà (voir `TABLE_EXISTS_ACTION`) |

---

## 12. SQL*Loader

### Notions

**SQL*Loader** (`sqlldr`) charge des données depuis un **fichier texte** vers une table Oracle. Il s'utilise à partir du terminal, avec un **fichier de contrôle** (`.ctl`).

```bat
sqlldr userid=user1/pwd1 control=artiste.ctl log=artiste.log bad=artiste.bad discard=artiste.dsc
```

| Fichier | Contenu |
|---|---|
| `.ctl` | Fichier de contrôle (description du chargement) |
| `.log` | Compte rendu |
| `.bad` | Lignes rejetées (erreur Oracle ou format) |
| `.dsc` | Lignes écartées par la clause `WHEN` |

### Structure d'un fichier de contrôle

```
LOAD DATA
INFILE 'fichier.dat'
[BADFILE 'x.bad'] [DISCARDFILE 'x.dsc']
APPEND | INSERT | REPLACE | TRUNCATE        -- mode de chargement
INTO TABLE ma_table
[WHEN condition]
[FIELDS TERMINATED BY ','  [OPTIONALLY ENCLOSED BY '"']]
[TRAILING NULLCOLS]
(colonne1, colonne2, ...)
```

| Mode | Effet |
|---|---|
| `INSERT` | La table doit être vide |
| `APPEND` | Ajoute aux lignes existantes |
| `REPLACE` | `DELETE` puis charge |
| `TRUNCATE` | `TRUNCATE` puis charge |

### Types de champs et outils

| Élément | Rôle |
|---|---|
| `POSITION(a:b)` | Champ à positions fixes (de la colonne a à b) |
| `CHAR`, `INTEGER EXTERNAL`, `DATE "format"` | Types de données |
| `"UPPER(:col)"` | Appliquer une fonction SQL à la valeur |
| `CONSTANT 'x'` | Valeur fixe |
| `SYSDATE` | Date du chargement |
| `SEQUENCE(MAX,1)` | Numéro séquentiel (max existant + 1) |
| `RECNUM` | Numéro de l'enregistrement dans le fichier |
| `FILLER` | Champ lu mais non chargé |
| `WHEN (1:1) = '0'` | Ne charger que les lignes qui respectent la condition |

### Partie 2, question 1 : table ARTISTE

```sql
CREATE TABLE Artiste (
    id     NUMBER,
    nom    VARCHAR2(30),
    prenom VARCHAR2(30),
    pays   VARCHAR2(30)
);
```

### Question 2 : `artiste.ctl`

> Hypothèse : le format de `artiste.txt` n'est pas donné. On suppose des champs séparés par une virgule, la 1ʳᵉ valeur étant le type de ligne (`0` = artiste), par exemple `0,1,Picasso,Pablo,Espagne`.

```
LOAD DATA
INFILE 'artiste.txt'
BADFILE 'artiste.bad'
DISCARDFILE 'artiste.dsc'
APPEND
INTO TABLE Artiste
WHEN (1:1) = '0'
FIELDS TERMINATED BY ','
TRAILING NULLCOLS
(
  type_ligne FILLER,
  id,
  nom,
  prenom,
  pays
)
```
> `FILLER` : le champ « type » est lu pour passer au suivant, mais n'est pas chargé. La clause `WHEN` écarte toutes les lignes qui ne commencent pas par `0`.

### Question 3 : `emp.dat` vers `modified_data`

Fichier : `11111AAAAAAAAAA991201` = 5 chiffres + 10 caractères + 6 chiffres (date).

```sql
CREATE TABLE modified_data (
    rec_no    NUMBER,
    region    VARCHAR2(2),
    load_date DATE,
    data1     NUMBER,
    data2     VARCHAR2(10),
    data3     DATE
);
```

`emp.ctl` :
```
LOAD DATA
INFILE 'emp.dat'
APPEND
INTO TABLE modified_data
(
  rec_no    SEQUENCE(MAX,1),
  region    CONSTANT '31',
  load_date SYSDATE,
  data1     POSITION(1:5)   INTEGER EXTERNAL,
  data2     POSITION(6:15)  CHAR "UPPER(:data2)",
  data3     POSITION(16:21) DATE "YYMMDD"
)
```
> Avec le format `YYMMDD`, `99` devient **2099**. Pour obtenir 1999, utilise `"RRMMDD"`.

### Question 4 : `emp1.dat` : filtrer avec `WHEN`

Fichier : 4 chiffres (n°) + 10 caractères (nom) + 6 chiffres (date).

```sql
CREATE TABLE Emp (
    empno    NUMBER,
    ename    VARCHAR2(10),
    hiredate DATE
);
```

`emp1.ctl` :
```
LOAD DATA
INFILE 'emp1.dat'
DISCARDFILE 'emp1.dsc'
APPEND
INTO TABLE Emp
WHEN (5:5) = 'B' AND (15:20) = '990112'
(
  empno    POSITION(1:4)   INTEGER EXTERNAL,
  ename    POSITION(5:14)  CHAR,
  hiredate POSITION(15:20) DATE "RRMMDD"
)
```

| Ligne | Nom commence par B ? | Date = 990112 ? | Chargée ? |
|---|---|---|---|
| `1111AAAAAAAAAA991202` | Non | Non | ❌ |
| `2222BBBBBBBBBB990112` | Oui | Oui | ✅ |
| `3333ABBBBBBBBB990112` | Non (commence par A) | Oui | ❌ |
| `4444ABBBBBBBBB990112` | Non (commence par A) | Oui | ❌ |

Seule la ligne **2222** est chargée, et les 3 autres vont dans le fichier `.dsc`.

---

## 13. Pièges et erreurs fréquentes

| Problème | Explication / solution |
|---|---|
| `ORA-00942: table or view does not exist` dans une procédure | Les privilèges obtenus **par rôle** ne marchent pas dans les procédures stockées. Il faut un `GRANT` **direct**. Exemple : `SYSTEM` utilise `DBA_` via le rôle DBA, donc exécute-toi en `SYS` ou fais `GRANT SELECT ANY DICTIONARY TO user;`. |
| Rien ne s'affiche avec `DBMS_OUTPUT` | Oublie de `SET SERVEROUTPUT ON`. |
| Résultat vide sur `WHERE username = 'hr'` | Le dictionnaire stocke en **MAJUSCULES** : `'HR'` ou `UPPER(...)`. |
| `ORA-01950: no privileges on tablespace` | L'utilisateur n'a pas de **quota** : `ALTER USER x QUOTA 10M ON USERS;`. |
| `ORA-01045: user lacks CREATE SESSION` | Il manque `GRANT CREATE SESSION TO x;`. |
| `ORA-12919` en supprimant un tablespace | C'est le tablespace par défaut de la base : en définir un autre avant. |
| `ORA-01537: file already part of database` | Nom de fichier de données déjà utilisé. |
| `ORA-02380: profile does not exist` / `ORA-02382` | `DROP PROFILE` d'un profil utilisé : ajouter `CASCADE`. |
| `ORA-28003` | Le mot de passe est rejeté par la fonction de vérification. |
| `ORA-65096: invalid common user or role name` | Création d'un utilisateur dans le CDB racine : se connecter à la PDB ou utiliser `_ORACLE_SCRIPT`. |
| Création d'un objet sans `COMMIT` | Le DDL valide automatiquement, mais les `INSERT`/`UPDATE`/`DELETE` nécessitent `COMMIT`. |
| `expdp` : commande non reconnue dans SQL*Plus | Se lance depuis `cmd`, jamais dans SQL*Plus. |
| Directory inexistant sur le disque | `CREATE DIRECTORY` ne crée pas le dossier : il doit déjà exister. |
| SQL*Loader : dates en 2099 au lieu de 1999 | Utiliser le format `RR` au lieu de `YY`. |

---

## ✅ Mémo ultra-rapide

```sql
-- Qui suis-je / où suis-je
SHOW USER;  SHOW CON_NAME;

-- Chercher dans le dictionnaire
SELECT table_name FROM dictionary WHERE table_name LIKE '%PRIV%';

-- Utilisateurs
CREATE USER u IDENTIFIED BY p DEFAULT TABLESPACE USERS QUOTA 10M ON USERS PROFILE profil;
GRANT CREATE SESSION, CREATE TABLE TO u;
ALTER USER u ACCOUNT UNLOCK;
DROP USER u CASCADE;

-- Stockage
CREATE TABLESPACE t DATAFILE 'chemin.dbf' SIZE 10M;
CREATE TEMPORARY TABLESPACE tt TEMPFILE 'chemin.dbf' SIZE 5M;
ALTER TABLESPACE t ADD DATAFILE 'chemin2.dbf' SIZE 5M AUTOEXTEND ON NEXT 1M MAXSIZE 20M;
ALTER DATABASE DEFAULT TABLESPACE t;
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE tt;

-- Profils et rôles
CREATE PROFILE p LIMIT SESSIONS_PER_USER 2 FAILED_LOGIN_ATTEMPTS 3;
CREATE ROLE r;  GRANT priv TO r;  GRANT r TO u;
```
