# Bonnes Pratiques pour Rester à Jour avec le Projet Upstream

Maintenir votre **fork RAGFlow** synchronisé avec le dépôt officiel tout en conservant vos personnalisations (SSO, i18n, RBAC, etc.) nécessite une discipline de développement et d'exploitation. Ce guide propose des principes et un workflow type.

---

## 1. Stratégie Git

| Action | Commandes clés | Fréquence | Conseils |
|--------|----------------|-----------|----------|
| **Ajouter le remote upstream** | `git remote add upstream https://github.com/datafuse-ai/ragflow.git` | Une fois | Conserver `origin` pour votre fork, `upstream` pour la source. |
| **Récupérer les nouveautés** | `git fetch upstream` | Hebdomadaire | Automatisez via un script ou GitHub Action. |
| **Rebaser votre branche principale** | `git checkout main`<br>`git rebase upstream/main` | Hebdomadaire | Préférez le **rebase** pour garder l'historique linéaire. |
| **Résoudre les conflits** | Éditez, testez, `git add`, puis `git rebase --continue` | Au besoin | Utilisez les tests automatisés pour valider. |
| **Mettre à jour vos branches de features** | `git rebase main feature/xxx` | Après chaque MAJ de `main` | Limite la dérive du code. |
| **Push forcé après rebase** | `git push -f origin main` | Après rebase | Utilisez des branches protégées si travail en équipe. |

### Astuce

- Activez un **GitHub Action** qui ouvre automatiquement un PR *upstream-sync* vers votre `main` lorsque `upstream/main` avance.

---

## 2. Isolation et Modularité du Code

1. **Éviter de modifier le core**:  
   -  Placez vos modules dans des paquets dédiés (`ragflow_custom/`) importés par la configuration.
   -  Exploitez les **plugins** existants (`plugin/embedded_plugins`) pour étendre les LLM tools.

2. **Configuration par Environnement**:  
   -  Centralisez vos overrides dans `conf/*.json` ou des variables d'environnement Docker, **pas** dans le code.

3. **Feature Flags**:  
   -  Utilisez un flag (ex: `ENABLE_SSO=true`) pour activer / désactiver chaque évolution.  
   -  Permet un rollback rapide si l'upgrade upstream casse une fonctionnalité.

---

## 3. Tests et CI/CD

1. **Couverture minimale** : Ajoutez des tests unitaires pour chaque évolution (RBAC, SSO, etc.).
2. **Pipeline CI** :  
   -  Exécutez `pytest`, `ruff`, `mypy` et `npm run test` pour le frontend.  
   -  Automatisez la construction des images Docker pour valider les `Dockerfile`.
3. **Environnements de Staging** :  
   -  Déployez sur un cluster de test avant la prod.  
   -  Rejouez un jeu de tests end-to-end (Cypress/Playwright).

---

## 4. Processus d'Upgrade Semestriel

1. **Geler la prod** : Activez un freeze (pas de merge).  
2. **Créer une branche `upgrade/YYYY-MM`** : Travaillez dessus pour intégrer `upstream/main`.  
3. **Rebaser, résoudre, tester**.  
4. **Mettre à jour la documentation** : Notez toute rupture (CHANGELOG).  
5. **Merge & Tag** : Fusionnez dans `main`, taguez `vX.Y.Z+company`.  
6. **Déployer en canari** : 5-10 % des utilisateurs pendant 24 h.

---

## 5. Gestion des Conflits Courants

| Conflit | Solution recommandée |
|---------|----------------------|
| **requirements.txt / package.json** | Fiez-vous à la version upstream puis ré-ajoutez vos dépendances. |
| **Schemas Peewee** | Mettez vos champs dans des modèles séparés via `ForeignKey`, évitez de toucher aux modèles core. |
| **Fichiers de traduction** | Ajoutez vos clés dans `locales/fr/*.json`; upstream ajoute rarement du FR. |
| **Routes Flask** | Regroupez vos routes custom dans un _Blueprint_ distinct (`api/apps/custom_app.py`). |

---

## 6. Observabilité & Rétrocompatibilité

- **Alerting**: Mettez une alerte Prometheus sur les codes 5xx après chaque upgrade.  
- **Dashboards de Régression**: Créez un dashboard Grafana « Upgrade Metrics » (latence, erreurs, coût LLM).  
- **Audit Log**: Vérifiez que les actions critiques continuent d'être journalisées après fusion.

---

## 7. Documentation Vivante

- **CHANGELOG interne** : Tenez un fichier `CHANGELOG_CORP.md` listant vos patchs.  
- **ADR** (Architectural Decision Records) : Documentez chaque évolution dans `docs/adr/YYYYMMDD-feature.md`.

---

## 8. Contribution Upstream (Optionnel)

- **Open-Source Mindset** : Si une feature est d'intérêt général (ex. RBAC basique), envisagez un PR upstream :  
  -  Réduction des conflits futurs.  
  -  Maintien gratuit par les mainteneurs.

---

## 9. Calendrier Recommandé

| Fréquence | Action |
|-----------|--------|
| **Hebdomadaire** | `git fetch upstream && git rebase upstream/main` + exécution des tests. |
| **Trimestrielle** | Revue des dépendances (`pip-audit`, `npm audit`), test de restauration des backups. |
| **Semestrielle** | Upgrade majeur upstream + campagne de tests de non-régression. |

---

## 10. Risques & Mitigations

| Risque | Mitigation |
|--------|-----------|
| Perte de patchs locaux lors d'un rebase | Maintenir une branche `archive/legacy` contenant l'historique avant rebase. |
| Incompatibilité de la DB après upgrade | Utiliser les migrations idempotentes, backups complets avant upgrade. |
| Régression fonctionnelle | Tests automatisés + canari + rollback simple via « feature flag ». |

---

En appliquant ces pratiques, vous maximisez la maintainabilité de votre fork tout en bénéficiant des évolutions rapides de RAGFlow. Un workflow régulier, des tests solides et une bonne séparation du code garantissent des mises à jour sereines. Good hacking ! 🍋 