# Déployer Rails 7+ sur MySQL chez OVH(Lab)... Gratuitement (à date : 13 juin 2022) _par Jean-Baptiste VIDAL_

## Mission THP - Cursus Développeur - Hiver 2022

### Introduction

Durant le cursus "Fullstack", comme au long de son complément "Développeur(++)", nous avons, la plupart du temps, choisi une certaine facilité en faisant appel à un déploiement de nos applications Rails via Heroku, Github Pages et autres DaaS (Dynos as a Service) qui acceptent les technos Rails et la BD PostgreSQL en standard, nativement. Mais voilà, dans la vraie vie, on ne déploie pas toujours via un joli CLI dédié, pas toujours avec la même BD (d'où l'intérêt d'une couche d'abstraction comme le modèle "ActiveRecord" offert par Rails) et pas toujours sans toucher du fichier un peu de fichier config'.
Ayant moi-même besoin d'héberger mon prototype chez OVH, j'ai récemment fouillé leurs docs clients... Puis le web, à la recherche des étapes nécessaires pour déployer une application Rails (v7.0.3) de base chez eux, avec le support de la BD MySQL (v5.6+) qu'ils proposent par défaut (comme unique choix à ce stade).
C'est tout ce travail et ces heures de recherche que je souhaite, dans les lignes qui suivent, partager avec vous dans le cadre d'une mission THP : bonne lecture !

### Le tutoriel

1. Commencez, que vous soyez ou pas client(e) OVH [https://www.ovh.com/](https://www.ovh.com/), par souscrire à leur offre intitulée "**Web Power Beta RUBY**". Pas de panique : ce service reste GRATUIT pendant toute la durée des 3 périodes de test, _i.e._ Alpha / Beta / Gamma... Dont l'étape _Alpha_ en cours durent depuis un petit moment déjà). A ce jour, vous pouvez notamment entamer l'inscription via la page dédiée à Ruby & Rails : [https://labs.ovh.com/](https://labs.ovh.com/).

2. Ce faisant, il vous sera paut-être demandé de créer un compte client OVH, gratuit lui aussi, si vous n'en disposez pas déjà.

3. Une fois l'offre souscrite, lancez votre terminal favori et connextez-vous, en SSH, à votre nouvel _host_ via la commande `ssh <adresse de votre cluster Power Web Ruby OVH> -p 22` puis entrez le mot de passe demandé (fourni via un système de token web temporaire mais également modifiable _a volo_ via l'interface de vorte Manager OVH).

4. Au besoin, mettez à jour la version de rails en entreant `gem install -v <version souhaitée>`. NB: à la date où j'écris, la dernière version stable, v7.0.2, est sortie en février 2022 ; depuis suivi d'une version beta, v7.0.3, disponible depuis mai 2022). A vous de voir !

5. Vérifiez que le _path_ vers les gem Ruby est bien configuré via `gem env gempath` qui devrait renvoyer quelque chose du genre: `/home/<username>/.gem/ruby/2.7.0:/usr/local/ruby2.7/lib/ruby/gems/2.7.0`. Ajoutez ledit chemin au _path_ de votre distribution Linux s'il n'y figurait pas déjà (_cf._ [ce tutoriel en anglais](https://linuxize.com/post/how-to-add-directory-to-path-in-linux/), si nécessaire).

6. De même, assuez-vous que le lien vers le serveur SPRING est bien dans votre _path_ et de l'ajouter sinon, via un `export SPRING_SERVER_COMMAND='$HOME/www/bin/spring server'`

7. Dans le répertoire de base de votre _host_, supprimez sans crainte le répertoire `www` existant via un `rm -rf www` (c'est bon, vous avez arrêté de trembler ;-) ) : nous allons le remplacer par un homonyme plus adapté.

8. Installez la gem "MySQL 2" via `gem install mysql2` et, plus tard, ajoutez-la dans le Gemfile généré à l'étape 9 (ajout suivi d'un `bundle update`, BTW).

9. Créez un projet Rails nommé `www`, adapté à une base de données MySQL via la commande "rails new www -d mysql"

10. Si tout a fonctionné, Rails devrait déjà avoir lancé le bundler MAIS, si tel n'était pas le cas, positionnez-vous dans le répertoier "www" (si vous n'étiez pas djéà dedans) et lancez `bundle install` ou `bundler`.

11. Créez votre base de données MySQL via l'interface OVH Lab. Rien de compliqué là-dedans normalement ; surtout qu'OVH Lab ne vous laisse pas le choix de la version :-)

