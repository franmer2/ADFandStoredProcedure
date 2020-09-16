# Azure Data Factory et procedure stockée (Azure SQL Database)

Lors d'un projet avec un client, une étape consistait à transformer des fichiers, déposés dans un stockage blob, à l'aide d'une procédure stockée existante, puis de déplacer le résultat dans un stockage "Azure Files".

Cet article a pour but de partager les différentes étapes pour réaliser ce pipeline de transformation et ainsi que les différentes astuces utilisées pour mener à bien cette partie du projet


## Pré requis

- [Un abonnement Azure](https://azure.microsoft.com/fr-fr/free/)
- [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/) 



# Création des services Azure
## Création d'un groupe de ressources
Nous allons commencer par créer un groupe de ressouces afin d'héberger les différents services de notre solution de transcription de fichiers audio.

Depuis le portail [Azure](https://portal.azure.com), cliquez sur "**Create a resource**"

![sparkle](Pictures/001.png)

 Puis, recherchez "**Resource group**"

 ![sparkle](Pictures/002.png)


Cliquez sur le bouton "**Create**"

![sparkle](Pictures/003.png)

Dans le champ "**Resource group**", donnez un nom à votre groupe de ressources. Cliquez sur le bouton "**Review + Create**"

![sparkle](Pictures/004.png)

Dans l'écran de validation, cliquez sur le bouton "**Create**"

![sparkle](Pictures/005.png)

Revenez à l'accueil du portail Azure. Cliquez sur le menu burger en haut à gauche puis sur "**Resource** **groups**"

![sparkle](Pictures/006.png)

Cliquez sur le groupe de ressources créé précédement

![sparkle](Pictures/007.png)

## Création du compte de stockage

Une fois dans le groupe de ressources, cliquez sur le bouton "**Add**"

![sparkle](Pictures/008.png)

Recherchez le compte de stockage

![sparkle](Pictures/009.png)

Cliquez sur le bouton "**Create**"

![sparkle](Pictures/010.png)



Complétez la création du compte de stockage et cliquez sur "**Review** + **create**"

![sparkle](Pictures/011.png)

Après avoir vérifié les informations de création du compte de stockage, cliquez sur le bouton "**Create**"

![sparkle](Pictures/012.png)

## Création d'une base de données Azure SQL
Ici, nous allons créer une base de données uniquement pour héberger et exécuter notre procédure stockée. Vous pouvez donc, si vous le souhiatez, utiliser une base de données Azure éxistante.

Retournez dans le groupe de ressources. Vous devez avoir votre compte de stockage comme première resource.

Cliquez sur le bouton "**Add**"

![sparkle](Pictures/013.png)

Puis, recherchez "**Azure SQL**" 

![sparkle](Pictures/014.png)

Cliquez sur le bouton **Create**

![sparkle](Pictures/015.png)

Choisissez **SQL Database** puis cliquez sur le bouton **Create**

![sparkle](Pictures/016.png)

Choisissez le groupe de ressouces précédement créé, définissez le nom de la base de données et créez un nouveau server SQL (il est aussi possible d'utiliser un serveur existant)

Un tier **Basic** sera largement suffiment pour notre démonstration

Cliquez sur le bouton **"Review + create"**


![sparkle](Pictures/017.png)

Cliquez sur le bouton **"Create"**

![sparkle](Pictures/018.png)

Après le déploiement de votre base de données et du server SQL vous devez avoir 3 services dans votre groupe de ressources

![sparkle](Pictures/019.png)

## Création du service Azure Data Factory

Dans votre groupe de ressources, cliquez sur le bouton **" + Add"**

![sparkle](Pictures/020.png)

Dans la barre de recherche entrez **"Data Factory"**

![sparkle](Pictures/021.png)

Puis cliquez sur le bouton **"Create"**

![sparkle](Pictures/022.png)


