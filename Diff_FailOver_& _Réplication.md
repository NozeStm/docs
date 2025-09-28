# 📚 Cours : Différence entre **Failover** et **Réplication**

## 1. Introduction
Dans les systèmes informatiques (bases de données, serveurs, réseaux…), il est essentiel d’assurer la **haute disponibilité** et la **continuité de service**.  
Deux notions reviennent souvent : **Failover** et **Réplication**.  
Elles sont complémentaires mais ne signifient pas la même chose.

---

## 2. Qu’est-ce que le **Failover** ?
👉 Le **Failover** (basculement automatique) est un mécanisme qui permet de **continuer le service** lorsqu’un système tombe en panne.  
- Si le serveur principal (ou la base de données principale) **échoue**, un autre serveur prend automatiquement le relais.  
- L’objectif est de **minimiser le temps d’arrêt**.  

### Exemple concret :
- Tu as deux serveurs : **Serveur A** (actif) et **Serveur B** (passif).  
- Si A plante (panne matérielle, crash logiciel…), le système **bascule automatiquement** vers B.  
- L’utilisateur final ne voit presque pas la coupure.  

⚠️ Important :  
- Le failover ne garantit pas toujours que les données sont parfaitement à jour sur le serveur de secours (ça dépend de la stratégie de réplication utilisée).

---

## 3. Qu’est-ce que la **Réplication** ?
👉 La **Réplication** consiste à **copier les données** d’un système vers un autre, en temps réel ou différé.  
- Le but est d’avoir plusieurs copies cohérentes (ou presque cohérentes) des données.  
- Elle peut être **synchrone** (copie immédiate) ou **asynchrone** (copie avec un petit retard).  

### Exemple concret :
- Une base de données MySQL a une **instance principale** (Master) et une ou plusieurs **répliques** (Slaves).  
- Quand tu écris une donnée sur le Master, elle est **répliquée** vers les Slaves.  
- Les Slaves peuvent être utilisés pour :  
  - soulager la charge de lecture,  
  - servir de secours en cas de panne.  

---

## 4. Différence essentielle
| Critère | Failover | Réplication |
|---------|----------|-------------|
| **But principal** | Assurer la **continuité de service** quand un système tombe | Assurer la **disponibilité et l’intégrité des données** sur plusieurs systèmes |
| **Action** | Basculement automatique vers un autre serveur/machine | Copie continue (ou planifiée) des données |
| **Temps d’impact** | Se déclenche seulement **en cas de panne** | Fonctionne **en permanence** |
| **Exemple** | Un site web passe d’un serveur à un autre après crash | Une base de données copie en temps réel ses transactions sur une autre instance |

---

## 5. Complémentarité
En pratique, on combine souvent **les deux** :
- La **réplication** assure que les données existent sur plusieurs serveurs.  
- Le **failover** permet de basculer automatiquement vers un serveur répliqué si le principal tombe.  

⚡ Exemple réel :  
- Dans une architecture PostgreSQL avec **Patroni** ou MySQL avec **MHA**, la réplication est mise en place entre plusieurs serveurs.  
- Si le **Master** tombe, le système fait un **failover** et élit une nouvelle instance répliquée comme Master.  

---

✅ En résumé :
- **Failover** = mécanisme de bascule (haute disponibilité).  
- **Réplication** = mécanisme de duplication des données (résilience des données).  
- Ensemble, ils forment la base de l’**haute disponibilité** et de la **tolérance aux pannes**.  
