# Typesense DocSearch scraper

## Dan's notes:

- Docusaurus site

npx seems to not be working today, so I copied in an old Docusaurus demo
so I can work.  It is in the dir `my-website`.  

Build the Docusaurus site:
```bash
cd my-website
yarn build
```

Host it on port 3000:
```bash
yarn serve
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
  typesense/docsearch-scraper:0.6.0
```


This is a fork of Algolia's awesome [DocSearch Scraper](https://github.com/algolia/docsearch-scraper), customized to index data in [Typesense](https://typesense.org). 

You'd typically setup this scraper to run on your documentation site, and then use [typesense-docsearch.js](https://github.com/typesense/typesense-docsearch.js) to add a search bar to your site. 

#### What is Typesense? 

If you're new to Typesense, it is an **open source** search engine that is simple to use, run and scale, with clean APIs and documentation. 

Think of it as an open source alternative to Algolia and an easier-to-use, batteries-included alternative to ElasticSearch. Get a quick overview from [this guide](https://typesense.org/guide/).

## Usage

Read detailed step-by-step instructions on how to configure and setup the scraper on Typesense's dedicated documentation site: https://typesense.org/docs/latest/guide/docsearch.html

## Compatibility

| typesense-docsearch-scraper | typesense-server |
| --- | --- |
| 0.5.0 | >= 0.22.1 |
| 0.4.x and below | >= 0.21.0  |

## Development Workflow

This section only applies if you're making changes to this scraper itself. If you only need to run the scraper, see Usage instructions above.

#### Releasing a new version

Basic/abbreviated instructions:

```shellsession
$ pipenv shell
$ ./docsearch docker:build
$ git tag -a 0.2.1 -m "0.2.1"
$ ./docsearch deploy:scraper
$ git push --follow-tags
```

Detailed instructions starting from a fresh Ubuntu Server 22.02:

```bash
# Install Docker:
# https://docs.docker.com/engine/install/ubuntu/
sudo apt update
sudo apt remove docker docker-engine docker.io containerd runc --yes
sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    --yes
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin \
  --yes
sudo docker run hello-world

# Run Docker as a non-root user:
# https://www.digitalocean.com/community/questions/how-to-fix-docker-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socket
sudo usermod -aG docker ${USER}
exit
# (Relogin.)
docker run hello-world

# Install dependencies for pyenv:
# https://github.com/pyenv/pyenv/wiki#suggested-build-environment
sudo apt update
sudo apt install \
  build-essential \
  curl \
  libbz2-dev \
  libffi-dev \
  liblzma-dev \
  libncursesw5-dev \
  libreadline-dev \
  libsqlite3-dev \
  libssl-dev \
  libxml2-dev \
  libxmlsec1-dev \
  llvm \
  make \
  tk-dev \
  wget \
  xz-utils \
  zlib1g-dev \
  --yes

# Install pyenv:
# https://github.com/pyenv/pyenv#automatic-installer
curl https://pyenv.run | bash

# Add pyenv to path:
echo >> ~/.bashrc
echo '# Adding pyenv' >> ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

# Install Python 3.10 inside pyenv:
pyenv install 3.10

# Set the active version of Python:
pyenv local 3.10

# Upgrade pip:
pip install --upgrade pip

# Install pipenv:
pip install --user pipenv

# There will be a warning:
# "The script virtualenv-clone is installed in '/home/[username]/.local.bin' which is not on PATH."
# Fix the warning by adding it to the PATH:
echo >> ~/.bashrc
echo '# Fixing pip warning' >> ~/.bashrc
echo 'PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc

# Ensure that you are in the "typesense-docsearch-scraper" directory.
# Then, install the Python dependencies for this project:
pipenv --python 3.10
pipenv lock --clear
pipenv install

# Then, open a shell with with the Python environment:
pipenv shell

# Build a new version of the Docker container.
export TAG="0.4.1"
docker buildx build -f ./scraper/dev/docker/Dockerfile -t typesense/docsearch-scraper:${TAG} .
docker push typesense/docsearch-scraper:${TAG}
docker tag typesense/docsearch-scraper:${TAG} typesense/docsearch-scraper:latest
docker push typesense/docsearch-scraper:latest

# Add a new Git tag.
git tag -a "$TAG" -m "$TAG"

# Sync with GitHub.
git push --follow-tags
```

## Help

If you have any questions or run into any problems, please create a Github issue and we'll try our best to help.
