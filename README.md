# TD4 — Séries temporelles : collecte, nettoyage, lissage et projection honnête

Pipeline automatisé qui récupère 90 jours de CO2 atmosphérique horaire réel
(Open-Meteo Air Quality API, modèle CAMS), l'agrège en série journalière, détecte
les jours manquants, calcule une moyenne mobile 7 jours, un lissage exponentiel
(α = 0,3) et une projection linéaire à 7 jours, puis publie le tout sur un
tableau de bord GitHub Pages avec une note d'honnêteté sur les limites de la
projection.

- **Dashboard :** https://lodesthard.github.io/td-time-series/
- **Réflexions :** [REFLECTIONS.md](REFLECTIONS.md)

## Structure

```
scripts/fetch_series.py          # API Open-Meteo -> data/series.csv (1 ligne par jour)
scripts/analyze_series.py        # nettoyage, lissages, projection -> docs/data/series_analysis.json
data/series.csv                  # généré et commité par l'Action
docs/index.html                  # dashboard (Chart.js)
docs/data/series_analysis.json   # généré et commité par l'Action
.github/workflows/update_series.yml  # chaque lundi 06:00 UTC + lancement manuel
```

## Résultats (run du 2026-11-10, fenêtre 2026-08-12 → 2026-11-09)

| Lieu | Jours | Trous | Pente | Dernière valeur | Projection J+7 |
|---|---|---|---|---|---|
| Le Mans (48.0061, 0.1996) — étapes 1 à 5 | 90 | 0 | **+0,165 ppm/jour** | 451,3 ppm | 451,18 ppm |
| Sydney (-33.8688, 151.2093) — étape 6 | 90 | 0 | **−0,075 ppm/jour** | 441,1 ppm | 444,06 ppm |

La version Le Mans est conservée dans l'historique : commit
[`e8217cc`](https://github.com/Lodesthard/td-time-series/tree/e8217ccf7e0ad136d6b43adddceb843a9fe03e8a).

## Étape 6 — changement de lieu

Seule la ligne `LAT, LON` de `fetch_series.py` a été modifiée pour faire tourner
le calcul (commit « Stage 6: switch location… »). Un second commit corrige
uniquement des **textes** devenus faux : le nom « Le Mans » codé en dur dans le
message console et dans `index.html`, et la note d'honnêteté qui parlait de
« 90 summer/autumn days » et de l'hémisphère Nord (faux à Sydney, où la fenêtre
couvre la fin de l'hiver et le printemps austral). Détails dans la réflexion 7.

## Relancer

Actions → **Update analysis** → Run workflow. En local :

```bash
pip install requests
python scripts/fetch_series.py
python scripts/analyze_series.py
```
