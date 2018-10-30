# Morottaja - CI/CD-harjoitus

Tämä on JAMK/Tikon OhTu-opintojakson CI/CD-demon esimerkkirepo. Morottaja-"sovellus" on staattinen html-sivu, johon tuodaan [Vue](https://vuejs.org/)-kirjastolla yksinkertainen toiminallisuus. Sovellukseen tehdään [Cypress](https://www.cypress.io):lla automatisoituja testejä, jotka ajetaan [CircleCI](https://circleci.com)-palvelua käyttäen. Lopuksi tehdään deployment Herokuun, mikäli testit menevät lävitse.

Deployment AWS S3:een kontaktitunnilla.

## Testit

- Kloonaa tämä repository omalle koneellesi.
- Asenna [Cypress](https://www.cypress.io) npm-moduulina kloonattuun hakemistoon `npm install cypress`. Koneellasi tulee olla luonnollisesti node asennettuna.
- Tee hakemiston juureen .gitignore-tiedosto, jossa ainakin kohta node_modules, jotta node-binaareja ei viedä gittiin.
- Avaa cypress ja tutustu sen toimintaa lyhyesti: `npx cypress open` tai pitkässä muodossa (jos ei npx:ää): `./node_modules/.bin/cypress open`. Käynnistyksen voi tehdä myös npm:n kautta, esim. `npm run cypress:open`. Tätä varten tulee editoida package.json-tiedostoa seuraavasti: 
```
{
  "scripts": {
    "cypress:open": "cypress open"
  }
}
```
- Siirry hakemistoon cypress/integrations ja poista hakemisto examples.
- Luo cypress/integrations-hakemistoon tiedosto nimi.js alla olevalla sisällöllä. Tämä tiedosto sisältää testit, jossa tutkitaan onko #nimi-elementissä kiinni css-luokka "punainen", minkä jälkeen kirjoitetaan kenttään merkkijono "John Doe" ja tutkitaan onko "punainen"-luokka hävinnyt. Lopuksi tutkitaan sisäältääkö #moro-otsikko -elementti tekstin "Moro John Doe".
```
describe('moro-nimi', function() {
  it('Syötä John Doe tekstikenttään', function() {
    debugger
    cy.visit(Cypress.env('HOST') || 'index.html');
    cy.get('#nimi').should('have.class','punainen').type('John Doe').should('not.have.class','punainen');
    cy.get('#moro-otsikko').contains('Moro John Doe');
  })
})
```
- Aja testi (`npm run cypress:open`) ja valitse integration tests -listasta nimi.js. Tsekkaa, että testi meni lävitse. Tee muutos index.html-tiedoston h1-elementtiin (esimerkiksi "moro" -> "morotus") ja varmistu, että testi ei mene lävitse. Palauta alkuperäinen sisältö h1-elementtiin.
- Aja testit komentoriviltä komennolla `npx cypress run` tai `$(npm bin)/cypress run` ja varmistu, että Cypress generoi videon testistä (hakemisto cypress/videos). Poista videos-hakemisto.
- Loggaudu githubiin ja ota [CircleCI](https://circleci.com) käyttöön: github marketplace -> circleci. Kirjaudu circleci-palveluun ja tutustu siihen.
- Poista kloonatusta git-tyohakemistosta .git-hakemisto (tai vaihda gitin origin), tee uusi repo githubiin ja pushaa tavarat sinne.
- Integroi githubin repository ja circleci-palvelu. Lisää git-tyohakemistoon .circleci-hakemisto ja sen alle config.yml-tiedosto alla olevalla sisällöllä. Commitoi ja pushaa.
```
version: 2

jobs:
  build:
    docker:
      # the Docker image with Cypress dependencies
      - image: cypress/base:10
        environment:
          ## this enables colors in the output
          TERM: xterm
    working_directory: ~/app
    parallelism: 1
    steps:
      - checkout
      - restore_cache:
          keys:
            - v2-deps-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - v2-deps-{{ .Branch }}-
            - v2-deps-
      - run: npm install cypress --save-dev
      - save_cache:
          key: v2-deps-{{ .Branch }}-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
            - ~/.cache
      - run:
          name: Morottajan nimi-kentän testaus
          command: npx cypress run
      - store_artifacts:
          path: cypress/videos
```
- Mene Circleci:hin ja aktivoi buildaus githubin repositorylle. Kun build menee lävitse, varmistu että artifacts-kohdassa on generoitu video. Nyt meillä on kasassa automaattinen buildien testaus. 
- Kokeile halutessasi lisätä automaattiset chat-notifikaatiot esimerkiksi Slackiin. Toiminto otetaan käyttöön CircleCI:n asetuksissa kohdassa Chat Notifications.

## Tuotantoon siirto

Tehdään seuraavaksi toimet sovelluksen automaattiselle deploymentille Herokuun.

- Kirjaudu [Herokuun](https://www.heroku.com/) ja luo uusi app. 
- Koska sovellus on staattista html:ää, lisää luomaasi appiin staattinen buildpack (osoite: https://github.com/heroku/heroku-buildpack-static) settings-sivulla.
- Lisää git-tyohakemistoon tiedosto static.json, jossa on seuraava sisältö (commitoi ja pushaa):
```
{
  "root": ".",
  "clean_urls": true
}
```
- Lisää CircleCI-projektin Environment variables -kohtaan ympäristömuuttujat: HEROKU_APP_NAME (arvoksi tekemäsi app:n nimi) ja HEROKU_API_KEY (käy kopioimassa tämän ympäristömuuttujan arvo kohdasta API key Herokun Account Settings -sivulta).
- Modifioi .circleci/config.yml -tiedostoa siten, että lisäät siihen deploy-osan ja workflow'n seuraavaan tapaan ([ohjeita](https://circleci.com/docs/2.0/deployment-integrations/#heroku)):
```
version: 2

jobs:
  build:
    docker:
      # the Docker image with Cypress dependencies
      - image: cypress/base:10
        environment:
          ## this enables colors in the output
          TERM: xterm
    working_directory: ~/app
    parallelism: 1
    steps:
      - checkout
      - restore_cache:
          keys:
            - v2-deps-{{ .Branch }}-{{ checksum "package-lock.json" }}
            - v2-deps-{{ .Branch }}-
            - v2-deps-
      - run: npm install cypress --save-dev
      - save_cache:
          key: v2-deps-{{ .Branch }}-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
            - ~/.cache
      - run:
          name: Morottajan nimi-kentän testaus
          command: npx cypress run
      - store_artifacts:
          path: cypress/videos

  deploy:
    docker:
      - image: buildpack-deps:trusty
    steps:
      - checkout
      - run:
          name: Deploy Master to Heroku
          command: |
            git push https://heroku:$HEROKU_API_KEY@git.heroku.com/$HEROKU_APP_NAME.git master

workflows:
  version: 2
  build-deploy:
    jobs:
      - build
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: master
```

- Commitoi muutokset ja pushaa githubiin. Mikäli testi menee lävitse, sovelluksen pitäisi olla käytettävissä osoitteessa https://appnimi.herokuapp.com.
