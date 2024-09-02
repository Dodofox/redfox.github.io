---
title: "Créer des machines sans PXE ni Compute ressource sur OSP via satellite"
slug: "Créer des machines sans PXE ni Compute Ressource sur OpenStack via Satellite"
date: "2024-01-01 00:00:00+0000"
weight: 1
tags: 
   - déploiement
   - satellite
   - openstack
categories: 
   - Red Hat
---

# Créer des machines sans PXE ni Compute Ressource sur OpenStack via Satellite

## Introduction

Dans mon quotidien de sysadmin, je passe pas mal de temps sur mon lab qui tourne exclusivement sous OpenStack, que je peux reconstruire à volonté grâce à Ansible (je sens qu’un article sur ce sujet se profile à l’horizon). Récemment, j'ai dû relever un défi intéressant : tester le provisioning sans PXE avec Satellite, mais sans utiliser de compute ressource. Comme tout bon ingénieur système qui se respecte, j'ai préféré tester cela sur mon lab existant plutôt que de créer un environnement plus approprié, même si, soyons honnêtes, cela m'aurait probablement fait gagner du temps.

Je me suis dit que ce serait intéressant de partager cette aventure, non seulement parce que ça a piqué ma curiosité et pris pas mal de mon temps, mais aussi parce que cette méthode de provisioning est assez peu utilisée avec Satellite.

## Description de l'environnement

Comme j'aime bien être à jour dans mes articles, je précise les versions des technologies utilisées, même si certaines choses échappent à mon contrôle (merci les mises à jour incessantes).

- **Red Hat OpenStack 17.1**
- **Red Hat Satellite 6.15** (sous RHEL 8 dernier cri)
- **Création d'un serveur RHEL 9.4**

## Procédure

### Préparation de Satellite

Si vous partez d'une installation toute fraîche de Satellite, il y a quelques étapes indispensables à ne pas manquer. Si votre environnement est déjà configuré, certains éléments pourraient être déjà en place.

- **Activer le HTTPBoot sur Satellite**

  ```bash
  satellite-installer --foreman-proxy-http true --foreman-proxy-httpboot true
  ```

  Petit rappel : les options non spécifiées ici restent inchangées, donc pas besoin de lister tout ce qui a été modifié depuis la création du cluster.

- **Produits**

  Il est impératif d'avoir les repositories BaseOS et AppStream, ainsi que les repositories kickstart correspondants.

  Dans *Content -> Red Hat Repositories*, activez-les, puis dans *Content -> Product*, synchronisez-les (un plan de synchronisation rendra les choses encore plus fluides).

- **Content Views (CV)**

  Pour chaque OS, il vous faudra une CV ou une CCV (Composite Content View) contenant les 4 repos et publiées sur les lifecycles adéquats.

- **Domaine et Sous-réseau**

  Cette étape est cruciale : pour que tout fonctionne, il faut associer une adresse IP et un domaine à votre futur hôte. Assurez-vous que ces éléments sont bien configurés.

  Pour le domaine : *Infrastructure -> Domains*, puis *Create Domain*. Spécifiez le nom de domaine DNS et n’oubliez pas de l’assigner aux "locations" et "organisations" nécessaires.

  Pour le sous-réseau : *Infrastructure -> Subnets*, puis *Create Subnet*. Remplissez les champs obligatoires et n'oubliez pas d'ajouter un serveur DNS primaire (il sera utilisé pour résoudre le nom du Satellite et/ou de la capsule), mode de démarrage "Static".

- **Provisioning Template**

  Ici aussi, c'est une étape importante. Vous pouvez utiliser vos propres templates de provisioning, ou partir du template par défaut (kickstart default), mais assurez-vous que la commande "reboot" ne soit pas présente.

- **Host Group**

  Pour créer ou modifier un Host Group : *Configure -> Host Groups*.

  Dans les onglets "Operating System" et "Activation Keys", je vais entrer dans les détails, pour le reste, suivez les pratiques habituelles.

  Dans *Operating System*, choisissez l'OS correspondant à la version des repos kickstart synchronisés, gardez les media sur Synced Content, et vérifiez qu'il utilise bien le repo BaseOS kickstart de la version cible. Pensez aussi à spécifier le mot de passe root si vous en utilisez un générique, sinon, n'oubliez pas de le créer lors d'une étape ultérieure.

  Dans *Activation Keys*, spécifiez bien celle qui permettra l'association à la CV/CCV.

