# Audit d'intégrité inter-SGBD — TP4

**Big Data Engineering — Master 1 Data Science — ISI / UCAD**  
Notebook : `notebooks/TP4_federation_multi_sgbd.ipynb`

La fédération Spark joint quatre tables qui ne partagent **aucune clé étrangère**.
Les anti-jointures (`LEFT ANTI JOIN`) révèlent trois orphelins volontairement
injectés dans les scripts `init_tp4_*.sql`.

---

## 1. Trois anomalies d'intégrité (partie 7)

### Anomalie 1 — commande sans client

| Champ | Valeur |
|---|---|
| Identifiant | `CMD21` / `client_id = C999` |
| Table source | `commandes` |
| SGBD | Oracle XE (`SPARK_USER`) |
| Impact métier | Commande **livrée** (15 000 FCFA, casque JBL) rattachée à un client inexistant dans le CRM MySQL : impossible de facturer, de livrer un reçu, ou d'attribuer le CA à une ville. |

### Anomalie 2 — commande sans produit

| Champ | Valeur |
|---|---|
| Identifiant | `CMD22` / `produit_id = P999` |
| Table source | `commandes` |
| SGBD | Oracle XE (`SPARK_USER`) |
| Impact métier | Commande **livrée** (50 000 FCFA) sur un article absent du catalogue PostgreSQL : le stock et la catégorie sont inconnus, le CA par rayon est faussé. |

### Anomalie 3 — paiement orphelin

| Champ | Valeur |
|---|---|
| Identifiant | `PAY27` / `commande_id = CMD99` |
| Table source | `paiements` |
| SGBD | SQL Server (`ecommerce_paiements`) |
| Impact métier | Encaissement **réussi** de 99 000 FCFA (Wave) sans commande Oracle correspondante : l'argent est en caisse mais n'apparaît pas dans la vue fédérée — écart trésorerie / CA. |

### Exercice 8 — livrées sans paiement réussi

**0 ligne.** Les 24 commandes `livree` ont toutes un paiement `reussi`.
Les impayés / échecs (`CMD07`, `CMD12`, `CMD15`, `CMD26`, `CMD28`, `CMD30`)
concernent des commandes `annulee` ou `en_cours`, hors périmètre relance
« livrée non encaissée ».

---

## 2. Tableau de relevés

| Mesure | Valeur |
|---|---:|
| nb_clients (MySQL) | 12 |
| nb_produits (PostgreSQL) | 10 |
| nb_commandes (Oracle) | 30 |
| nb_paiements (SQL Server) | 27 |
| jointure commandes-clients | 29 |
| jointure 3 tables | 28 |
| vue federee (4 SGBD) | 22 |
| commandes sans client | 1 |
| commandes sans produit | 1 |
| paiements orphelins | 1 |
| livrees sans paiement reussi | 0 |
| CA federe total (FCFA) | 1 452 000 |

### Détail du CA fédéré (22 ventes livrées et payées)

**Par ville** (FCFA, décroissant) : Dakar 723 000 · Thiès 362 000 · Ziguinchor 103 000 · Rufisque 85 000 · Saint-Louis 60 000 · Mbour 46 000 · Kaolack 39 000 · Touba 28 000 · Diourbel 6 000.

**Par catégorie** : Informatique 740 000 · Téléphonie 400 000 · Mode 174 000 · Beauté 84 000 · Électroménager 54 000.

**Par moyen de paiement** : Wave 537 000 · carte_bancaire 395 000 · Orange Money 371 000 · especes_livraison 76 000 · Free Money 73 000.

---

## 3. Entonnoir des jointures internes

```
30 commandes Oracle
  − 1  CMD21 (client C999 absent de MySQL)          → 29
  − 1  CMD22 (produit P999 absent de PostgreSQL)    → 28
  − 6  non livrées ou sans paiement réussi          → 22  vue fédérée
```

Les 22 lignes écrites dans `analytics.ventes_federees` (PostgreSQL) sont
exactement celles de la vue : lecture 4 SGBD → consolidation → entrepôt.
