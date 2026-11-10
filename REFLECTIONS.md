# Réflexions — TD4, séries temporelles

Données utilisées : run de l'Action du 2026-11-10 (run `42751085810`), CO2 horaire
Open-Meteo / CAMS pour Le Mans (48.0061, 0.1996), 2 160 heures du 2026-08-12 au
2026-11-09 agrégées en 90 jours. Pour l'étape 6 : Sydney (-33.8688, 151.2093), même
fenêtre.

---

## 1. Stage 0 — Comment vérifier l'hypothèse « pas de données historiques réelles »

Action concrète : **faire un seul appel de test à l'API avant de concevoir quoi que ce
soit**, en collant dans le navigateur
`https://air-quality-api.open-meteo.com/v1/air-quality?latitude=48.0061&longitude=0.1996&hourly=carbon_dioxide&past_days=90`
et en comptant les lignes renvoyées (ou, à défaut, lire la doc du paramètre
`past_days`). Trente secondes, contre une conception entière construite sur une
hypothèse fausse.

Je l'ai fait pendant ce TD, et ça montre que même la nouvelle hypothèse « jusqu'à ~90
jours » n'était pas vérifiée : `past_days=120` renvoie 2 880 heures, et une requête
`start_date=2025-08-01` renvoie **15 mois** d'historique horaire sans aucun trou
(11 184 heures). La limite réelle est donc bien plus large que celle annoncée par le
script, et c'est elle qui permet de vérifier le cycle saisonnier (réflexions 2 et 6)
au lieu de le supposer. Une hypothèse sur la disponibilité des données se teste par
une requête, pas par un raisonnement.

## 2. Stage 1 — Prédiction : CO2 en hausse, en baisse ou stable ?

**Prédiction (avant de lancer le script) : en hausse.** La fenêtre couvre mi-août →
début novembre. En été, la végétation de l'hémisphère Nord absorbe un maximum de CO2
par photosynthèse : le minimum annuel tombe vers fin août–septembre. À l'automne les
feuilles tombent, la respiration et la décomposition dominent, et le chauffage
redémarre : la concentration remonte jusqu'à un maximum en fin d'hiver. Je m'attendais
donc à une pente positive de quelques dixièmes de ppm par jour.

**Vérification (étape 3) : juste.** Pente de **+0,165 ppm/jour**. Moyennes mensuelles :
août 437,6 → septembre 441,0 → octobre 443,9 → novembre (9 jours) 455,0 ppm.
Minimum 429,2 ppm le 31 août, maximum 470,6 ppm le 6 novembre. Ce que je n'avais pas
prévu, c'est que la hausse **accélère** : pente de +0,07 ppm/jour sur les 45 premiers
jours, +0,22 sur les 45 derniers, +0,54 sur les 30 derniers. Le cycle saisonnier est
une courbe, pas une droite.

## 3. Stage 2 — Zéro trou détecté : peut-on supprimer la détection de trous ?

Non. Mon run donne 0 heure nulle sur 2 160 et 0 jour manquant, et j'ai obtenu la même
chose pour Rennes, Helsinki, Sydney et sur 15 mois au Mans. Mais ça prouve seulement
que **cette source a été propre lors de ces appels**, pas qu'elle le sera toujours.
Rien dans le contrat de l'API ne le garantit : une panne côté Open-Meteo, une heure
récente pas encore calculée par le modèle, une variable qui n'existe que sur un
domaine régional (certaines ne sont calculées que pour l'Europe) ou le passage à un
vrai capteur (étape 6, ou un capteur fixé à un bâtiment) produiraient des trous. Et
le code coûte quelques lignes, alors que son absence aurait un coût silencieux.

Le reindexage par date calendaire a aussi un second rôle : la variable `day` qui sert
d'abscisse à la régression est le **numéro du jour calendaire**, pas le numéro de
ligne. Sans lui, un jour manquant décalerait tous les points suivants d'un jour et
fausserait la pente sans aucun message.

Deux limites que j'ai remarquées en lisant `clean_series()` : un trou **au début ou à
la fin** de la fenêtre n'est pas détecté (l'intervalle commence au premier jour réel
et finit au dernier), et un jour avec moins de 24 heures n'est signalé que dans la
console de `fetch_series.py`, pas dans `series_analysis.json` : le dashboard ne peut
pas le montrer.

