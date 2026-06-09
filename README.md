# ALGO — Système de trading algorithmique génétique

Système de trading algorithmique basé sur l'évolution génétique, conçu pour apprendre des patterns de marché sur données historiques et déployer des stratégies en production.

> **Statut** : En développement actif — architecture complète, backtesting validé, déploiement live en cours.

---

## Concept

L'idée centrale : plutôt que de coder des règles de trading à la main, laisser une population de stratégies évoluer par sélection naturelle sur des données historiques réelles.

Chaque "gène" est une stratégie de trading complète avec son propre ADN (paramètres d'entrée, de sortie, filtres de qualité). Les meilleurs survivent, se croisent et mutent — les mauvais sont éliminés. Après des milliers de générations, les gènes promus en production ont démontré leur robustesse sur un split train/test chronologique strict.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Agent MTF (Météo)                     │
│  Analyse multi-timeframe 1M/1W/1D/4H/15m                │
│  Produit des vecteurs contextuels : phase, signal,       │
│  position dans la boîte, solidité de tendance           │
│  Polarité H (bull) et B (bear miroir)                   │
└─────────────────────┬───────────────────────────────────┘
                      │ vecteurs contextuels
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  Gene Trainer                            │
│                                                          │
│  Coordinator (seul writer disque)                       │
│      ├── Worker 1 ──┐                                   │
│      ├── Worker 2 ──┤── GeneTrainer (évolution)         │
│      └── Worker N ──┘                                   │
│                                                          │
│  guided_depth : curseur chaos → exploitation            │
│  0 = chaos pur     3 = calibration percentiles          │
│  1 = stratégie/phase  4 = exploitation pépite           │
└─────────────────────┬───────────────────────────────────┘
                      │ gènes promus
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  Patrimoine génétique                    │
│                                                          │
│  Or (score ≥ 0.55) · Argent · Bronze                   │
│  Empreinte comportementale (axe X/Y/Z)                  │
│  Détection de clones par overlap temporel               │
│  Généalogie complète de chaque gène                     │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              Dashboard (patrimoine.html)                 │
│  Vue globale · Heatmap · Runs · HOF · Entraîner         │
│  Filtres : niveau / asset / polarité H∕B                │
│  Commande d'entraînement générée en temps réel          │
└─────────────────────────────────────────────────────────┘
```

---

## Composants principaux

### Agent MTF — Contexte de marché
Analyse le marché sur 5 timeframes simultanément et produit pour chaque bougie 15m un vecteur contextuel structuré :

- **Phase** 
- **Fenêtre** : LARGE / NORMALE / FERMEE
- **Horizon** : LONG / MOYEN / COURT
- **Signal** : direction, score d'alignement MTF, solidité 1D/1W
- **position_Y** : position dans la boîte 4H (0 = bas, 1 = haut)
- **Polarité B** : inversion miroir complète pour trader les deux sens

### Gene Trainer — Évolution génétique
Architecture multi-worker avec coordinator comme seul writer disque pour éviter les race conditions.

**ADN d'un gène** :
```python
{
  "params_techniques": {
    "ema_rapide": 9, "ema_lente": 21,
    "rsi_periode": 14, "atr_mult_stop": 2.1, ...
  },
  "logique_entree": {
    "strategie": "momentum",          # momentum / rebond / breakout /
                                      # croisement_ma / rsi_extreme
    "filtre_direction_1d": True,
    "confirmation_bougies": 2, ...
  },
  "logique_sortie": {
    "strategie": "trailing_atr",      # trailing_atr / mixte / cible_fixe / timeout
    "cible_rr": 3.5,
    "sovereign_trailing": 1.8, ...
  },
  "filtres_qualite": {
    "score_signal_min": 0.25,
    "autorisation_min": "RESTREINT", ...
  }
}
```

**Score composite** :
```
profit_factor,  +  sharpe   +  wr_pondéré + couverture   +  pénalité_dd +  réactivité 
```

**Logique de promotion** :
```
score ≥ 0.55                          → Or (promu directement)
0.35 ≤ score < 0.55                   → check empreinte doublon
  même stratégie + overlap > 70%      → refusé (clone)
  stratégie différente                → Argent / Bronze
score < 0.35                          → refusé
```

### Gene Coherence — Moteur chaos/guidé
Curseur `guided_depth` qui contrôle l'espace de recherche :

| depth | Comportement |
|-------|-------------|
| 0 | *** |
| 1 | Stratégie fixée selon la phase MTF, reste *** |
| 2 | + Canaries de la stratégie gelés |
| 3 | + Calibration depuis percentiles historiques (standard) |
| 4 | Exploitation d'une pépite existante, mutation douce |

Les **canaries** sont les paramètres critiques d'une stratégie (ex: `ema_rapide/lente` pour `croisement_ma`) — les toucher aveugle le signal d'entrée. Ils sont gelés uniquement en exploitation (depth ≥ 3).

### Empreinte comportementale
Chaque gène promu génère une empreinte 3D qui caractérise son comportement :

- **Axe X** : distribution temporelle des trades (timestamps, couverture)
- **Axe Y** : stratégies utilisées, raisons de sortie
- **Axe Z** : zones d'activité selon `position_Y` (début / milieu / fin de boîte)

Utilisée pour détecter les clones et enrichir le patrimoine en diversité réelle.

### Polarité B — Miroir bear
Le système entraîne deux populations de gènes en parallèle :
- **H** (bull) : sur les vecteurs MTF normaux
- **B** (bear) : sur les vecteurs MTF inversés (EXPANSION↔CONTRACTION, LONG↔SHORT, BULL↔BEAR, position_Y → 1-Y)

L'inversion est appliquée à un seul endroit (`_inverser_label_contexte()`) — le reste du pipeline est identique. Les gènes B tradent les mouvements baissiers avec la même rigueur que les H tradent les mouvements haussiers.

---

## Stack technique

- **Python 3.11+** — core, backtest, évolution génétique
- **Pandas / NumPy** — traitement données OHLCV
- **Multiprocessing** — workers parallèles avec write queue
- **HTML/JS vanilla** — dashboard temps réel (aucune dépendance frontend)
- **JSON** — persistance légère, pas de base de données

---

## Structure du projet

```
project/
├── gene_trainer/           # Package entraînement multi-worker
│   ├── __main__.py         # CLI : --tous --pool --polarite --guided-depth
│   ├── coordinator.py      # Seul writer disque
│   ├── worker.py           # Logique évolution
│   ├── data_source.py      # Sources OHLCV (fichier / live / polarisé)
│   └── gene_trainer_cfg.py # Hyperparamètres
├── gene_trader.py          # ADN + backtest d'un gène
├── gene_trainer_core.py    # GeneTrainer : arène + boucle évolution
├── gene_coherence.py       # Moteur chaos/guidé + canaries
├── gene_contexte_mtf.py    # Agent météo MTF
├── gene_empreinte.py       # Empreinte comportementale 3D
├── gene_registre.py        # Métadata + généalogie
├── asset_profile.py        # Profil statistique par asset
├── exporter_dashboard.py   # Construit patrimoine.json
├── dashboard.html          # Interface visuelle
└── genes/
    ├── metadata/           # Historique de chaque gène
    ├── production/         # Gènes actifs (actif_CONTEXTE.json)
    └── empreintes/         # Empreintes comportementales
```

---

## Utilisation

```bash
# Entraînement standard
python -m gene_trainer --tous --pool contexte_mtf_btc.json --mode C --workers 4

# Mode exploration chaos (trouver de nouvelles pépites)
python -m gene_trainer --tous --pool contexte_mtf_btc.json \
  --explorer --force-promo --depuis-zero --guided-depth 1

# Mode bear miroir
python -m gene_trainer --tous --pool contexte_mtf_btc_b.json \
  --polarite B --mode C --workers 4

# Run ciblé sur un contexte spécifique
python gene_lab.py --contexte EXPANSION_LARGE_LONG \
  --generations 30 --mutation 0.45 --arene 20

# Boucle entraînement continu (H + B en alternance)
boucle_entrainement.bat

# Dashboard
python exporter_dashboard.py
# → http://localhost:8080/dashboard.html
```

---

## Ce qui est en cours

- Déploiement live (orchestrateur H+B en parallèle)
- Paper trading pour valider les gènes en conditions réelles
- Support multi-asset étendu (ETH, SOL)
- Sous-score unicité génomique

---

*Projet personnel — conçu et développé de A à Z sans formation formelle.*
