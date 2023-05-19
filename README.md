# Typesense DocSearch scraper

## Start things up

1. Start Docusaurus
```bash
cd my-website
yarn build
yarn serve
```

1. Start Typesense server
```bash
docker compose up -d
```

1. Start crawler
```bash
docker run \
    -it --env-file=./.env \
    -e "CONFIG=$(cat config.json | jq -r tostring)" \
    --add-host=host.docker.internal:host-gateway  \
    docker.io/typesense/docsearch-scraper
```

## Dan's notes:

- Docusaurus site

npx seems to not be working today, so I copied in an old Docusaurus demo
so I can work.  It is in the dir `my-website`.  

I had to edit the `docusaurus.config.js` file to set the URL to 'http://host.docker.internal:3000' as this is used when generating the sitemap that the crawler uses.

Build the Docusaurus site:
```bash
cd my-website
yarn build
```

Host it on port 3000:
```bash
yarn serve
```

- Crawler config
The crawler docs provide a link to a Docusaurus 2.x crawler config and one line to update.
Since I am running Docusaurus on localhost and the crawler in a container I had to specify
the hostname as `host.docker.internal` instead of localhost.  Here is the crawler config:

```json
{
  "index_name": "docusaurus-2",
  "sitemap_urls": [
    "http://host.docker.internal:3000/sitemap.xml"
  ],
  "selectors": {
    "lvl0": {
      "selector": "(//ul[contains(@class,'menu__list')]//a[contains(@class, 'menu__link menu__link--sublist menu__link--active')]/text() | //nav[contains(@class, 'navbar')]//a[contains(@class, 'navbar__link--active')]/text())[last()]",
      "type": "xpath",
      "global": true,
      "default_value": "Documentation"
    },
    "lvl1": "article h1, header h1",
    "lvl2": "article h2",
    "lvl3": "article h3",
    "lvl4": "article h4",
    "lvl5": "article h5, article td:first-child",
    "lvl6": "article h6",
    "text": "article p, article li, article td:last-child"
  },
  "strip_chars": " .,;:#",
  "custom_settings": {
    "separatorsToIndex": "_",
    "attributesForFaceting": [
      "language",
      "version",
      "type",
      "docusaurus_tag"
    ],
    "attributesToRetrieve": [
      "hierarchy",
      "content",
      "anchor",
      "url",
      "url_without_anchor",
      "type"
    ]
  },
  "conversation_id": [
    "833762294"
  ],
  "nb_hits": 46250
}
```

- compose file for typesense server

```yaml
version: '3.4'
services:
  typesense:
    image: typesense/typesense:0.24.0
    restart: on-failure
    ports:
      - "8108:8108"
    volumes:
      - ./typesense-data:/data
    command: '--data-dir /data --api-key=xyz --enable-cors'
```

- Run typesense server

```bash
docker compose up -d
```

- env file that crawler will use
name is `.env`.  Since both typesense server and crawler are running in Docker containers
the TYPESENSE_HOST is `host.docker.internal`.  I should change this to be two services
in the docker compose file.

```
TYPESENSE_API_KEY=xyz
TYPESENSE_HOST=host.docker.internal
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
```

- Run a crawl
```bash
docker run \
  -it --env-file=./.env \
  -e "CONFIG=$(cat config.json | jq -r tostring)" \
  --add-host=host.docker.internal:host-gateway  \
  docker.io/typesense/docsearch-scraper
```  

Note: The above docker run command uses a local build of the image.  If you
want to run the typesense build from Docker Hub then replace the last line
of the above docker run command with:
```bash
  typesense/docsearch-scraper:0.6.0
```

- Add search widget to Docusaurus site
This requires adding a package, and editing the docusaurus.config.js file.

```bash
yarn add docusaurus-theme-search-typesense@next
```

Docusaurus config:

I add the `themes` line just before `themeConfig`.

I added the `typesense` theme config as the first entry in `themeConfig`.  You can see it below, it ends just before the navbar configuration.
```js
...
        theme: {
          customCss: require.resolve('./src/css/custom.css'),
        },
      }),
    ],
  ],

  themes: ['docusaurus-theme-search-typesense'],

  themeConfig:
    /** @type {import('@docusaurus/preset-classic').ThemeConfig} */
    ({
      typesense: {
      typesenseCollectionName: 'docusaurus-2', // Replace with your own doc site's name. Should match the collection name in the scraper settings.

      typesenseServerConfig: {
        nodes: [
          {
            host: 'localhost',
            port: 8108,
            protocol: 'http',
          },
        ],
        apiKey: 'xyz',
      },

      // Optional: Typesense search parameters: https://typesense.org/docs/0.21.0/api/documents.md#search-parameters
      typesenseSearchParameters: {},

      // Optional
      contextualSearch: true,
    },

      navbar: {
        title: 'My Site',
        ...
```

## Editing the code
As I am planning to customize the Docker container (update to latest Scrapy, etc.)
I had to update the Dockerfile as some of the packages are obsolete
(for example, the Google Chrome used by Selenium--I think, I don't
actually use Selenium myself).

So, to build a new container the command is:
```bash
# Run this from the root of the repo
./docsearch docker:build
```

## Updating Pipfile.lock
The ./docsearch docker:build command copies in the Pipfile.lock, so updating the Pipfile 
and then building is not sufficient.  To generate a new lock file run the container with
an overridden entrypoint:
```bash
docker run --entrypoint /bin/bash \
  -it --env-file=./.env 
  -e "CONFIG=$(cat config.json | jq -r tostring)" 
  --add-host=host.docker.internal:host-gateway  
  docker.io/typesense/docsearch-scraper
```

And then in the container run:
```bash
pipenv lock
```

Copy out the new Pipfile.lock file:
```bash
docker cp <container name>:/home/seleuser/Pipfile.lock .
```
