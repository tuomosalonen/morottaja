# Morottaja - CI/CD-harjoitus

Tämä on JAMK/Tikon WebKehYmp-opintojakson CI/CD-demon esimerkkirepo. Morottaja-"sovellus" on staattinen html-sivu, johon tuodaan [Vue2](https://vuejs.org/)-kirjastolla yksinkertainen toiminallisuus. Sovellukseen tehdään [Cypress](https://www.cypress.io):lla automatisoitu testicase, joka ajetaan [CircleCI](https://circleci.com)-palvelua käyttäen. Lopuksi tehdään deployment Herokuun, mikäli testit menevät lävitse.

Asenna aluksi node, mikäli sitä ei ole koneellasi.

## Testit

- Forkkaa tämä repository ja kloonaa repo forkista omalle koneellesi.
- Kokeile "sovellusta" selaimessasi
- Tee hakemiston juureen .gitignore-tiedosto, jossa ainakin kohta node_modules (node-binaareja ei viedä gittiin).
- Tee hakemiston juureen tyhjä package.json-tiedosto antamalla komento `npm init --yes`. Asenna [Cypress](https://www.cypress.io) npm-moduulina komennolla `npm install cypress`.
- Avaa cypress ja tutustu sen toimintaan lyhyesti: `npx cypress open`. Voit tehdä myös npm-skriptin, jolloin modifioi `package.json`-tiedoston scripts-osaa seuraavasti:

```json
{
  "scripts": {
    "cypress:open": "cypress open",
    "cypress:run": "cypress run"
  }
}
```

- Siirry hakemistoon cypress/integrations ja poista sen sisältö (kaksi hakemistoa esimerkkejä).
- Luo cypress/integrations-hakemistoon tiedosto _nimi.js_ alla olevalla sisällöllä. Tämä tiedosto sisältää testit, jossa tutkitaan onko nimi-elementissä kiinni css-luokka "punainen", minkä jälkeen kirjoitetaan kenttään merkkijono "John Doe" ja tutkitaan onko "punainen"-luokka hävinnyt. Lopuksi tutkitaan sisäältääkö moro-otsikko-elementti tekstin "Moro John Doe".

```js
describe('moro-nimi', function () {
  it('Syötä John Doe tekstikenttään', function () {
    debugger;
    cy.visit(Cypress.env('HOST') || 'index.html');
    cy.get('#nimi')
      .should('have.class', 'punainen')
      .type('John Doe')
      .should('not.have.class', 'punainen');
    cy.get('#moro-otsikko').contains('Moro John Doe');
  });
});
```

- Aja testi (`npx cypress open` tai `npm run cypress:open`) ja valitse integration tests -listasta nimi.js. Tarkasta, että testi meni lävitse. Tee muutos index.html-tiedoston h1-elementtiin (esimerkiksi "moro" -> "morotus") ja varmistu, että testi ei mene lävitse. Palauta alkuperäinen sisältö h1-elementtiin.
- Aja testit komentoriviltä komennolla `npm run cypress:run` (jos lisäsit skriptit yllä) tai `npx cypress run` ja tarkasta, että Cypress generoi videon testistä (hakemisto cypress/videos). Poista videos-hakemisto.
- Loggaudu GitHubiin ja ota [CircleCI](https://circleci.com) käyttöön: GitHub Marketplace -> CircleCI. Kirjaudu CircleCI-palveluun ja tutustu siihen.
- Commitoi ja pushaa paikalliseen Git-työhakemistoon tekemäsi muutokset. Mikäli kloonasit alkuperäisen repon, tee uusi repo GitHubiin, vaihda origin sekä commitoi ja pushaa tavarat sinne. Mikäli nimeät master-branchin main-branchiksi, muista tehdä vastaava muutos myöhemmin circleCI:n konffitiedostoon.
- Integroi GitHubin repository ja circleci-palvelu: lisää git-tyohakemistoon _.circleci_-hakemisto ja sen alle _config.yml_-tiedosto alla olevalla sisällöllä. Commitoi ja pushaa.

```yaml
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
      - run:
          name: Morottajan nimi-kentän testaus
          command: npx cypress run
      - store_artifacts:
          path: cypress/videos
```

- Mene CircleCI:hin ja aktivoi buildaus GitHubin repositorylle. Kun build menee lävitse, varmistu että Artifacts-kohdassa on generoitu video. Nyt meillä on kasassa automaatisoitu e2e-testaus.
- Voit halutessasi kokeilla lisätä automaattiset chat-notifikaatiot Slackiin, jos käytät kyseistä sovellusta. Toiminto otetaan käyttöön CircleCI:n projektiasetuksissa.

## Tuotantoon siirto

Tehdään seuraavaksi toimet sovelluksen automaattiselle deploymentille Herokuun.

- Kirjaudu [Herokuun](https://www.heroku.com/) ja luo uusi app.
- Koska sovellus on staattista html:ää, lisää luomaasi appiin staattinen buildpack (osoite: https://github.com/heroku/heroku-buildpack-static) Settings-sivulla.
- Lisää git-tyohakemistoon tiedosto _static_.json, jossa on alla oleva sisältö. Commitoi.

```json
{
  "root": ".",
  "clean_urls": true
}
```

- Lisää CircleCI-projektin Environment variables -kohtaan ympäristömuuttujat: HEROKU_APP_NAME (arvoksi tekemäsi app:n nimi) ja HEROKU_API_KEY (käy kopioimassa tämän ympäristömuuttujan arvo kohdasta API key Herokun Account Settings -sivulta).
- Modifioi _.circleci/config.yml_ -tiedostoa siten, että lisäät siihen deploy-osan ja workflow'n seuraavaan tapaan ([ohjeita](https://circleci.com/docs/2.0/deployment-integrations/#heroku)). Mikäli vaihdoit master-branchin main-branchiksi, tee koodiin vastaava muutos.

```yaml
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
          name: Master Herokuun
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

- Commitoi muutokset ja pushaa GitHubiin. Mikäli testi menee lävitse, sovelluksen pitäisi olla käytettävissä osoitteessa https://appnimi.herokuapp.com.
- Poista lopuksi duunisi: Heroku-sovellus, Circleci-konffaus ja GitHub-forkki.
