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

Nous allons créer un répertoire pour notre fichier de format.

Cliquez sur **"Storage Explorer (preview)"**, sélectionnez le conteneur créé précédement, puis cliquez sur **"New Folder"**

![sparkle](Pictures/038.png)

Donnez un nom au répertoire et cliquez sur le bouton **"Ok"**

![sparkle](Pictures/039.png)

Cliquez sur le bouton **"Upload"** et téléchargez le fichier de format précédement créé avec la fonction BCP.

![sparkle](Pictures/040.png)

Votre fichier est maintenant téléchargé.

![sparkle](Pictures/041.png)

### Préparation du partage de fichier

Depuis le portail Azure, sélectionnez votre compte de stockage puis cliquez sur **"File shares"**

![sparkle](Pictures/0411.png)

Cliquez sur le bouton **"+ File share"**

![sparkle](Pictures/0412.png)

Donnez un nom à votre partage de fichier puis cliquez sur le bouton **"Create"**

![sparkle](Pictures/0413.png)

Vous devriez obtenir un résultat similaire à la copie d'écran ci-dessous :

![sparkle](Pictures/0414.png)


## Création de la procédure stockée

Notre procédure stockée va lire des fichiers qui se trouvent dans notre compte de stockage et effectuer des opéations sur les données qu'elle va récupérer.

Il est donc néccessaire de faire des étapes préliminaires pour permettre à la procédure stockée d'accéder au compte de stockage

- Création d'une signature d'accès partagé (compte de stockage) [(Documentation)](https://docs.microsoft.com/fr-fr/azure/storage/common/storage-sas-overview)
- Création d'une clef principale de base de données [(Doucumentation)](https://docs.microsoft.com/fr-fr/sql/t-sql/statements/create-master-key-transact-sql?view=sql-server-ver15)

- Création des informations d'identification pour accéder au compte de stockage [(Documetation)](https://docs.microsoft.com/fr-fr/sql/t-sql/statements/create-database-scoped-credential-transact-sql?view=sql-server-ver15)
- Création d'une source externe [(Documentation)](https://docs.microsoft.com/fr-fr/sql/t-sql/statements/create-external-data-source-transact-sql?view=sql-server-ver15)


### Création d'une signature d'accès partagé (Compte de Stockage)

Depuis le portail Azure, allez dans votre compte de stockage puis cliquez sur **"Shared Access Signature"**.

Définissez les options de la signature d'accès partagé puis cliquez sur le bouton **"Generate SAS and connection string"**


![sparkle](Pictures/042.png)


Copiez le contenu du champ **"SAS Token"** puis gardez le sous la main, on va en avoir besoin un peu plus tard.

![sparkle](Pictures/043.png)


### Création d'une clef principale de base de données (Azure SQL)

Depuis Azure Data Studio, copiez la reqûete ci-dessous :

```javascript
CREATE MASTER KEY ENCRYPTION BY PASSWORD='<EnterStrongPasswordHere>';

```

Puis cliquez sur le bouton **Run**

![sparkle](Pictures/044.png)


### Création des informations d'identification pour accéder au compte de stockage

Depuis Azure Data Studio, exécutez le script ci-dessous :

**ATTENTION !!!!**, retirer le signe **"?"** après avoir copier votre signature d'accès partagé. 

```javascript
CREATE DATABASE SCOPED CREDENTIAL AccessAzureStorage
WITH
  IDENTITY = 'SHARED ACCESS SIGNATURE',
  -- Remove ? from the beginning of the SAS token
  SECRET = '<YOUR SHARED ACCESS SIGNATURE>' ;

```

Pour plus de clarté, voici une copie d'écran

![sparkle](Pictures/045.png)

### Création des informations d'identification pour accéder au compte de stockage 

Depuis Azure Data Studio, exécutez le script ci-dessous :

```javascript
CREATE EXTERNAL DATA SOURCE AzureStorageExternalData
WITH
  ( LOCATION = '<YOUR LOCATION>' ,
    CREDENTIAL = AccessAzureStorage ,
    TYPE = BLOB_STORAGE
  ) ;

```

Remplacez <YOUR LOCATION> par le chemin de votre conteneur. Cette information peut être retouvée dans le portail Azure, dans les propriétés de du conteneur

![sparkle](Pictures/046.png)

Ci-dessous une copie d'écran dans Azure Data Studio :

![sparkle](Pictures/047.png)

### Création de la procédure stockée

Dans Azure Data Studio, copiez le script ci-dessous :

```javascript
CREATE PROCEDURE Franmer
       @MyFileName nvarchar(MAX)
AS
BEGIN
       declare @query nvarchar(MAX)
       set @query = 'Select LastName, sum(Sales) as TotalSales FROM OPENROWSET(BULK ''' + @MyFileName + ''', 
       DATA_SOURCE = ''AzureStorageExternalData'',
       FORMAT=''CSV'',
       FORMATFILE=''Format/format.xml'',
       FORMATFILE_DATA_SOURCE = ''AzureStorageExternalData'') as products
       GROUP BY LastName;'
 
       EXEC sp_executesql @query
 END

```


![sparkle](Pictures/048.png)

Il est possible de tester la procédure stockée en téléchargeant le fichier d'exemple (qui se trouve dans le [github](https://github.com/franmer2/ADFandStoredProcedure/blob/master/Resources/test.csv)) à la racine du conteneur.


![sparkle](Pictures/049.png)

Puis dans Azure Data Studio, entrez le script ci-dessous:

```Javascript
EXECUTE franmer @MyFileName='test.csv'
```

Si tout se déroule comme prévu, vous devriez obtenir le résultat suivant :

![sparkle](Pictures/050.png)

## Création du pipeline Azure Data Factory

Depuis le portail Azure, retrouvez votre service Azure Data Factory, puis cliquez sur **"Author & Monitor"**

![sparkle](Pictures/051.png)

## Création des services liés
### Service lié Blob Storage

Une fois sur la pagge d'accueil d'AZure Data Factory, cliquez sur le bouton **"Manage"** à gauche de l'écran


![sparkle](Pictures/052.png)


Cliquez sur **"Linked services"** puis sur le bouton **"+ New"**  

![sparkle](Pictures/053.png)

Dans le liste des services liés, sélectionnez **"Azure Blob Storage"**

![sparkle](Pictures/054.png)

Donnez un nom au service lié, sélectionnez le compte de stockage puis testez la connexion en cliquant sur **"Test connection"** (1). Une fois le test réussi, cliquez sur le bouton **"Create"** (2).

![sparkle](Pictures/055.png)

### Service lié Azure File Storage

Créez un nouveau service lié de type **"Azure File Storage"**

![sparkle](Pictures/056.png)

Puis complétez les informations de connexion

![sparkle](Pictures/057.png)

### Service lié Azure SQL Database

Enfin, Créez un service lié de type **"Azure SQL Database"**

![sparkle](Pictures/058.png)

Puis complétez les informations de connexion

![sparkle](Pictures/059.png)

Vous devez avoir en tout 3 services liés

![sparkle](Pictures/060.png)
