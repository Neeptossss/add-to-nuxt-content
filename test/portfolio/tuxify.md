---
title: Tuxify2
description: Tuxify is a workflow automation tool version 2.
date: 18-04-2023
image: /images/tuxify.jpg
---

::Hero
---
src: /images/tuxify.jpg
alt: Tuxify
---
::

# Tuxify

- [Introduction](#introduction)
- [Génération des modules et implémentation de base pour le développement](#génération-des-modules-et-implémentation-de-base-pour-le-développement)
- [Ajouter le heartbeat du fournisseur](#ajouter-le-heartbeat-du-fournisseur)
- [Implémentation de l'authentification](#implémentation-de-lauthentification)

## Introduction

Tuxify étant basé sur une architecture micro-services, il est très facile d'ajouter un nouveau fournisseur. Chaque
fournisseur est un micro-service à part entière. Il est donc possible d'ajouter un nouveau fournisseur sans avoir à
effectuer d'actions particulières sur les autres micro-services existants.

**Caractéristiques principales d'un fournisseur:**

- Un module d'authentification (OAuth2, OpenID, ...)
- Un module d'action
- Un module de réaction
- Un module de gestion des jetons d'accès (rafraîchissement, révocation, ...)

## Génération des modules et implémentation de base pour le développement

**Génération du micro-service**
Chaque micro-service de fournisseur se situera dans le dossier `backend/external-providers/`. Pour générer un nouveau
micro-service, il suffit d'utiliser la commande suivante:

```bash
nest new <provider-name>
```

**Ajout du dockerfile de développement**
Pour que le micro-service soit pris en compte par le docker-compose de développement, il faut ajouter un
fichier `Dockerfile.dev` à la racine du micro-service. Voici un exemple:

```dockerfile
FROM node:latest

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install
```

**Ajout du micro-service au docker-compose de développement**
Pour que le micro-service soit pris en compte par le docker-compose de développement, il l'ajouter au
fichier `docker-compose.yml`, voici un exemple:

```yaml
api_microsoft:
  container_name: tuxify-api-<provider-name>
  build:
    context: ./backend/external-providers/<provider-name>
    dockerfile: Dockerfile.dev
  restart: always
  command: npm run start:dev --watch-preserve-output
  working_dir: /usr/src/app
  ports:
    - ${NESTSV_<provider-name>_PORT}:${NESTSV_<provider-name>_PORT}
  volumes:
    - ./backend/external-providers/<provider-name>:/usr/src/app
    - /usr/src/app/node_modules
  depends_on:
    - api_providers
  networks:
    - tuxify-network
  env_file:
    - .env
```

**_Note:_** _Il est important de bien respecter la convention de nommage des micro-services de fournisseurs, c'est à
dire: `api_<provider-name>`._ ainsi pour les variables d'environnement, il faut utiliser le
préfixe `NESTSV_<PROVIDER_NAME>_<VARIABLE_NAME>`.

**Génération des modules d'authentification, d'action, de réaction et de gestion des jetons d'accès**
Pour générer les modules d'authentification, d'action, de réaction et de gestion des jetons d'accès, il suffit
d'utiliser la commande suivante:

```bash
nest g module <provider-name>/auth
nest g module <provider-name>/actions
nest g module <provider-name>/reactions
nest g module <provider-name>/tokens
```

**Génération des contrôleurs d'authentification, d'action, de réaction et de gestion des jetons d'accès**
Pour générer les contrôleurs d'authentification, d'action, de réaction et de gestion des jetons d'accès, il suffit
d'utiliser la commande suivante:

```bash
nest g controller <provider-name>/auth
nest g controller <provider-name>/actions
nest g controller <provider-name>/reactions
```

**Génération des services d'authentification, d'action, de réaction et de gestion des jetons d'accès**
Pour générer les services d'authentification, d'action, de réaction et de gestion des jetons d'accès, il suffit
d'utiliser la commande suivante:

```bash
nest g service <provider-name>/auth
nest g service <provider-name>/actions
nest g service <provider-name>/reactions
nest g service <provider-name>/tokens
```

## Ajouter le heartbeat du fournisseur

**Ajout du heartbeat**
Pour ajouter le heartbeat du fournisseur, il faut ajouter les lignes suivantes dans le constructeur du
contrôleur `<provider-name>/app.controller.ts`:

```typescript
setInterval(() => {
    const providerInfos: ActionReactionService = {
        name: "<provider-name>",
        image: "<provider-image>",
        title: "<provider-title>",
        description: "<provider-desc>",
        actions: this.availableActions,
        reactions: this.availableReactions,
    };
    this.natsClient.emit<ActionReactionService>(
        "heartbeat.providers.<provider-name>",
        providerInfos
    );
}, 5000);
```

Ajouter également les routes d'écoutes des heartbeat des actions & réactions dans le
contrôleur `<provider-name>/app.controller.ts`:

```typescript
@EventPattern("heartbeat.providers.<provider-name>.actions")
setActionsInfos(@Payload()
data: ActionReaction[]
):
void {
    this.availableActions = data;
}

@EventPattern("heartbeat.providers.<provider-name>.reactions")
setReactionsInfos(@Payload()
data: ActionReaction[]
):
void {
    this.availableReactions = data;
}
```

Le heartbeat du fournisseur est maintenant fonctionnel. Le micro-service enverra toutes les 5 secondes les informations
sur les actions et réactions disponibles. Ces informations seront ensuite utilisées par le micro-service `api_providers`
pour mettre à jour la liste des actions et réactions disponibles.

## Implémentation de l'authentification

**Ajout des variables d'environnement**
Pour que le micro-service puisse fonctionner correctement, vous devez ajouter les variables d'environnement utilisées
par le module d'authentification. Voici un exemple utilisant une OAuth2:

```dotenv
NESTSV_<PROVIDER_NAME>_CLIENT_ID=<client-id>
NESTSV_<PROVIDER_NAME>_CLIENT_SECRET=<client-secret>
```

**Ajout des routes d'authentification**
Pour ajouter les fonctionnalités d'authentification, il faut ajouter les routes d'écoute suivantes au
fichier `<provider-name>/auth/auth.controller.ts`:

```typescript
@MessagePattern("provider.<provider-name>.add")
async
addProvider(@Payload()
addProvider: AddProvider
):
Promise < string | void > {
    try {
        return await this.authService.addProvider(addProvider);
    } catch(e) {
        throw new RpcException(e.message);
    }
}

@MessagePattern("provider.<provider-name>.add.callback")
async
addProviderCallback(@Payload()
addProviderCallback: AddProviderCallback
):
Promise < AddedProvider > {
    try {
        return await this.authService.addProviderCallback(
            addProviderCallback,
        );
    } catch(e) {
        throw new RpcException(e.message);
    }
}

@MessagePattern("provider.<provider-name>.refresh")
async
refreshTokens(@Payload()
providerEntity: ProviderEntity
):
Promise < ProviderEntity > {
    try {
        return await this.authService.refreshTokens(providerEntity);
    } catch(e) {
        throw new RpcException(e.message);
    }
}
```

**Ajout des méthodes d'authentification**
Pour ajouter les méthodes d'authentification, il faut ajouter les méthodes spécifique à votre service au
fichier `<provider-name>/auth/auth.service.ts`.
Chaque implémentation est différente, veillez à bien respecter les spécifications de votre fournisseur.

**_Note:_** _Vous pouvez utiliser les wrappers OAuth2 spécifique à votre fournisseur, ou bien utiliser le wrapper axios
pour effectuer les requêtes HTTP._

## Implémentation du module de gestion des jetons d'accès

**Ajout des méthodes de gestion des jetons d'accès**
Pour ajouter les méthodes de gestion des jetons d'accès, il faut ajouter les lignes suivantes au
fichier `<provider-name>/tokens/tokens.service.ts`:

```typescript
export class TokensService {
    public readonly logger: Logger = new Logger(TokensService.name);

    constructor(
        @Inject("NATS_CLIENT") private readonly natsClient: ClientProxy
    ) {
    }

    /**
     * The function `getTokens` retrieves user provider tokens for a specific user
     * from the <provider-name> provider.
     * @param {number} userId - The `userId` parameter is a number that represents
     * the unique identifier of a user. It is used to retrieve the tokens for a
     * specific user from a provider.
     * @returns a Promise that resolves to an object of type UserProviderTokens.
     */
    public async getTokens(userId: number): Promise<UserProviderTokens> {
        try {
            const providerRequestTokens: ProviderRequestTokens = {
                provider: "<provider-name>",
                userId,
            };
            return lastValueFrom<UserProviderTokens>(
                this.natsClient.send(
                    "providers.<provider-name>.getTokens",
                    providerRequestTokens
                )
            );
        } catch (err) {
            throw new RpcException(err);
        }
    }

    /**
     * The function `getAllTokens` retrieves all tokens from the <provider-name>
     * provider using NATS messaging.
     * @returns a Promise that resolves to an array of ProviderEntity objects.
     */
    public async getAllTokens(): Promise<ProviderEntity[]> {
        try {
            const usersProviderTokensObservable = this.natsClient.send(
                "providers.<provider-name>.getAllTokens",
                "<provider-name>"
            );
            return lastValueFrom<ProviderEntity[]>(usersProviderTokensObservable);
        } catch (err) {
            throw new RpcException(err);
        }
    }
}
```

## Implémentation du module d'action

**Ajout du heartbeat des actions**
Pour ajouter le heartbeat des actions, il faut ajouter les actions disponibles sour la forme d'un objet de
type `ActionReaction` dans le constructeur du contrôleur `<provider-name>/actions/actions.controller.ts`. Voici un
exemple:

```typescript
setInterval(() => {
    const availableActions: ActionReaction[] = [
        {
            name: "provider.microsoft.action.outlook",
            type: "action",
            title: "New email",
            description: "Trigger when a new email arrives",
            inputs: [],
            outputs: [
                {
                    name: "from",
                    title: "From",
                },
                {
                    name: "subject",
                    title: "Subject",
                },
                {
                    name: "body",
                    title: "Body",
                },
            ],
        },
    ];
    this.natsClient.emit<ActionReaction[]>(
        "heartbeat.providers.<provider-name>.actions",
        availableActions
    );
}, 5000);
```

Votre micro-service enverra toutes les 5 secondes les informations sur les actions disponibles. Ces informations seront
ensuite renvoyée vers votre module d'application principale qui les utilisera pour mettre à jour la liste des actions
disponibles et les enverra vers le micro-service `api_providers`.

**Ajout des routes d'écoute des actions**
Les routes d'écoutes suivent tout un même schéma, `provider.<provider-name>.action.<action-name>^` pour les actions
et `provider.<provider-name>.subscribe.<reaction-name>^` pour les abonnements.

Pour ajouter les routes d'écoute des actions, il faut ajouter les routes d'écoute au
fichier `<provider-name>/actions/actions.controller.ts`. Cette action va être déclanchée lors de l'arrivée d'un payload
sur le webhook configuré. Voici un exemple:

```typescript
@EventPattern("provider.microsoft.action.outlook")
/**
 * The `receiveEmail` function is an asynchronous function that receives an
 * Outlook message notification and passes it to the `receiveEmail` method of
 * the `actionsService` object.
 * @param {OutlookMessageNotification} data - The `data` parameter is of type
 * `OutlookMessageNotification` and represents the payload of the email
 * notification received.
 */
async
receiveEmail(@Payload()
data: OutlookMessageNotification
):
Promise < void > {
    await this.actionsService.receiveEmail(data);
}
```

Il se peut que votre fournisseur exige un abonnement à un webhook pour pouvoir recevoir les notifications. Dans ce cas,
il faut ajouter les routes d'écoute des webhooks au fichier `<provider-name>/actions/actions.controller.ts`. Voici un
exemple:

```typescript
@EventPattern("provider.microsoft.subscribe.outlook")
/**
 * The function `subscribeToOutlook` is an asynchronous function that takes a
 * payload and subscribes to Outlook using the provided input.
 * @param csi - The `csi` parameter is of type
 * `CommonSubscribeInput<SubscribeOutlookInput>`. It is an input object that
 * contains the necessary information for subscribing to Outlook. The
 * `CommonSubscribeInput` is a generic type that takes `SubscribeOutlookInput`
 * as its type argument.
 * @returns a Promise that resolves to void.
 */
async
subscribeToOutlook(
    @Payload()
csi: CommonSubscribeInput<SubscribeOutlookInput>
):
Promise < void > {
    return await this.actionsService.subscribeToOutlook(csi);
}
```

**_Note:_** _La route d'écoute d'abonnement (si elle existe) est déclanchée lorsqu'un utilisateur ajoute l'action en
question à un de ses flows._

## Implémentation du module de réaction

**Ajout du heartbeat des réactions**
Pour ajouter le heartbeat des réactions, il faut ajouter les réactions disponibles sour la forme d'un objet de
type `ActionReaction` dans le constructeur du contrôleur `<provider-name>/reactions/reactions.controller.ts`. Voici un
exemple:

```typescript
setInterval(() => {
    const availableReactions: ActionReaction[] = [
        {
            name: "provider.microsoft.reaction.onenote.create",
            type: "reaction",
            title: "Create a OneNote page",
            description: "Create a new OneNote page",
            inputs: [
                {
                    name: "title",
                    title: "Title",
                    placeholder: "title",
                    required: true,
                },
                {
                    name: "content",
                    title: "Content",
                    placeholder: "content",
                    required: true,
                },
            ],
            outputs: [
                {
                    name: "pageId",
                    title: "Page ID",
                },
            ],
        },
    ];
    this.natsClient.emit<ActionReaction[]>(
        "heartbeat.providers.<provider-name>.reactions",
        availableReactions
    );
}, 5000);
```

Votre micro-service enverra toutes les 5 secondes les informations sur les réactions disponibles. Ces informations
seront ensuite renvoyée vers votre module d'application principale qui les utilisera pour mettre à jour la liste des
réactions disponibles et les enverra vers le micro-service `api_providers`.

**Ajout des routes d'écoute des réactions**
Les routes d'écoutes suivent tout un même schéma, `provider.<provider-name>.reaction.<reaction-name>^`.

Pour ajouter les routes d'écoute des réactions, il faut ajouter les routes d'écoute au
fichier `<provider-name>/reactions/reactions.controller.ts`. Voici un exemple:

```typescript
@MessagePattern("provider.microsoft.reaction.onenote.create")
/**
 * The function `createOneNotePage` takes a `commonReactionInput` and returns
 * an observable of type `OneNotePageOutput`.
 * @param commonReactionInput - A generic input object that contains the
 * necessary data for creating a OneNote page. It is of type
 * CommonReactionInput<OneNotePageInput>.
 * @returns an Observable of type OneNotePageOutput.
 */
createOneNotePage(@Payload()
commonReactionInput: CommonReactionInput<OneNotePageInput>
):
Observable < OneNotePageOutput > {
    return this.reactionsService.createOneNotePage(commonReactionInput);
}
```

## Ajout des tests unitaires

**Ajout des tests unitaires**
Pour ajouter les tests unitaires, il faut ajouter les tests unitaires au fichier `<provider-name>.service.spec.ts`.