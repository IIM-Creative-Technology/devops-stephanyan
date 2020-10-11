#  Ruby on Rails : GitHub Actions et Heroku

<p align=center>
  <img src="https://avatars0.githubusercontent.com/u/44036562?s=200&v=4" alt="github actions logo" width="200">
  <img src="https://pbs.twimg.com/media/CZGHPChUAAA3jqE.png" alt="ruby on rails logo" width="200">
  <img src="https://dailysmarty-production.s3.amazonaws.com/uploads/post/img/509/feature_thumb_heroku-logo.jpg" alt="heroku logo" width="200">
</p>

## Introduction

Ce README.md explique comment mettre en place un système de déploiement continu (CI/CD) avec [GitHub Actions](https://github.com/features/actions) et [Heroku](https://heroku.com/), pour un blog développé avec le framework Ruby on Rails.

## Branches et environnements

Ce repository contient deux branches principales liées à un environnement fonctionnant sur Heroku :

- `preprod` : il s'agit de la branche reliée à l'environnement de pré-production (parfois appelée `staging`), c'est-à-dire celui qui fait tournée une application sur laquelle on teste les nouvelles fonctionnalités ou autre changement à tester avant la présentation aux utilisateurs finaux. L'application est accessible sur [https://preprod-devops-stephanyan.herokuapp.com/](https://preprod-devops-stephanyan.herokuapp.com/).

- `master` : il s'agit de la branche reliée à l'environnement de pré-production , c'est à dire celle utilisée par les utilisateurs finaux. L'application est accessible sur [https://prod-devops-stephanyan.herokuapp.com/](https://prod-devops-stephanyan.herokuapp.com/).

Ces deux branches sont protégées : pour y apporter des changements, il faut passer par des pull requests. Il est donc impossible de pousser ou de merger ses changements directement sur ces deux branches.

<p align=center>
  <img src="https://i.ibb.co/7tSJz1N/Screenshot-2020-10-11-at-19-59-14.png" alt="protected branches">
</p>

## Workflows et automatisation

### Les 3 workflows

GitHub Actions facilite l'automatisation des workflows avec le système de CI/CD. Les différents contrôles de code (linting, tests unitaires ou end-to-end, détécteur de vulnérabilité de sécurité, etc.) et le déploiement du projet en ligne se font de manière automatique et non plus à la main grâce à des fichiers de configuration appelé "workflows" écrits en yaml. Dans ce repository, il en existe trois :

- `preprod.yml`
- `prod.yml`
- `pull_request.yml`

### Les pull requests

```yml
on: pull_request
```

Lorsqu'une pull request est ouverte et prête à la revue, plusieurs jobs vont s'exécuter en parallèle des autres, dont chacun contiennent plusieurs étapes ou steps. Le workflow qui va s'exécuter est le fichier pull_request.yml qui continent trois jobs : `linter`, `test` et `security`. **Le merge d'une pull request peut se faire seulement si ces trois jobs sont parfaitement au vert**.

<p align=center>
  <img src="https://i.ibb.co/G5Xk0tB/Screenshot-2020-10-11-at-19-59-49.png" alt="required checks">
</p>

<p align=center>
  <img src="https://i.ibb.co/P56Tqsh/Screenshot-2020-10-11-at-20-04-44.png" alt="applying ci">
</p>

#### Le linting

<p align=center>
  <img src="https://raw.githubusercontent.com/rubocop-hq/rubocop/master/logo/rubo-logo-square.png" alt="rubocop logo" width="150">
</p>

`linter` : il s'agit du job qui va mettre en place l'environnement qui permettra l'exécution du linter RuboCop. [RuboCop](https://rubocop.org/) est un vérificateur de style de code Ruby (linter) et un formateur basé sur le guide de style de Ruby faite par la communauté. RuboCop a besoin de Ruby:

```yml
jobs:

  # Nom du job
  linter:
  
    # Création d'un environnement virtual sous Ubuntu
    runs-on: ubuntu-latest

    # Liste des étapes
    steps:

      # Cette action permet de faire un "checkout" sur le repository dans
      # $GITHUB_WORKSPACE afin que le workflow puisse y accéder
      - uses: actions/checkout@v2

      # Cette action télécharge une version pré-buildée Ruby 
      # en version 2.7.0 et l'ajoute au PATH
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1.46.1
        with:
          ruby-version: 2.7.0

      # Cette commande installe le gestionnaire de dépendances Bundler
      - name: Install dependencies
        run: gem install bundler

      # Cette commande installe l'ensemble de dépendances et
      # permet l'exécution de quatre jobs en parallèle
      - name: Install Gems with Bundler
        run: bundle install --jobs 4

      # Cette commande lance le linter RuboCop qui affichera ou non
      # des warnings
      - name: Lint with RuboCop
        run: bundle exec rubocop
```

#### Les tests

<p align=center>
  <img src="https://rspec.info/images/logo.png" alt="rspec logo" width="150">
</p>
 
`test` : il s'agit du job qui va mettre en place l'environnement qui permettra l'exécution de RSpec. [RSpec](https://rspec.info/) est une librairie de tests pour Ruby. Dans ce projet Ruby on Rails, les tests ont besoin d'effectuer des appels dans une base de données. Dans ce workflow, il s'agira d'un service.

```yml
test:
    runs-on: ubuntu-latest
    
    # Liste des services
    services:
    
      # Cette commande installe la version 11 de PostgreSQL
      # et crée un utilisateur avec un mot de passe
      db:
        image: postgres:11
        ports: ['5432:5432']
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres

    steps:
    - uses: actions/checkout@v2

    - name: Setup Ruby
      uses: ruby/setup-ruby@v1.46.1
      with:
        ruby-version: 2.7.0

    # Cette commande installe Yarn et exécute `yarn install`
    - name: Install and run Yarn
      uses: Borales/actions-yarn@v2.3.0
      with:
        cmd: install

    # Cette étape installe d'abord des dépendances nécessaire
    # pour faire fonctionner la gem `pg` puis installe Bundler
    - name: Install dependencies
      run: |
        sudo apt install -yqq libpq-dev
        gem install bundler

    - name: Install Gems with Bundler
      run: bundle install --jobs 4

    # Cette commande crée la base de données dédiée
    # aux tests pour le projet
    - name: Create database
      run: bundle exec rails db:create RAILS_ENV=test

    # Cette commande joue les migrations pour
    # construire la base de donnée
    - name: Migrate database
      run: bundle exec rails db:migrate RAILS_ENV=test

    # Cette commande exécute les tests
    - name: Run RSpec tests
      run: bundle exec rspec
```

#### La détéction de vulnérabilité

<p align=center>
  <img src="https://camo.githubusercontent.com/92cf013ec2d2c5538bd5d0ec8b1fd600d3614f2b/687474703a2f2f6272616b656d616e7363616e6e65722e6f72672f696d616765732f6c6f676f5f6d656469756d2e706e67" alt="brakeman logo" width="150">
</p>

`security` : il s'agit du job qui va mettre en place l'environnement qui permettra l'exécution de Brakeman. [Brakeman](https://brakemanscanner.org/) est un outil d'analyse statique qui détecte les failles des sécurité des applications Ruby on Rails.

```yml
security:
  runs-on: ubuntu-latest

  steps:
    - uses: actions/checkout@v2

    - name: Setup Ruby
      uses: ruby/setup-ruby@v1.46.1
      with:
        ruby-version: 2.7.0

    # Cette commande installe la gem Brakeman
    - name: Install Brakeman
      run: |
        gem install brakeman

    # Cette commande exécute Brakeman et enregistre les résultats
    # dans un fichier
    - name: Run Brakeman
      run: |
        brakeman -f json > tmp/brakeman.json || exit 0

    # Cette action permet l'affichage du rapport de Brakeman
    - name: Brakeman Report
      uses: devmasx/brakeman-linter-action@v1.0.0
      env:
        GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
        REPORT_PATH: tmp/brakeman.json
```

### Les déploiements sur Heroku

Une fois les différents checks passées et une fois la branche mergée, en fonction de la branche visée, cette application est déployée sur deux environnements: celle de production et de pré-production.

Les workflows respectifs de ces deux environnements sont prod.yml et preprod.yml et contiennent exactement le même job `deploy` qui se charge de pousser en ligne les changements lors de push sur les branches `master` et `preprod`.

```yml
on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      # Cette action simplifie le déploiement de l'application sur Heroku
      - uses: akhileshns/heroku-deploy@v3.5.7
      
        # On précise les 3 clés secrètes nécessaire pour le déploiement
        with:
          heroku_api_key: ${{secrets.HEROKU_API_KEY}}
          heroku_app_name: ${{ secrets.HEROKU_PROD_APP_NAME }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
```

Il existe deux applications, donc deux noms différents. `HEROKU_PROD_APP_NAME` est est écrit le workflow prod.yml et `HEROKU_PREPROD_APP_NAME` dans le workflow preprod.yml. Ces clés secrètes sont ajoutées dans les paramètres du repository.

<p align=center>
  <img src="https://i.ibb.co/C5YH5VH/Screenshot-2020-10-11-at-19-00-17.png" alt="secret keys">
</p>

## Les configurations pour Heroku

Pour pouvoir lancer un projet Ruby on Rails, il faut construire la base de données du projet et Heroku ne le fera pas automatiquement. Deux solutions s'offrent à vous :

- La première serait rentrer dans la console Heroku et exécuter la commande `rails db:migrate` **à chaque déploiement...**

- La deuxième serait d'indiquer à Heroku de le faire automatiquement grâce à un fichier appelé Procfile. Comme nous voulons automatiser le plus de choses possible, c'est la solution que l'on choisira.

Pour cela, il faut créer un fichier Procfile à la racine du projet. Dedans, on y mettra :

```bash
web: jemalloc.sh bundle exec puma -C config/puma.rb
release: rails db:migrate
```

Ainsi, la commande `rails db:migrate` sera exécutée lors de la phase "Release Phase", soit à chaque déploiement.

[jemalloc](http://jemalloc.net/) est une implémentation de malloc à usage général qui évite la fragmentation de la mémoire dans les applications multithread. Pour pouvoir l'utiliser, les applications Heroku se basent sur le buildpack dédié [heroku-buildpack-jemalloc](https://elements.heroku.com/buildpacks/gaffneyc/heroku-buildpack-jemalloc).

Ce qui nous donne au total une liste de deux buildpacks.

<p align=center>
  <img src="https://i.ibb.co/mCrWDN3/Screenshot-2020-10-11-at-19-16-43.png" alt="buildpacks list">
</p>

Une fois l'application déployée, Heroku saura remplir seul les différentes variables d'environnement. Les variables ajoutées à la main sont `RAILS_MASTER_KEY` et `JEMALLOC_ENABLED`.

<p align=center>
  <img src="https://i.ibb.co/mCL0V7q/Screenshot-2020-10-11-at-19-16-53-png-final.png" alt="env variables">
</p>