12. Allez, on relève ses manches et on plonge dans la configuration des liens entre tout ça ! Commencez donc pas configurer la connexion à la BD dans Rails en éditant le fichier `/config/database.yml`

```ruby
    default: &default
    adapter: mysql2
    encoding: utf8mb4
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") {5} %>
    username: <mysql_username> # Entrez ici votre utilisateur MySQL     (pas votre utilisateur OVH, s'il sont différents)
    password: <mysql_password> # Entrez ici, en clair, le mot de    passe MySQL (pas celui pour vous connecter à votre compte   client OVH)
    host: <mysql_username>.mysql.db

    development:
    <<: \*default

    # mettre en commentaires la BD par défaut ci-dessous...
    # database: www_development
    # ... Et la remplacer par le nom de la bd juste créée     (généralement identique au <username>)

    database: <ma_base_de_données>
```

13. Autorisez bien les requêtes de votre `hostname` à être traitées par le serveur Rails en éditant (pour l'environnement de développement là encore) le fichier `/config/environments/development.rb` pour lui ajouter en pied de fichier (juste avant le "end" final) :

    `config.hosts << '<ovh_username>.<servername>'`

    (ex: `config.hosts << 'abcdef.cluster030.hosting.ovh.net`)

14. l'étape 13 peut être répétée pour chaque environnement (_i.e._ dev / test / prod) si utile, ou, plus exceptionnellement, intégrer le fichier commun de configuration des environnements `/config/environment.rb` pour s'appliquer aux 3 contextes en même temps.

15. A ce stade, pas besoin de passer par un `rails db:create` puisque la BD a normalement été générée à l'étape n°11, via l'interface graphique web d'OVH. A noter que la création effective peut prendre quelques minutes: à vérifier dans votre tableau de bord, donc !

16. Pour (re)lancer votre environnement Rails, 3 options:

- Un bon petit `rails restart`  
  OU
- (créer, si pas déjà présent, puis) mettre à jour le fichier `restart.txt` dans `./tmp/`, en vous positoinnant dans `www` puis tapant `touch ./tmp/restart.txt`, par exemple
  OU
- Passez en mode "auto restart" (mise à jour à chaque modification) en créant un fichier `always_restart.txt` dans ce même répertoire `./tmp/`

17. A présent, lancez votre navigateur et butinez vers l'adresse `http://<ovh_username>.<ovh_servername>/` (ex. "http://abcedf.cluster030.hosting.ovh.net/")

18. vous devriez voir apparaître la mire au rond rouge caractéristique de Rails v7+ : félicitations, vous l'avez fait !

### Sources:

- [https://docs.ovh.com/fr/web-power](https://docs.ovh.com/fr/web-power/premiers-pas-avec-hebergement-web-POWER/)
- [https://labs.ovh.com/managed-ruby](https://labs.ovh.com/managed-ruby)
- [https://docs.ovh.com/fr/web-power/nodejs-installer-rails/](https://docs.ovh.com/fr/web-power/nodejs-installer-rails/)

### Pour aller plus loin:

- Tutorial HOTWIRE (parce que Rails 7+ ne jure plus que par ce framework... Exit les autres front et même Webpack en middleware de gestion d'assets, a priori) - [https://www.bootrails.com/blog/rails-7-hotwire-a-tutorial/](https://www.bootrails.com/blog/rails-7-hotwire-a-tutorial/)
- Explorer l'API OVH - [https://eu.api.ovh.com/console/](https://eu.api.ovh.com/console/)
- Connecter Git - [https://git-scm.com/book/fr/v2/Git-sur-le-serveur-Mise-en-place-du-serveur](https://git-scm.com/book/fr/v2/Git-sur-le-serveur-Mise-en-place-du-serveur)
