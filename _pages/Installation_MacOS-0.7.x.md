---
layout: default
title: Ada Installation Guide (MacOS) - Version 0.7.x
custom_css:
- main.css
permalink: /installation/macos/0_7_x
---

<p>&nbsp;</p>
<p>&nbsp;</p>

# Ada Installation Guide (MacOS) - Version 0.7.x

(Expected time: 30-45 mins)

&nbsp;

### 0. Preparation

Recommended OS: *MacOS* 10.14.4

Recommended resources:

* **Ada Server**: 8 GB RAM, 8 CPUs, 40 GB disc space
* **Mongo DB**: 8 GB RAM, 4 CPUs, 100 GB disc space
* **Elastic Search DB**: 8 GB RAM, 4 CPUs, 100 GB disc space

&nbsp; 

### 1. **Java** 1.8

```sh
brew cask install java8
```

&nbsp;

### 2. **Mongo** DB
* Install MongoDB (3.2.9)

```sh
curl -L -O http://downloads.mongodb.org/osx/mongodb-osx-x86_64-3.2.9.tgz
tar -xvf mongodb-osx-x86_64-3.2.9.tgz
mv mongodb-osx-x86_64-3.2.9 mongodb
mkdir -p /data/db
sudo chown -R `id -un` /data/db
```