## 4. Stage 3 — Moyenne mobile ou lissage exponentiel ?

Sur ma sortie, les deux restent proches (écart moyen 1,5 ppm, max 3,9 ppm), mais ils
divergent sur le pic du 5–7 novembre (449,6 → 459,5 → **470,6** → 463,6 → 454,3 →
451,3 ppm), vraisemblablement un épisode météo (air stagnant) plus qu'un changement
de fond :

| Date | Brut | Moyenne mobile 7 j | Lissage exp. α=0,3 |
|---|---|---|---|
| 05/11 | 459,5 | 450,3 | 451,78 |
| 06/11 | 470,6 | 453,5 | 457,43 |
| 07/11 | 463,6 | 455,6 | 459,28 |
| 08/11 | 454,3 | 456,06 | 457,79 |
| 09/11 | 451,3 | **456,6** | **455,84** |

- **Je préfère la moyenne mobile pour suivre une tendance lente** et publier un
  indicateur hebdomadaire stable (par ex. « niveau moyen de CO2 de la semaine » dans un
  rapport). Un jour exceptionnel ne pèse qu'un septième : le 6 novembre, elle monte de
  3,2 ppm quand le lissage exponentiel monte de 5,7. Et sa définition est explicable
  à n'importe qui (« la moyenne des 7 derniers jours »).
- **Je préfère le lissage exponentiel pour réagir à un changement soudain**, par
  exemple piloter une ventilation ou déclencher une alerte sur un capteur. Il a vu la
  hausse plus vite et il a déjà **redescendu** le 8 novembre, alors que la moyenne
  mobile continuait de monter jusqu'au 9 alors que le brut était revenu à 451.
  Inversement, la moyenne mobile va chuter d'un coup vers le 12–13 novembre, quand le
  pic sortira de la fenêtre : un mouvement qui ne correspond à rien de ce qui se passe
  ce jour-là. Le lissage exponentiel oublie progressivement, sans cet effet de bord.

## 5. Stage 4 — Que se passe-t-il si l'appel API échoue à chaque tentative ?

Les fichiers **restent figés avec les chiffres de la dernière exécution réussie ; le
dashboard ne casse pas, il devient périmé sans le dire.** Raisonnement sur le code :

