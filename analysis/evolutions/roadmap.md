# Feuille de Route & Estimation des Évolutions

Ce document synthétise la charge prévisionnelle et le séquencement recommandé pour les travaux décrits dans le dossier `evolutions`.

| Évolution | Domaine | Spéc. | Charge | Impact Upstream |
|-----------|---------|-------|--------|-----------------|
| Observabilité (Logs & Métriques) | Opérations | `observability_strategy.md` | 5 j.h | Faible |
| Sauvegarde & Restauration | Opérations | `backup_restore_strategy.md` | 2 j.h | Nul |
| Stratégie de Mises à Jour | Maintenance | `update_strategy.md` | 1 j.h | Nul |
| Sécurisation et Durcissement | Sécurité | `security_hardening.md` | 5 j.h | Faible |
| Contrôle d'Accès (RBAC) | Sécurité | `rbac_strategy.md` | 4 j.h | Moyen |
| Piste d'Audit (Audit Trail) | Opérations/Séc. | `observability_strategy.md` | 3 j.h | Moyen |
| SSO Microsoft Entra ID | Identité | `sso_strategy.md` | 4 j.h | Moyen |
| Internationalisation (i18n) | UI/UX | `i18n_strategy.md` | 5 j.h | Élevé |
| Thématisation & Marque Blanche | UI/UX | `theming_strategy.md` | 3 j.h | Élevé |
| **Total Estimé** | | | **32 j.h** | |

## Phasage Recommandé (Optimisé pour la Maintainabilité Upstream)

### Phase 0 : Pré-Maintenance (3 jours)

1. **Stratégie de Mises à Jour (1j)** : Mettre en place workflow Git & CI.
2. **Sauvegarde & Restauration (2j)** : Garantir la reversibilité avant toute modif.

### Phase 1 : Observabilité & Sécurité Baseline (10 jours)

1. **Observabilité (5j)** : Ajout non intrusif via sidecars & instrumentation.
2. **Sécurisation & Durcissement (5j)** : Configuration serveur, rate-limit, etc.

### Phase 2 : Gouvernance Applicative (7 jours)

1. **Contrôle d'Accès RBAC (4j)** : Ajout de rôles modulaires.
2. **Piste d'Audit (3j)** : Journaliser les actions critiques.

### Phase 3 : Intégration Entreprise (4 jours)

1. **SSO Microsoft (4j)** : Authentification fédérée, peu de conflits core.

### Phase 4 : Expérience Utilisateur (8 jours)

1. **Internationalisation FR (5j)** : Risque de conflits UI, planifié tard.
2. **Thématisation & Marque Blanche (3j)** : Custom UI, haut risque de merge.

## Risques & Attentions particulières

- **Hardening** : migration Gunicorn simple mais critique ; bien tester uploads après nouvelles limites.
- **SSO** : veillez au stockage secret, au paramètre `state` et au contrôle de redirection.
- **Observabilité** : anonymiser PII dans les logs ; protéger `/metrics`.
- **i18n** : prévenir XSS dans les fichiers de traduction.
- **Thématisation** : valider le type MIME des logos SVG/PNG.

Ce calendrier suppose une seule ressource. Avec deux développeurs (frontend / backend) le planning peut être parallélisé : Hardening + Observabilité en parallèle, puis SSO + i18n, ce qui ramènerait l'effort à ~5 semaines calendaires. 