> Prérequis : Comprendre les languages de programations.

# Comment ai-je créé ce blog ?

J'aurais pu choisir la manière simple, mettre en place un WordPress ou tout autre outil de blogging. Héberger ça sur un de mes serveurs. Mais non, il a fallu que j'écrive [kblog](https://github.com/L-Briand/kblog). Un serveur web qui utilise des fichiers markdown pour produire des pages HTML.

Plus bas, vous verrez comment j'ai utilisé [ktor](https://ktor.io/), [commonsmark](https://commonmark.org/) et [korte](https://github.com/korlibs/korte).

## Kblog

### L'idée de base

Je voulais vraiment quelque chose de basique, un serveur web. Principalement du texte, sans cookies, sans tracking, sans base  de données. Il me fallait aussi un format de fichier simple et maléable dans l'optique d'écrire des articles / pages html simples. (Je voulais du markdown, oui.)

Avec des prérequis si minimalistes, plutôt que d'ouvrir google pour trouver une solution semi-préfaite, j'ai ouvert mon IDE  et ai commencé à développer avec mon language de prédilection, le [Kotlin](https://kotlinlang.org/).

### Les fonctionalitées

#### 🚀 Un serveur web avec du cache et de la compression.

J'ai utilisé la lib [ktor](https://ktor.io/). C'est une lib faite pour la « création de web services », mais elle fournit les même fonctionnalitées qu'un serveur web classique.

Pour l'aspect serveur et autonomie, le plugin gradle `org.gradle.application` permet de créer un applicatif kotlin / java standalone. Dans mon cas, je compte héberger ce site dans un container sur un serveur, ce qui est bien adapté.

#### 🔨 La transformation HTML des fichiers markdown.

Après un peu de recherche, il s'avère que la lib [commonsmark](https://commonmark.org/) fait très bien le travail. Il est même possible d'ajouter soit-même ses propre tags markdown et d'en générer du HTML customisé. 

#### 👀 Pouvoir modifier librement la forme des pages et leur contenu.

J'ai essayé certains standards de templating comme mustache, mais l'utilisation Kotlin en était horrible. Ma chance a été d'avoir trouvé [korte](https://github.com/korlibs/korte). Avec moins de 10 lignes de code j'ai réussi à insérer des fichiers markdown dans un template HTML, avec la possibilité d'importer d'autres fichiers à la volée.

## La technique

### [Ktor](https://ktor.io/)

```kotlin
fun main() {
    embeddedServer(Netty, port = 8000) {
        routing {
            get ("/") {
                call.respondText("Hello, world!")
            }
        }
    }.start(wait = true)
}
```

Ça, c'est ce qu'il faut pour créer un web-service « Hello World » avec Ktor. Plutôt simple non ? En fait non, pas vraiment. Sans connaissance de kotlin on peut croire en la magie... 

L'idée c'est qu'en Kotlin on peut changer de contexte en fonction de la lambda dans laquelle on est. Par contexte, j'entends que `this === MyObect`. Ça veut dire qu'on peut appeler des méthodes de classe directement par `myFun()` au lieu de `this.myFun()`.

Toute fonction prenant comme dernier paramètre une lambda peut être dispensée des parenthèses. Au lieu d'écrire `myFun("hello", { ... })` on aura tendance a l'écrire `myFun("hello") { ... }`. 

Dans l'exemple ci-dessus la fonction `routing` appartient à la classe `Application`. Et la méthode `embeddedServer` prend comme paramètre une lambda avec pour contexte `Application`. De ce fait, il est donc viable d'appeler la méthode `routing` comme par magie.

#### Structuration

Avec plusieurs endpoints le code peut vite devenir illisible, surtout que Ktor propose une panoplie de plugins pour aider à créer des endpoints. Pour remédier à ça, j'ai créé une interface. L'idée étant d'avoir un ou plusieurs web services par objet et de structurer mon code autour.

```kotlin
interface KtorModule {
    /** 
     * Create automatically an Application route, 
     * described by interface's implementation
     */
    fun module(app: Application) = app.routing { module() }

    /** Route implementation */
    fun Routing.module()
}
```

Je peux donc ensuite créer des singletons via cette interface.

```kotlin
object Articles : KtorModule {
    override fun Routing.module() {
        get("/articles") { ... }
        get("/articles/{id}") { ...}
    }
}

object Users : KtorModule {
    override fun Routing.module() {
        get("/user/me") { ... }
        get("/user/{id}") { ...}
    }
}
```

Les référencer dans une liste.

```kotlin
val MODULES = listOf(Articles, Users)
```

Puis les assigner via ktor.

```kotlin
embeddedServer(Netty, port = APP_CONFIG.port.toInt()) {
    MODULES.onEach { it.module(this) }
}.start(true)
```

### [Commonsmark](https://commonmark.org/)

Cette lib markdown est vraiment géniale. Même si elle est quelque peu basique, elle permet de parser des fichiers markdown et de les transformer en html de manière efficace. La lib est très modulaire et dispose d'un système de plugin pour faire soit-même de la génération html en fonction de ses propres critères.

Après l'ajout des 9 dépendances pour parser les tags markdown les plus communs...

```kotlin 
dependency {
    val commonmarkVersion = "0.18.1"
    implementation("org.commonmark:commonmark:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-ins:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-gfm-tables:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-autolink:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-gfm-strikethrough:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-image-attributes:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-task-list-items:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-heading-anchor:$commonmarkVersion")
    implementation("org.commonmark:commonmark-ext-yaml-front-matter:$commonmarkVersion")
}
```

On prend tous les plugins, on les met dans une liste.

```kotlin
private val extensions = listOf(
    AutolinkExtension.create(),
    StrikethroughExtension.create(),
    TablesExtension.create(),
    HeadingAnchorExtension.create(),
    InsExtension.create(),
    YamlFrontMatterExtension.create(),
    ImageAttributesExtension.create(),
    TaskListItemsExtension.create(),
)
```

Et on s'en sert pour générer notre markdown.

```kotlin
val parser = Parser.builder().extensions(extensions).build()
val renderer = HtmlRenderer.builder().extensions(extensions).build()

val markdown = """
# Hello world

Hello My friend
"""

val html: String = renderer.render(parser.parse(markdown))
```

### [korte](https://github.com/korlibs/korte)

Korte est un processeur de template développé en pur kotlin. Il n'est pas forcément destiné aux fichiers html mais on peut s'en servir pour.

Voici un exemple simple de template : 

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    {% include "_head.html" %}
</head>
<body>
    {{ content }}
</body>
</html>
````

Le plus dur ce n'est pas de faire un template mais de l'utiliser. Pour ça il faut comprendre un peu comment l'outil fonctionne.

Tout template porte un nom. Disons que celui du dessus s'appelle `_content.html`. Vu qu'il inclu `_head.html`, l'outil doit être capable de trouver un autre template nommé `_head.html`. Pour ça, il faut donc créer un template provider :

```kotlin
val provider = TemplateProvider(
    "_content.html" to "..."
    "_head.html" to "..."
)
```

Puis, de ce provider on en crée un objet Templates.

```kotlin
val templates = Templates(provider)
```

Et enfin, de cet objet on peut fournir nos informations pour la transformation.

```kotlin
templates.get("_content.html")(
    "title" to "My title",
    "content" to renderer.render(parser.parse(markdown))
)
```

## Récapépète pour voir ?

On prend un outil pour faire des web services, on utilise un autre outil pour générer des pages web en markdown et on met ça en forme avec un processeur de template. En 1 block ça ressemble à ça :

```kotlin
// j'omet l'aspect création de serveur.

val extensions = listOf(
    AutolinkExtension.create(),
    StrikethroughExtension.create(),
    TablesExtension.create(),
    HeadingAnchorExtension.create(),
    InsExtension.create(),
    YamlFrontMatterExtension.create(),
    ImageAttributesExtension.create(),
    TaskListItemsExtension.create(),
)

val templateProvider = TemplateProvider(
    "content" to "<!DOCTYPE html><html><head><title>{{ title }}</title></head><body>{{ content }}</body></html>"
)
val templates = Templates(provider)

val parser = Parser.builder().extensions(extensions).build()
val renderer = HtmlRenderer.builder().extensions(extensions).build()

val markdown = """
# Hello world

Hello My friend
"""

get("/") { 
    call.respondText(ContentType.Text.Html) { 
        templates.get("_content.html")(
            "title" to "My title",
            "content" to renderer.render(parser.parse(markdown))
        )
     }
}
```

# Le mot de la fin

C'est toujours une bonne idée de se faire quelques lignes de code pour tester son idée et voir le potentiel. J'ai omis pas mal de choses dans l'implémentation réelle de kblog… Cependant, ces quelques lignes de code expliquent bien le principe derrière l'idée que j'en avais. Elles peuvent être dérivées de la manière dont vous le souhaitez !

Sur ce, Bonne journée.

[Retour à l'accueil](/)