1. `fetch_hourly_co2()` lève `RuntimeError` après 4 essais (5 + 10 + 15 s d'attente).
   L'écriture de `data/series.csv` est **après** cet appel dans `main()` : le fichier
   n'est même pas ouvert, donc pas d'écrasement à moitié écrit.
2. Le script sort avec un code non nul → l'étape échoue → les étapes suivantes du job
   (analyse, commit) n'ont pas de `if: always()` et sont sautées. Rien n'est commité.
3. GitHub Pages continue de servir le `series_analysis.json` de la semaine précédente.
   Le dashboard affiche normalement ses cartes, son graphique et sa note d'honnêteté.
   Le seul indice est la ligne « Data last analyzed … » avec une date ancienne, et la
   projection affiche des dates **déjà passées** comme si c'était le futur.
4. GitHub envoie un e-mail d'échec à l'auteur du dernier changement de cron, mais le
   visiteur du dashboard n'en sait rien. Si la panne dure, plus aucun commit n'arrive
   et, après 60 jours sans activité sur le dépôt, GitHub **désactive** le workflow
   planifié : l'arrêt devient définitif et silencieux.

Cas plus sournois : l'API répond 200 mais avec des heures `null`. Les jours
partiels sont moyennés sur les heures restantes (signalé seulement en console), et
si un jour entier est nul il devient un trou signalé. Si **tout** est nul,
`load_series()` renvoie une liste vide, `clean_series()` plante sur `rows[0]`, et on
retombe dans le cas 2. Le pipeline échoue donc proprement, mais le dashboard n'a
aucun contrôle de fraîcheur : j'ajouterais un bandeau quand `generated_at` a plus de
8 jours.

## 6. Stage 5 — Combien de jours je ferais confiance à la projection ?

**7 jours pour l'ordre de grandeur, et pas plus de 2 semaines. La direction « en
hausse », jusqu'à début décembre au maximum.** Mes arguments, tous tirés de mes
propres chiffres :

- **Le bruit écrase la pente.** Sur 7 jours la droite monte de 1,2 ppm
  (450,19 → 451,18), mais l'écart-type des valeurs autour de la droite est de
  **6,1 ppm** et un épisode météo fait +20 ppm en deux jours. La projection ne dit pas
  « 451,18 ppm le 16 novembre » mais « autour de 450 ± 12 ppm ». Elle part même
  6 ppm sous la moyenne des 7 derniers jours (456,6), parce qu'elle est ancrée sur
  les 90 jours et pas sur la semaine récente.
- **Backtest.** J'ai refait la projection en cachant les 7 derniers jours : erreur
  absolue moyenne de **9,8 ppm**, contre 9,2 ppm pour « demain = dernière valeur ».
  À 7 jours, la droite ne fait pas mieux qu'une prévision naïve. Sur 14 à 28 jours,
  elle devient légèrement meilleure (5,3 à 7,7 ppm contre 6,2 à 9,2) : la tendance
  saisonnière est réelle, mais petite face au bruit.
- **La pente dépend de la fenêtre choisie.** +0,165 sur 90 jours, +0,54 sur 30 jours.
  L'écart (≈ 0,375 ppm/jour) dépasse l'écart-type du bruit au bout d'environ
  **16 jours** : au-delà de 2 semaines, la projection reflète surtout mon choix de
  fenêtre, pas les données.
- **Le processus va changer de régime.** L'historique de 15 mois montre le maximum
  saisonnier en décembre (454,1 ppm en décembre 2025, puis 450,5 en janvier). Une
  droite qui monte indéfiniment est donc certainement fausse vers fin décembre.
  Extrapolée sur un an, elle donnerait +60 ppm, alors que la vraie hausse d'une année
  sur l'autre est d'environ +2,5 à +3,3 ppm (octobre 441,4 → 443,9 ; août 434,8 →
  438,1). Plus de 95 % de la pente mesurée est donc saisonnière.

## 7. Stage 6 — Pourquoi le pipeline a continué de marcher en changeant de lieu ?

J'ai remplacé `LAT, LON` par Sydney (-33.8688, 151.2093) dans un commit d'une seule
ligne. Le nettoyage, les lissages, la projection et le dashboard ont fonctionné sans
autre modification : 90 jours, 0 trou, pente **−0,075 ppm/jour** (contre +0,165 au
Mans). Le signe opposé correspond au printemps austral, et le cycle est plus faible
(≈ 8 ppm d'amplitude annuelle à Sydney contre ≈ 20 au Mans : moins de terres et de
végétation dans l'hémisphère Sud).

Pourquoi ça marche : les fonctions de `analyze_series.py` (`clean_series`,
`rolling_mean`, `exponential_smoothing`, `linear_trend_projection`) ne connaissent
**que des dates et des nombres**. Aucune ne mentionne le lieu, les coordonnées, le
nom de la variable (`carbon_dioxide`) ou la physique du CO2. `load_series()` lit une
colonne et la renomme en `value` ; tout ce qui suit travaille sur des `{date, value}`
génériques. Le contrat entre les deux scripts est le format de `series.csv`, pas son
contenu.

Ce qui n'a **pas** suivi, ce sont les textes qui contenaient des hypothèses sur le
lieu. La console affichait « for Le Mans (-33.8688, 151.2093) ». `index.html` disait
« for Le Mans ». Et surtout, la note d'honnêteté parlait de « 90 summer/autumn days »
et de la saison de croissance de l'hémisphère Nord : faux à Sydney, où la fenêtre va
de la fin de l'hiver au printemps. Le calcul est générique, mais pas le discours qui
l'accompagne. J'ai corrigé ces textes dans un second commit (`PLACE` dans
`fetch_series.py`, libellés du dashboard, note rendue neutre sur l'hémisphère et
datée avec la vraie fenêtre) sans toucher à aucune fonction de calcul.
