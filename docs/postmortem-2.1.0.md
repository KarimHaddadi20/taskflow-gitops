# Postmortem — la 2.1.0 répond en erreur sous charge

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 8 octobre 2026, de 13:13:41 à 13:15:13 (UTC+2) |
| Version en cause | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| PR à l'origine | [PR 21](https://github.com/KarimHaddadi20/taskflow-gitops/pull/21), mergée à 13:12:00 |
| Durée d'exposition | 1 min 32 s, du premier pod prêt (13:13:41) au retrait du canary (13:15:13) |
| Part du trafic touché | 25 %, le premier palier du canary |
| Détecté par | AnalysisRun `taskflow-df976ccb5-2-1`, Job k6 sur le Service `taskflow-canary` |
| Résolu par | Abort automatique d'Argo Rollouts à 13:14:56, puis revert Git par la [PR 22](https://github.com/KarimHaddadi20/taskflow-gitops/pull/22) à 13:41:08 |

## Chronologie

| Heure | Événement |
| --- | --- |
| 12:25 | Étalon sur la `2.0.0` : 665 requêtes, 0,00 % d'erreurs, p95 = 34,5 ms |
| 12:28:16 | PR 20 mergée : le test k6 est branché sur le palier 25 % |
| 13:12:00 | PR 21 mergée : l'image voulue passe à `2.1.0` |
| 13:13:41 | Un pod `2.1.0` est prêt. Il reçoit 25 % du trafic |
| 13:13:54 | L'AnalysisRun démarre |
| 13:14:56 | Le Job k6 échoue : 29,20 % d'erreurs (73/250) et p95 = 684,71 ms. Argo Rollouts annule |
| 13:15:13 | Plus aucun pod `2.1.0`. `observe.sh` : 40 réponses `2.0.0` en HTTP 200 |
| 13:41:08 | PR 22 : Git revient à l'image `2.0.0`. L'étape d'analyse reste en place |
| 13:48:06 | Sur la `2.2.0` (PR 23), le second AnalysisRun réussit : 0,00 % d'erreurs, p95 = 60,58 ms |

## Composant défaillant et cause racine

- Le composant en échec est l'application `2.1.0`, sur la route `/tasks`. Preuve : le Job k6, lancé uniquement contre `http://taskflow-canary`, relève 73 réponses non 200 sur 250 et un p95 de 684,71 ms. Les deux seuils (moins de 2 % d'erreurs, p95 sous 250 ms) sont franchis. Le message de l'AnalysisRun est `Metric "test-de-charge-k6" assessed Failed due to failed (1) > failureLimit (0)`.
- Les probes Kubernetes n'ont rien vu parce qu'elles interrogent `/health`, toutes les 5 s pour la readiness et toutes les 10 s pour la liveness. Cette route répondait, donc le pod restait Ready et continuait de recevoir sa part de trafic. Le défaut est sur `/tasks`, la route que le test de charge appelle.
- Cause racine : la `2.1.0` sert `/tasks` trop lentement et avec des erreurs dès qu'elle reçoit du trafic, alors que la `2.0.0` tenait 665 requêtes sans erreur à 34,5 ms de p95. L'étalon du matin donne la comparaison.

## Ce qui a bien fonctionné

L'analyse a tourné sans clic. Entre le pod prêt et l'annulation, 1 min 32 s se sont écoulées, et seulement un quart des requêtes a touché la mauvaise version. Le contrôleur a retiré le pod canary et laissé les 4 pods `2.0.0`. `observe.sh` l'a confirmé juste après.

L'abort ne réécrit pas Git. Tant que la PR 22 n'était pas mergée, l'état voulu restait la `2.1.0`. Le revert remet le dépôt d'accord avec ce que le cluster sert vraiment, et il conserve l'étape d'analyse pour la suite.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Revert de la PR 21, image revenue à `2.0.0`, analyse k6 conservée | Karim Haddadi | Fait, PR 22, 13:41 |
| Redéployer une version qui passe les deux seuils | Karim Haddadi | Fait, PR 23 : la `2.2.0` est Healthy, 4 pods, p95 = 60,58 ms |
| Garder le palier d'analyse devant toute montée de version | Karim Haddadi | En place dans `apps/taskflow/rollout.yaml` |

Le premier essai de la `2.2.0` a échoué à 13:45 avec 0,00 % d'erreurs et un p95 de 293,05 ms. La version répondait correctement : c'est un pic de lenteur du cluster qui a franchi le seuil. La relance de 13:47 a passé les deux seuils. Le seuil de 250 ms reste celui du cours.
