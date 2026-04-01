# Analyse de la base GLPI

## Q1. 10 tables principales de GLPI

1. `glpi_tickets` : table centrale des tickets (incident/demande), contient le statut, la priorite, les dates, les liens vers categorie/entite/SLA.
2. `glpi_users` : utilisateurs GLPI (demandeurs, techniciens, administrateurs), informations d'identite et d'authentification.
3. `glpi_computers` : inventaire des postes informatiques (nom, numero de serie, etat, entite, etc.).
4. `glpi_entities` : structure organisationnelle (filiales/services/sites) utilisee pour cloisonner les donnees.
5. `glpi_itilcategories` : categories ITIL des tickets (incident, demande), utile pour l'analyse fonctionnelle.
6. `glpi_groups` : groupes d'utilisateurs/techniciens pour l'affectation et la gestion des droits.
7. `glpi_tickets_users` : table de liaison entre tickets et utilisateurs (demandeur, observateur, assigne).
8. `glpi_computers_users` : liaison entre ordinateurs et utilisateurs (affectation des equipements).
9. `glpi_slms` : definitions SLA (regles de service, objectifs de delai).
10. `glpi_logs` : traces des actions/evenements (audit) pour le suivi des modifications.

## Q2. Nombre de tickets par statut

```sql
SELECT
  status,
  CASE status
    WHEN 1 THEN 'Nouveau'
    WHEN 2 THEN 'En cours (attribue)'
    WHEN 3 THEN 'En cours (planifie)'
    WHEN 4 THEN 'En attente'
    WHEN 5 THEN 'Resolu'
    WHEN 6 THEN 'Clos'
    ELSE 'Autre'
  END AS statut_libelle,
  COUNT(*) AS total
FROM glpi_tickets
GROUP BY status
ORDER BY status;
```

Statuts GLPI :
- `1` Nouveau
- `2` En cours (attribue)
- `3` En cours (planifie)
- `4` En attente
- `5` Resolu
- `6` Clos

## Q3. Tickets crees par mois (12 derniers mois)

```sql
SELECT
  DATE_FORMAT(date_creation, '%Y-%m-01') AS mois,
  COUNT(*) AS tickets_crees
FROM glpi_tickets
WHERE date_creation >= DATE_FORMAT(DATE_SUB(CURDATE(), INTERVAL 11 MONTH), '%Y-%m-01')
GROUP BY 1
ORDER BY 1;
```

## Q4. Equipements associes a un utilisateur

```sql
SELECT
  u.id AS user_id,
  u.name AS user_login,
  c.id AS computer_id,
  c.name AS computer_name,
  cu.is_dynamic
FROM glpi_users u
JOIN glpi_computers_users cu ON cu.users_id = u.id
JOIN glpi_computers c ON c.id = cu.computers_id
WHERE u.name = 'nom_utilisateur'
ORDER BY c.name;
```

## Q5. SLA : table, champs et relation

- Table SLA : `glpi_slms`
- Champs utiles typiques : `id`, `name`, `comment`, `number_time` (ou champs de delai selon version), `is_active`
- Dans `glpi_tickets`, les liaisons SLA passent generalement par :
  - `slas_id_ttr` (SLA de temps de resolution)
  - `slas_id_tto` (SLA de temps de prise en charge)

Exemple de jointure ticket -> SLA :

```sql
SELECT
  t.id AS ticket_id,
  t.name AS ticket_title,
  t.slas_id_ttr,
  s1.name AS sla_ttr_name,
  t.slas_id_tto,
  s2.name AS sla_tto_name
FROM glpi_tickets t
LEFT JOIN glpi_slms s1 ON s1.id = t.slas_id_ttr
LEFT JOIN glpi_slms s2 ON s2.id = t.slas_id_tto
ORDER BY t.id DESC
LIMIT 50;
```

La relation est une relation de type "plusieurs tickets -> un SLA" via les cles etrangeres de `glpi_tickets` vers `glpi_slms`.
