# Rapport d'audit EtherShield

Outils : Foundry (Forge, Anvil), Solidity 0.8.24, OpenZeppelin 5.0.2, Slither
Réseau : local uniquement

## 1. Résumé

| ID | Faille | Gravité | Corrigée |
|----|--------|---------|----------|
| V1 | Reentrancy | Critique | oui |
| V2 | Access Control | Critique | oui |
| V3 | tx.origin | Haute | oui |
| V4 | Signature Replay | Haute | oui |
| V5 | Front-running | Moyenne | oui |
| V6 | Denial of Service | Haute | oui |

Chaque faille est reproduite par un test qui réussit sur le contrat vulnérable et qui échoue sur le contrat corrigé.

## 2. Détail des failles

### V1 - Reentrancy (Critique)

- **Contrat** : VulnerableVault, fonction `withdraw()`
- **Cause** : le contrat envoie les ETH avant de mettre le solde à 0. Le contrat de l'attaquant rappelle `withdraw()` dans sa fonction `receive()`.
- **Impact** : l'attaquant dépose 1 ETH et vide tout le vault (9 ETH dans le test).
- **Preuve** : `test_attack_on_vulnerable_vault`
- **Correction** : on met le solde à 0 avant l'envoi (Checks-Effects-Interactions) et on ajoute `nonReentrant`.
- **Validation** : `test_attack_on_fixed_vault_fails` (l'attaque est annulée), test fuzz et test invariant (le solde du vault est toujours égal aux dépôts moins les retraits).

### V2 - Access Control (Critique)

- **Contrat** : VulnerableTreasury, fonctions `setOwner()` et `setFee()`
- **Cause** : aucune vérification de qui appelle la fonction.
- **Impact** : n'importe qui devient owner et retire les 10 ETH, ou change le fee.
- **Preuve** : `test_attack_takeover_vulnerable`, `test_attack_change_fee_vulnerable`
- **Correction** : `AccessControl` d'OpenZeppelin avec un rôle MANAGER, et le fee est limité à 10.
- **Validation** : `test_attack_fails_on_fixed`, `testFuzz_fixed_random_user_cannot_set_fee`

### V3 - tx.origin (Haute)

- **Contrat** : VulnerableWallet, fonction `transfer()`
- **Cause** : le contrat vérifie `tx.origin`, qui est la personne qui a signé la transaction, et pas celui qui appelle directement le contrat.
- **Impact** : si le propriétaire appelle un contrat malveillant (phishing), celui-ci vide le wallet.
- **Preuve** : `test_attack_on_vulnerable_wallet`
- **Correction** : on vérifie `msg.sender`.
- **Validation** : `test_attack_on_fixed_wallet_fails`

### V4 - Signature Replay (Haute)

- **Contrat** : VulnerableSigVault, fonction `claim()`
- **Cause** : la signature contient seulement l'adresse et le montant. On peut la réutiliser autant de fois qu'on veut.
- **Impact** : une signature de 1 ETH permet de vider le contrat (3 ETH retirés dans le test).
- **Preuve** : `test_replay_on_vulnerable_vault`
- **Correction** : le message signé contient maintenant un nonce, une date d'expiration, l'adresse du contrat et le chainid. Le nonce augmente après chaque claim.
- **Validation** : rejeu refusé, signature expirée refusée, signature refusée sur un autre contrat et sur une autre chaîne.

### V5 - Front-running (Moyenne)

- **Contrat** : VulnerablePuzzle, fonction `solve()`
- **Cause** : la réponse est visible dans la transaction en attente. Un attaquant la copie et passe avant.
- **Impact** : l'attaquant gagne les 5 ETH, la transaction de la victime échoue.
- **Preuve** : `test_frontrun_vulnerable_puzzle`
- **Correction** : commit-reveal. La victime envoie d'abord un hash lié à son adresse, puis révèle la réponse dans un bloc suivant.
- **Validation** : `test_frontrun_fails_on_fixed_puzzle`, `test_copied_commit_does_not_work`, `test_reveal_in_same_block_fails`

### V6 - Denial of Service (Haute)

- **Contrat** : VulnerableAuction, fonction `bid()`
- **Cause** : le contrat rembourse l'ancien meilleur enchérisseur dans la même fonction. Si ce remboursement échoue, tout échoue.
- **Impact** : un contrat attaquant qui refuse les ETH bloque l'enchère pour toujours.
- **Preuve** : `test_dos_on_vulnerable_auction`
- **Correction** : pull payment. Les remboursements sont stockés dans `refunds` et chacun les retire avec `withdrawRefund()`.
- **Validation** : `test_dos_fails_on_fixed_auction`

## 3. Résultats Slither

Commande : `slither . --filter-paths "lib|test|script"` (23 contrats analysés, 28 résultats, voir `slither.txt`)

| Détecteur | Où | Verdict |
|-----------|----|---------|
| reentrancy-eth | VulnerableVault.withdraw | Vrai positif (V1) |
| reentrancy-eth | VulnerableAuction.bid | Vrai positif partiel : le vrai problème exploitable est le DoS (V6) |
| tx-origin | VulnerableWallet.transfer | Vrai positif (V3) |
| arbitrary-send-eth | VulnerableSigVault, VulnerableWallet | Vrais positifs (V4, V3) |
| arbitrary-send-eth | VulnerablePuzzle | Voulu : le gagnant reçoit la récompense |
| arbitrary-send-eth | FixedTreasury, FixedSigVault | Faux positifs : protégés par un rôle ou une signature |
| timestamp | FixedSigVault.claim | Faux positif : c'est la date d'expiration voulue |
| immutable-states | plusieurs variables | Conseil d'optimisation, pas une faille |
| low-level-calls | contrats avec `.call` | Information : on vérifie bien le retour de l'appel |
| erc20-interface | fonctions `transfer` des wallets | Pas important : ce ne sont pas des tokens ERC20 |
| missing-zero-check | plusieurs adresses | Amélioration possible, pas une faille |

Ce que Slither n'a pas trouvé :

- V2 (Access Control) : pas de détecteur pour des fonctions publiques sans vérification
- V4 (Signature Replay) : c'est un problème de logique
- V5 (Front-running) : dépend de l'ordre des transactions

Conclusion : Slither trouve bien les failles classiques (reentrancy, tx.origin), mais il faut aussi des tests et une relecture manuelle.

## 4. Limites

- Les tests simulent l'ordre des transactions avec `vm.prank` et `vm.roll`. Ce n'est pas un vrai mempool.
- FixedPuzzle : si la réponse est facile à deviner, on peut la retrouver à partir du hash.
- FixedSigVault : la sécurité dépend de la clé privée du signataire.
- FixedTreasury : un seul administrateur. En vrai, il faudrait un multisig.
- FixedAuction : pas de date de fin de l'enchère.
- Pas de vérification formelle.
