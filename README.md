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

Après le déploiement de votre base de données Azure SQL et du server Azure SQL vous devez avoir 3 services dans votre groupe de ressources

![sparkle](Pictures/019.png)

## Création du service Azure Data Factory (ADF)

Dans votre groupe de ressources, cliquez sur le bouton **" + Add"**

![sparkle](Pictures/020.png)

Dans la barre de recherche entrez **"Data Factory"**

![sparkle](Pictures/021.png)

Puis cliquez sur le bouton **"Create"**

![sparkle](Pictures/022.png)

Vérifiez que vous avez bien sélectionné le bon groupe de ressources et donnez un nom à votre service ADF.

Sélectionnez **"V2"** pour la version.

Cliquez sur le bouton **"Next: Git configuration "**

![sparkle](Pictures/023.png)

Cochez la case **"Configure Git Later"** et cliquez sur le bouton **"Review + create"**

![sparkle](Pictures/024.png)

Dans la page de validation, ciquez sur le bouton **"Create"**

![sparkle](Pictures/025.png)

Après la création du service Azure Data Factory, vous devriez avoir 4 services dans votre groupe de ressources

![sparkle](Pictures/026.png)

## Préparation de la procédure stockée.

Dans notre exemple, la procédure stockée va lire des données dans un stockage blob et effectuer des transformations. Les transformations réalisées ici seront extrêmement basiques. Le but ici est d'illustrer l'utilisation des procédures stockées avec Azure Data Factory.

### Paramètrage du serveur Azure SQL

Configurez le Firewall du serveur Azure SQL afin de pouvoir vous y connecter avec des outils comme SQL Server Management Studio ou Azure Data Studio

Depuis le portail Azure, sélection votre serveur Azure SQL, puis cliquez sur **"Firewalls and virtual networks"**

Saisissez les adresses ip nécessaires.

![sparkle](Pictures/027.png)


Après configuration des adresses ip, cliquez sur le bouton **"Save"**

![sparkle](Pictures/028.png)

### Crétion du fichier de format

La procédure stockée va utiliser la fonction [OPENROWSET](https://docs.microsoft.com/fr-fr/sql/t-sql/functions/openrowset-transact-sql?view=sql-server-ver15). Et comme on souhaite récupérer les informations du fichier dans le but de faire des opérations sur les données, nous avons besoin de définir un [fichier de format](https://docs.microsoft.com/en-us/sql/t-sql/functions/openrowset-transact-sql?view=sql-server-ver15).

Le format des fichiers que nous allons traiter pour cet exemple est très simple. Il est constitué de 3 colonnes :

- Nom
- Prénom
- Vente

La création du fichier de format va se faire en 3 étapes

- Création d'une table SQL correspodant au format du fichier
- Utilisation de l'outil BCP pour créer le fichier de format
- Téléchargement du fichier dans le compte de stockage


#### Création de la table SQL

Avec [Azure Data Studio](https://docs.microsoft.com/fr-fr/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15), connectez vous à votre base Azure SQL, avec les touches **Ctrl** et **N** créé un nouveau fichier SQL

![sparkle](Pictures/029.png)


Puis copiez le script ci-dessous. Cliquez sur le bouton "Play"



CREATE TABLE [dbo].[MyFirstImport](
	[LastName] [varchar](30) NULL,
	[FirstName] [varchar](25) NULL,
	[Sales] [int] NULL
) ON [PRIMARY]
GO


![sparkle](Pictures/030.png)

Si tout se passe bien vous devriez avoir le message suivant et accès à votre table via le menu de gauche

![sparkle](Pictures/031.png)

### Création du fichier de format

Assurez-vous d'avoir la dernière version de l'outil BCP. Pour cet exemple, j'ai utilisé la [version 15](https://docs.microsoft.com/en-us/sql/tools/bcp-utility?view=sql-server-ver15).

Pour être certains d'utiliser la bonne version de l'outil BCP, allez dans le répertoire d'installation. Dans mon cas le répertoire est :

C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn

Puis utilisez la commande suivante (J'ai un répetoire *Temp* sur mon disque C:)

```javascript

bcp dbo.MyFirstImport format nul -c -x -f C:\Temp\format.xml -t, -U <Your User> -S tcp:<Your Server Name>.database.windows.net -d <Your Database name> -P <Your Password>

```

Une illustration ci-dessous :
![sparkle](Pictures/032.png)


Youd devez obtenir le ficier de format dans le répertoire spécifié avac la commande BCP

![sparkle](Pictures/033.png)


### Téléchargez le fichier format dans le compte de stockage Azure

Depuis le portail Azure, allez sur votre compte de stockage

![sparkle](Pictures/034.png)

Puis cliquez sur **"Containers"**

![sparkle](Pictures/035.png)

Et cliquez sur le bouton **"+ Container"**


![sparkle](Pictures/036.png)

Donnez un nom et cliquez sur le bouton **"Create"**

![sparkle](Pictures/037.png)