* Configure memory and other settings in `/usr/local/etc/mongod.conf`
(set a reasonable `cacheSizeGB`, recommended to 50% of available RAM, [ref](https://docs.mongodb.com/v3.2/reference/configuration-options/#storage.wiredTiger.engineConfig.cacheSizeGB))

```sh
  ...
  wiredTiger:
    engineConfig:
      cacheSizeGB: 5
      journalCompressor: none
    collectionConfig:
      blockCompressor: snappy
```

* Create a limits file `/etc/security/limits.d/mongodb.conf`
 

```sh
mongodb    soft    nofile          1625538
mongodb    hard    nofile          1625538
mongodb    soft    nopro           64000
mongodb    hard    nopro           64000
mongodb    soft    memlock         unlimited
mongodb    hard    memlock         unlimited
mongodb    soft    fsize           unlimited
mongodb    hard    fsize           unlimited
mongodb    soft    cpu             unlimited
mongodb    hard    cpu             unlimited
mongodb    soft    as              unlimited
mongodb    hard    as              unlimited
```
* To force your new limits to be loaded log out of all your current sessions and log back in

* Add MongoDB installation folder to PATH environment variable `/etc/paths`

```
<mongo_installation_folder>/bin
```

* Start Mongo

```sh
mongod
```

* To check  if everything works as expected see the log file `/usr/local/var/log/mongodb`.
* *Recommendation*: For convenient DB exploration and query execution install [Robomongo (Robo3T)](https://robomongo.org/download) UI client.

* Optionally set up users with authentication as described [here](https://docs.mongodb.com/v3.2/tutorial/create-users/).
* For tuning tips go to [here](https://www.percona.com/blog/2016/08/12/tuning-linux-for-mongodb).

&nbsp; 

## 3. **Elastic Search**

* Install ES (2.3.4)

```sh
curl -L -O https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.4/elasticsearch-2.3.4.tar.gz
tar -xvf elasticsearch-2.3.4.tar.gz
mv elasticsearch-2.3.4 elasticsearch
```

* Modify the configuration in `<es_installation_folder>/config/elasticsearch.yml`

```sh
  cluster.name: ada-cluster       (if not changed "elasticsearch" is used by default)
  bootstrap.mlockall: true
  network.host: x.x.x.x           (set to a non-localhost ip address if the db should be accessible within a newtwork)
  threadpool:
      index:
          size: 30
          queue_size: 8000
      search:
          size: 30
          queue_size: 8000 
      bulk:
          size: 10
          queue_size: 500
  index.query.bool.max_clause_count: 4096
```

* If you want to store data in a special directory set also
 
```
path.data: /your_custom_path
```
(Note that you need to make `your_custom_path` writeable for the `elasticsearch` user

* Create a limits file `/etc/security/limits.d/elasticsearch.conf`

```sh
elasticsearch    soft    nofile          1625538
elasticsearch    hard    nofile          1625538
elasticsearch    soft    memlock         unlimited
elasticsearch    hard    memlock         unlimited
```

* To force your new limits to be loaded log out of all your current sessions and log back in

* Configure memory constraints in `/etc/default/elasticsearch`
(set a reasonable `ES_HEAP_SIZE`; recommended to 50% of available RAM, but no more than 31g)


```
ES_HEAP_SIZE=5g
MAX_OPEN_FILES=1625538
```

* Add Elastic Search to PATH environment variable `/etc/paths`

~~~
<es_installation_folder>/bin
~~~

* Start ES
```
elasticsearch
```

* To check if everything works as expected see the log file(s) at `/usr/local/var/log/elasticsearch` and/or curl the server info by `curl -XGET localhost:9200`.

* *Recommendation*: For convenient DB exploration and query execution install the `kopf` plugin:

```sh
<es_installation_folder>/bin/plugin install lmenezes/elasticsearch-kopf/v2.1.1
```
(Kopf ES web client is then accessible at [http://localhost:9200/_plugin/kopf](http://localhost:9200/_plugin/kopf)) 

&nbsp;

### 4. Application Server (Netty)

* Download the latest version e.g., 0.7.3 from (the password is: "ada2019")

[https://owncloud.lcsb.uni.lu/s/h5HJykkj2ftU0lO](https://owncloud.lcsb.uni.lu/s/h5HJykkj2ftU0lO)

* Unzip the server binaries

```sh
sudo apt-get install unzip
unzip ada-web-$VERSION.zip
cd ada-web-$VERSION/bin
```

* Create a temp folder wherever you want,  e.g.

```
mkdir ada_temp
```

* Set the path and memory constraint variables `ADA_TEMP` and `ADA_MEM` in `runme` script (Ada's `bin` folder).

* Configure DB connections in `set_env.sh` (`bin` folder)

1 . *Mongo*
     
if not password-protected (default) set `ADA_MONGO_DB_URI`

```sh
export ADA_MONGO_DB_URI="mongodb://localhost:27017/ada?rm.nbChannelsPerNode=20"
```
     
 if password-protected set the following variables


```
export ADA_MONGO_DB_HOST=x.x.x.x:27017
export ADA_MONGO_DB_NAME="ada"
export ADA_MONGO_DB_USERNAME="xxx_user"
export ADA_MONGO_DB_PASSWORD="XXX"
```

2 . *Elastic Search*

if non-localhost server is used set `ADA_ELASTIC_DB_HOST`

```
export ADA_ELASTIC_DB_HOST=x.x.x.x:9300
```

Also set the custom ES cluster name if it was configured in Section 3 (see `/etc/elasticsearch/elasticsearch.yml`) 

```
export ADA_ELASTIC_DB_CLUSTER_NAME="ada-cluster"
```

3 . *General Setting*


* Configure `project`, `ldap` (to enable access **without** authentication), and `datasetimport` in `custom.conf` (Ada `conf` folder) 

```sh
project {
  name = "Ultimate"
  url = "https://ada-discovery.org"
  logo = "images/logos/ada_logo_v4.png"
}

ldap {
  mode = "local"
  port = "65505"
  debugusers = true
}

datasetimport.import.folder = "/custom_path"
```

* Switch a Netty transport implementation to `jdk` (in `custom.conf`), which is required for MacOS deployments

```sh
play.server.netty.transport = "jdk"
```

* Optionally if you want to use external images not shipped by default with Ada you must register your resource folder(s) placed in the Ada root by editing `custom.conf`

```sh
assets.external_paths = ["/folder_in_classpath"]
```
(*Warning*: Never set any of the folders to "/" since this would make all the classpath files, including common configuration files, accessible from outside as application assets. An example: https://localhost:8080/assets/application.conf)

Now you can use the logo images placed in your `folder_in_classpath` by adapting `custom.conf` for instance as

```
project {
  ....
  logo = "syscid_large.png"
}

footer.logos = [
  {url: "https://www.aetionomy.eu", logo: "aetionomy.png", height: 150},
  {url: "https://www.efpia.eu", logo: "efpia_logo.png"}
]
```
(Note an optional `height` attribute)

* Start Ada (Netty)

```sh
./runme
```
(if cannot be executed `chmod +x runme` might be needed)

* To check if everything works as expected see the log file at `$ADA_ROOT/bin/logs/application.log`

* You can now access Ada at

```sh
http://localhost:8080
```

* To login as admin go to

```sh
http://localhost:8080/loginAdmin
```

* Set your custom `Homepage`, `Contact`, and `Links` in *Admin  →  HTML Snippets → [+]*

`Homepage` example:
```html
<p>
	Ada provides key infrastructure for secured integration, visualization,
	and analysis of heterogeneous clinical and experimental data generated
	during the <a href="http://www.project_url_to_set.com">My TODO project</a>.
</p>
<p>
	My TODO research project focuses on improving ...
</p>
<button type="button" class="btn btn-default btn-sm" data-toggle="collapse" data-target="#more-info">Read More</button>
<div id="more-info" class="collapse">
	<blockquote>
		<h4>The platform currently ...</h4>
	</blockquote>
</div>
<br/>
```
 
`Contact` example:
```html
<strong>Dr. John Snow</strong></br>
Winterfell Team</br>
Westeros Centre For Systems Biomedicine (WCSB)</br>
University of Seven Kingdoms</br></br>
<i class="glyphicon glyphicon-envelope"></i>
<a href="mailto:john.snow@north.edu?Subject=Ada Question">john.snow@north.edu</a><br>
<i class="glyphicon glyphicon-chevron-right"></i>
<a target="_blank" href="http://www.north.edu/wcsb">www.north.edu/wcsb</a><br>
```

`Links` example:
```html
<li><a href="https://www.project-redcap.org">RedCap</a></li>
<li><a href="https://www.synapse.org">Synapse</a></li>
<li role="separator" class="divider"></li>
<li><a href="https://uni.lu/lcsb">LCSB Home</a></li>
```

 &nbsp; 

### 5. LDAP

* If your Ada is accessible from outside/the internet or having two users (admin: `/loginAdmin` and basic `/loginBasic`) without authentication is not sufficient you **must** configure LDAP users with authentication
 
* Configure LDAP in `set_env.sh`

```
export ADA_LDAP_HOST="XXX"
export ADA_LDAP_BIND_PASSWORD="XXX"
```
  
* Go to `custom.conf`and remove `mode` (default: remote) and `port` (default: 389) and configure the LDAP groups, example: 

```sh
ldap {
  dit = "cn=users,cn=accounts,dc=north,dc=edu"
  groups = ["cn=my-group-name,cn=groups,cn=accounts,dc=north,dc=edu"]
  bindDN = "uid=my-ldap-reader-xxx,cn=users,cn=accounts,dc=north,dc=edu"
  debugusers = true
}
```
* Restart Ada

```sh
./stopme
```
and
```sh
./runme
```
* Log in to Ada

* Import LDAP users by clicking
*Admin → User Actions → Import from LDAP*

* Assign admin role to at least one user (yourself)
*Admin → Users → double click on a user → Roles → [+] → write `admin` → Update*

* Disable `debugusers` by removing it from `custom.conf` or setting it to `false`

* Restart Ada

* Check if `debugusers` have been disabled by trying to access `http://localhost:8080/loginAdmin` (must not allow you in)

* Login using your LDAP username and password

* Start by importing your first data set in *Admin →  Data Set Imports → [+]*

* Have fun!