- **Operating System (OS)**

  Assurez-vous que l'OS que vous souhaitez utiliser est bien associé au bon provisioning template.

  Pour ce faire, allez dans *Hosts -> Provisioning Setup -> Operating System*, choisissez l'OS souhaité, et dans l'onglet "Templates", sélectionnez le bon template à la ligne "Provisioning Template" (celui spécifié précédemment).

### Préparation d’OpenStack (OSP)

Rassurez-vous, le plus dur est fait. Côté OSP, il y a aussi quelques préparations à faire.

Je pars du principe que vous avez déjà ce qu’il faut en termes de projet (réseau, sous-réseau, flavor, etc.).

- **Préparation d’un port réseau**

  Satellite ayant besoin de spécifier une adresse MAC lors de la création d’un hôte, nous allons préparer un port et lui attribuer une adresse IP.

  ```bash
  openstack port create --network <nom du réseau> --fixed-ip subnet=<nom du sous-réseau>,ip-address=<IPv4 choisie> <nom de l'interface>
  ```

  Pensez à récupérer l'adresse MAC de la carte réseau fraîchement créée :

  ```bash
  openstack port show <nom de l'interface> | grep "mac_address"
  ```

### Provisioning de l’hôte

Enfin, l'étape tant attendue !

- **Création d’un hôte sous Satellite**

  Créez un nouvel hôte sous Satellite :

  *Hosts -> Create Host*

  Spécifiez les éléments suivants :

  - **Name** : le nom d'hôte du futur serveur
  - **Organisation & Location**
  - **Host Group** : celui spécifié précédemment (cela remplira automatiquement les autres champs de cet onglet et de "Operating System")
  - Dans *Interface*, cliquez sur *Edit* de l'interface existante, puis :
    - Mentionnez l'adresse MAC récupérée précédemment
    - Spécifiez le domaine ainsi que le sous-réseau
    - Mentionnez l'adresse IPv4 utilisée pour créer le port sous OpenStack

  L'hôte est maintenant créé côté Satellite, et il devrait être en statut "Pending Installation". C'est précisément ce statut qui nous permet de récupérer l'étape clé de ce provisioning : la Full Host Image.

  Pour cela, sur la page de l'hôte (si vous n’avez pas cédé à la tentation de changer d'onglet), cliquez en haut à droite sur les trois petits points, puis sur "Full host '<hostname>' image".

  Vous téléchargez alors une image ISO contenant toutes les informations nécessaires pour builder cet hôte. Pas de panique, il ne s'agit en réalité que d'une image "discover foreman" avec des informations surchargées, comme le nom, l'adresse IP, et le fichier kickstart à récupérer. Le poids de cette image ne devrait pas dépasser la centaine de Mo.

- **Import de l’image côté OpenStack**

  Maintenant que vous avez cet ISO, sous OpenStack, vous pouvez créer l'image associée :

  ```bash
  openstack image create --disk-format iso --file <nom de l'image>
  ```

- **Lancement du provisioning**

  Tout est prêt, il ne reste plus qu’à lancer l’installation de l’instance. Comme nous allons un peu à l’encontre des principes de base d’OSP, voici comment procéder :

  1. Créez une instance qui démarrera sur l’image, à laquelle vous allez associer un volume non temporaire.
  2. Une fois l'installation terminée (en attente de reboot), supprimez cette instance (le volume ne sera pas supprimé).
  3. Créez une nouvelle instance qui démarrera sur le volume fraîchement installé.

Et voilà ! Vous avez maintenant une instance provisionnée par votre Satellite.

## Conclusion

Dans une démarche de reproductibilité, j’ai pu utiliser le système de provisioning sans PXE de Satellite en exploitant les "Full Host Images". Une fois la partie préparation terminée, on peut même envisager d’automatiser la création de l’hôte via un playbook Ansible, en prenant en entrée les informations clés comme l’OS, le hostname, l’IPv4, etc.