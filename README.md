# EtherShield

Projet Blockchain & Cybersécurité : laboratoire de sécurité pour smart contracts Ethereum.

On écrit des contrats Solidity avec des failles volontaires, on les attaque avec des tests Foundry, puis on les corrige. Tout se passe sur un réseau local (Anvil) avec de faux ETH.

## Les 6 failles

| Faille | Contrat vulnérable | Correction | Fichier |
|--------|--------------------|------------|---------|
| Reentrancy | VulnerableVault | Checks-Effects-Interactions + ReentrancyGuard | src/Reentrancy.sol |
| Access Control | VulnerableTreasury | AccessControl (rôle MANAGER) + limite sur le fee | src/AccessControlDemo.sol |
| tx.origin | VulnerableWallet | msg.sender à la place de tx.origin | src/TxOrigin.sol |
| Signature Replay | VulnerableSigVault | nonce + deadline + adresse du contrat + chainid | src/SignatureReplay.sol |
| Front-running | VulnerablePuzzle | commit-reveal | src/FrontRunning.sol |
| Denial of Service | VulnerableAuction | pull payment (refunds) | src/DenialOfService.sol |

Chaque fichier de `src/` contient le contrat vulnérable et le contrat corrigé.
Chaque fichier de `test/` contient l'attaque sur le contrat vulnérable, puis la même attaque sur le contrat corrigé (qui échoue).

## Installation

Il faut installer Foundry : https://book.getfoundry.sh/getting-started/installation

```
git clone <lien-du-repo>
cd ethershield
forge install foundry-rs/forge-std OpenZeppelin/openzeppelin-contracts@v5.0.2
forge build
```

## Lancer les tests

```
forge test
```

Pour voir plus de détails :

```
forge test -vvv
```

Un seul fichier :

```
forge test --match-path test/Reentrancy.t.sol -vvv
```

Résultat : 27 tests, tous passent.

## Déploiement sur Anvil

Terminal 1 :

```
anvil
```

Terminal 2 :

```
forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --broadcast --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

La clé utilisée est la clé de test d'Anvil (publique, sans valeur réelle).

## Démonstration

1. Lancer Anvil et déployer le contrat (voir au-dessus)
2. Lancer l'attaque : `forge test --match-test test_attack_on_vulnerable_vault -vvv`
3. Expliquer la cause et l'impact (voir rapport.md)
4. Montrer que la correction bloque l'attaque : `forge test --match-test test_attack_on_fixed_vault_fails -vvv`
5. Lancer tous les tests : `forge test`
6. Montrer Slither : `slither . --filter-paths "lib|test|script"`

## Slither

```
pip install slither-analyzer
slither . --filter-paths "lib|test|script"
```

Le résultat de notre analyse est dans `slither.txt` et expliqué dans `rapport.md`.

## Règles de sécurité

- Tests uniquement sur nos propres contrats et sur Anvil
- Aucune vraie clé privée, aucun vrai argent
- Ne jamais déployer les contrats vulnérables sur un vrai réseau
