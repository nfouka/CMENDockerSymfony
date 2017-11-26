1) [PHP-FPM 7.0, Apache 2.4 + mod_proxy_fcgi, Symfony 3.0, MySQL 5.7, phpMyAdmin](php-fpm/README.md)

2) [PHP 5.6, Apache 2.4 + mod_php, Symfony 3, MySQL 5.7, phpMyAdmin](php/README.md)




<h1 class="article-titre">Développer une application Symfony avec Docker</h1>

                <div class="article-meta">
                    Publié le 27/04/2016.
                                            Dernière mise à jour le 17/07/2016.
                                    </div>
            </header>

            <p>Cet article explique comment mettre en place une application Symfony avec Docker. L'application qui va nous servir de test possède comme couche technique : PHP 5.6, Apache 2, Symfony 3, MySQL 5.7 et phpMyAdmin. Toutes les procédures d'installations décrites dans l'article se font sur Ubuntu 15.10.</p>

<h2>Note de l'auteur</h2>
<p>Le code contenu dans cet article peut être trouvé sur <a href="https://github.com/cmen/CMENDockerSymfony/tree/master/php" target="_blank">github</a>.</p>

<p>D'un point de vue technique si vous le pouvez, je vous recommande plutôt de suivre <a href="http://www.christophe-meneses.fr/article/developper-une-application-symfony-avec-docker-version-php-fpm" target="_blank">mon article plus récent</a> car la couche technique utilisée est plus performante (<strong>PHP-FPM</strong> avec le module Apache <strong>mod_proxy_fcgi</strong> à la place de <strong>PHP</strong> avec le module Apache <strong>mod_php</strong>). Si vous ne connaissez pas PHP-FPM, vous utilisez probablement PHP avec le module Apache mod_php (méthode qui est utilisée dans cet article ici). Avant de poursuivre votre lecture, je vous invite à aller lire mon article sur <a href="http://www.christophe-meneses.fr/article/mettre-en-place-un-environnement-apache-php-fpm" target="_blank">comment mettre en place un environnement Apache / PHP-FPM</a>. Vous trouverez les différents avantages et inconvénients de chaque méthode.</p>

<h2>Installation de Docker</h2>

<h3>Prérequis</h3>
<p>Docker a besoin au minimum du kernel Linux 3.10. Pour vérifier le numéro de version de votre kernel, tapez la commande :</p>
<pre><code class="bash hljs">uname -r
</code>
</pre>

<p>Une fois la vérification faite, on commence par mettre à jour la liste des dépôts :</p>
<pre><code class="bash hljs">sudo apt-get update
</code>
</pre>

<p>Il faut s'assurer que APT fonctionne avec HTTPS et que les certificats CA sont bien installés :</p>
<pre><code class="bash hljs">sudo apt-get install apt-transport-https ca-certificates
</code>
</pre>

<p>On peut ensuite rajouter une clé GPG :</p>
<pre><code class="bash hljs">sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
</code>
</pre>

<p>Ouvrez le fichier <strong>/etc/apt/sources.list.d/docker.list</strong>. Il faut supprimer tout le contenu de ce fichier. S'il n'existe pas, il faut le créer. Ensuite, il faut y ajouter une ligne en fonction de la version de votre Ubuntu (voir le <a href="https://docs.docker.com/engine/installation/linux/ubuntulinux/" target="_blank">site officiel</a>). Par exemple pour la version 15.10 :</p>
<pre><code class="bash hljs">deb https://apt.dockerproject.org/repo ubuntu-wily main
</code>
</pre>

<p>Puis sauvegardez le fichier. Vous pouvez maintenant relancer une mise à jour des dépôts : </p>
<pre><code class="bash hljs">sudo apt-get update
</code>
</pre>

<p>Purgez les anciens dépôts s'ils existent :</p>
<pre><code class="bash hljs">sudo apt-get purge lxc-docker
</code>
</pre>

<p>Vérifiez que APT utilise les bons dépôts : </p>
<pre><code class="bash hljs">sudo apt-cache policy docker-engine
</code>
</pre>

<p>Pour cette version d'Ubuntu, il est recommandé d'installer le packet <em>linux-image-extra</em> :</p>
<pre><code class="bash hljs">sudo apt-get install linux-image-extra-$(uname -r)
</code>
</pre>

<h3>Installation de Docker Engine</h3>

<p>Pour installer Docker, tapez la commande :</p>
<pre><code class="bash hljs">sudo apt-get install docker-engine
</code>
</pre>

<p>Démarrez le service Docker :</p>
<pre><code class="bash hljs">sudo service docker start
</code>
</pre>

<p>Pour tester si l'installation s'est effectuée correctement, vous pouvez lancer le programme de test <em>hello-world</em> :</p>
<pre><code class="bash hljs">sudo docker run hello-world
</code>
</pre>

<p>Pour éviter de devoir taper à chaque fois sudo devant les commandes docker, il faut faire quelques actions supplémentaires. Créez un groupe d'utilisateur <strong>docker</strong> et ajoutez-y votre login :
</p><pre><code class="bash hljs">sudo usermod -aG docker votre_login 
</code>
</pre>

<p>Ensuite déconnectez-vous de votre session puis reconnectez-vous. Lors de son démarrage, le service docker va donner les accès de lecture et d'écriture au groupe <strong>docker</strong> (dont vous faites parti). Maintenant si vous tapez une commande docker sans le sudo, la commande peut s'exécuter correctement. Par exemple :</p>
<pre><code class="bash hljs">docker run hello-world
</code>
</pre>

<h3>Installation de Docker Compose</h3>

<p>Docker Compose est un outil qui permet de configurer ses conteneurs via un fichier yml. On évite ainsi de taper X commandes docker.</p>

<p>Allez sur la page github de <a href="https://github.com/docker/compose/releases" target="_blank">Docker compose</a>, et suivez les instructions pour installer la bonne version. Pour la version 1.6.2 :</p>
<pre><code class="bash hljs">curl -L https://github.com/docker/compose/releases/download/1.6.2/docker-compose-`uname -s`-`uname -m` &gt; /usr/<span class="hljs-built_in">local</span>/bin/docker-compose
chmod +x /usr/<span class="hljs-built_in">local</span>/bin/docker-compose
</code>
</pre>

<p>Puis testez l'installation :</p>
<pre><code class="bash hljs">docker-compose --version
</code>
</pre>

<h2>«Dockerisation » de l'application Symfony</h2>

<h3>Objectif</h3>

<p>Attention, ici je ne vais pas expliquer comment fonctionne Docker. Si vous débutez, je vous invite à aller lire la documentation bien fournie sur le <a href="https://docs.docker.com" target="_blank">site officiel</a>.</p> 

<p>On va découper l'application en 5 conteneurs : <br>
1) Un conteneur web : PHP + Apache.<br>
2) Un conteneur base de données : MySQL.<br>
3) Un conteneur phpMyAdmin.<br>
4) Un conteneur donnée qui va stocker le projet Symfony.<br>
5) Un conteneur donnée qui va stocker les données de la base de données.
</p>

<p>Pour cela, on va commencer par créer les images qui vont être utilisées pour initialiser ces conteneurs.</p>

<h3>Création des images</h3>

<p>On crée un répertoire <strong>Docker</strong> dans lequel on crée un répertoire par image. Chaque répertoire va contenir le <strong>Dockerfile</strong> de l'image. C'est visible <a href="https://github.com/cmen/CMENDockerSymfony/tree/master/php/Docker" target="_blank">ici sur github</a>.</p>

<h4>Image n°1</h4>
<pre><code class="hljs dockerfile"><span class="hljs-keyword">FROM</span> ubuntu:<span class="hljs-number">15.10</span>

<span class="hljs-keyword">MAINTAINER</span> Christophe Meneses

<span class="hljs-keyword">RUN</span><span class="bash"> <span class="hljs-built_in">echo</span> <span class="hljs-string">"deb http://ppa.launchpad.net/ondrej/php5-5.6/ubuntu wily main"</span> | tee -a /etc/apt/sources.list \ 
</span>    &amp;&amp; apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <span class="hljs-number">4</span>F4EA0AAE5267A6C \ 
    &amp;&amp; apt-get update

<span class="hljs-keyword">RUN</span><span class="bash"> apt-get install -y apache2 
</span>
<span class="hljs-keyword">RUN</span><span class="bash"> apt-get install -y php5-common php5-cli libapache2-mod-php5
</span>
<span class="hljs-keyword">RUN</span><span class="bash"> apt-get install -y php5-mcrypt php5-mysql php5-apcu php5-curl php5-intl php5-xdebug
</span>
<span class="hljs-keyword">ADD</span><span class="bash"> php-custom.ini /etc/php5/apache2/conf.d
</span><span class="hljs-keyword">ADD</span><span class="bash"> 20-xdebug.ini /etc/php5/apache2/conf.d
</span><span class="hljs-keyword">ADD</span><span class="bash"> php-custom.ini /etc/php5/cli/conf.d
</span><span class="hljs-keyword">ADD</span><span class="bash"> 20-xdebug.ini /etc/php5/cli/conf.d
</span>
<span class="hljs-keyword">RUN</span><span class="bash"> php -r <span class="hljs-string">"readfile('https://getcomposer.org/installer');"</span> | php -- --install-dir=/usr/<span class="hljs-built_in">local</span>/bin --filename=composer \
    &amp;&amp; chmod +x /usr/<span class="hljs-built_in">local</span>/bin/composer
</span>
<span class="hljs-keyword">RUN</span><span class="bash"> apt-get install -y git
</span>
<span class="hljs-keyword">EXPOSE</span> <span class="hljs-number">80</span>

<span class="hljs-keyword">WORKDIR</span><span class="bash"> /etc/php5
</span>
<span class="hljs-keyword">CMD</span><span class="bash"> /usr/sbin/apache2ctl -D FOREGROUND
</span></code>
</pre>

<h4>Image n°2</h4>
<pre><code class="hljs dockerfile"><span class="hljs-keyword">FROM</span> mysql:<span class="hljs-number">5.7</span>

<span class="hljs-keyword">MAINTAINER</span> Christophe Meneses
</code>
</pre>

<h4>Image n°3</h4>
<pre><code class="hljs dockerfile"><span class="hljs-keyword">FROM</span> phpmyadmin/phpmyadmin:<span class="hljs-number">4.6</span>.<span class="hljs-number">0</span>-<span class="hljs-number">1</span>

<span class="hljs-keyword">MAINTAINER</span> Christophe Meneses
</code>
</pre>

<h4>Image n°4</h4>
<pre><code class="hljs dockerfile"><span class="hljs-keyword">FROM</span> ubuntu:<span class="hljs-number">15.10</span>

<span class="hljs-keyword">MAINTAINER</span> Christophe Meneses

<span class="hljs-keyword">VOLUME</span><span class="bash"> /var/www/html
</span></code>
</pre>

<h4>Image n°5</h4>
<pre><code class="hljs crystal">FROM <span class="hljs-symbol">ubuntu:</span><span class="hljs-number">15.10</span>

MAINTAINER Christophe Meneses

VOLUME /var/<span class="hljs-class"><span class="hljs-keyword">lib</span>/<span class="hljs-title">mysql</span></span>
</code>
</pre>

<p>Une fois les fichiers Dockerfile prêts, il faut demander à Docker de créer les images. Pour cela, mettez-vous dans le répertoire <strong>Docker</strong> et lancez à la suite les commandes ci-dessous :</p>
<pre><code class="bash hljs">docker build -t cmeneses/apache_php:2-5.6 ./apache2_php5.6/
docker build -t cmeneses/data_application ./data_application/
docker build -t cmeneses/data_mysql ./data_mysql/
docker build -t cmeneses/mysql:5.7 ./mysql5.7/
docker build -t cmeneses/phpmyadmin:4.6 ./phpmyadmin4.6/
</code>
</pre>

<p>Pour lister les images disponibles sur votre machine, tapez la commande : </p>
<pre><code class="bash hljs">docker images
</code>
</pre>

<p>Les images crées précédemment doivent apparaitre :</p>
<pre><code class="bash hljs">REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
cmeneses/data_application   latest              ab7eb6a8ba30        6 days ago          135.9 MB
cmeneses/phpmyadmin            4.6              5c0b3987b89c        6 days ago          57.57 MB
cmeneses/mysql                 5.7              2cf6f84ca859        6 days ago          361.2 MB
cmeneses/data_mysql         latest              356bb880ef1e        6 days ago          135.9 MB
cmeneses/apache_php          2-5.6              4435d4b78463        6 days ago          298 MB
</code>
</pre>

<h3>Création du fichier docker-compose.yml</h3>

<p>Le fichier <strong>docker-compose.yml</strong> est à créer à la racine du projet Symfony. Il va contenir toute la configuration des conteneurs du projet et leurs relations. Ainsi, on pourra lancer la construction de l'ensemble des conteneurs avec une seule commande.</p>

<pre><code class="hljs dts"><span class="hljs-symbol">version:</span> <span class="hljs-string">'2'</span>
<span class="hljs-symbol">services:</span>
<span class="hljs-symbol">    test_application:</span>
<span class="hljs-symbol">        image:</span> cmeneses/data_application
<span class="hljs-symbol">        volumes:</span>
            - <span class="hljs-meta-keyword">/xxx/</span>Test-Docker-Sf/Symfony:<span class="hljs-meta-keyword">/var/</span>www/html
<span class="hljs-symbol">    test_data:</span>
<span class="hljs-symbol">        image:</span> cmeneses/data_mysql
<span class="hljs-symbol">        volumes:</span>
            - <span class="hljs-meta-keyword">/xxx/</span>Test-Docker-Sf/Data:<span class="hljs-meta-keyword">/var/</span>lib/mysql
<span class="hljs-symbol">    test_database:</span>
<span class="hljs-symbol">        image:</span> cmeneses/mysql:<span class="hljs-number">5.7</span>
<span class="hljs-symbol">        ports:</span>
            - <span class="hljs-number">60001</span>:<span class="hljs-number">3306</span>
<span class="hljs-symbol">        environment:</span>
<span class="hljs-symbol">            MYSQL_ROOT_PASSWORD:</span> root
<span class="hljs-symbol">            MYSQL_DATABASE:</span> test
<span class="hljs-symbol">            MYSQL_USER:</span> test
<span class="hljs-symbol">            MYSQL_PASSWORD:</span> test
<span class="hljs-symbol">        volumes_from:</span>
            - test_data
<span class="hljs-symbol">    test_phpmyadmin:</span>
<span class="hljs-symbol">        image:</span> cmeneses/phpmyadmin:<span class="hljs-number">4.6</span>
<span class="hljs-symbol">        ports:</span>
            - <span class="hljs-number">60002</span>:<span class="hljs-number">80</span>
<span class="hljs-symbol">        links:</span>
            - test_database:db
<span class="hljs-symbol">        environment:</span>
            - PMA_USER=root
            - PMA_PASSWORD=root
<span class="hljs-symbol">    test_web:</span>
<span class="hljs-symbol">        image:</span> cmeneses/apache_php:<span class="hljs-number">2</span><span class="hljs-number">-5.6</span>
<span class="hljs-symbol">        ports:</span>
            - <span class="hljs-number">60000</span>:<span class="hljs-number">80</span>
<span class="hljs-symbol">        environment:</span>
<span class="hljs-symbol">            XDEBUG_CONFIG:</span> remote_host=<span class="hljs-number">172.22</span><span class="hljs-number">.0</span><span class="hljs-number">.1</span>
<span class="hljs-symbol">        volumes_from:</span>
            - test_application
<span class="hljs-symbol">        links:</span>
            - test_database
<span class="hljs-symbol">        working_dir:</span> <span class="hljs-meta-keyword">/var/</span>www/html
</code>
</pre>

<p>Les adresses des volumes sur la machine hôte doivent être modifiées (c'est en fonction de votre installation). Le reste est à personnaliser en fonction de vos besoins.</p>

<h3>Lancement de l'application</h3>

<p>Placez-vous dans le dossier du projet Symfony, et lancez la commande :</p>

<pre><code class="bash hljs">docker-compose build
</code>
</pre>

<p>Cette commande va construire l'ensemble du projet en lisant le fichier <strong>docker-compose.yml</strong>. Comme on a déjà construit nous-même les images à l'étape précédente, la commande va nous dire qu'il n'y a plus rien à faire.</p>

<p>Il ne reste plus qu'à démarrer l'ensemble des containers : </p>
<pre><code class="bash hljs">docker-compose up
</code>
</pre>

<p>Voilà le projet est lancé ! Cependant avant de pouvoir accéder à l'application Symfony, il peut être nécessaire de lancer un <strong>composer install</strong> pour télécharger tous les vendors. Pour cela, on doit d'abord récupérer l'identifiant de notre conteneur <em>test_web</em>. Tapez la commande : </p>

<pre><code class="bash hljs">docker ps
</code>
</pre>

<p>Dans les lignes affichées, repérez le container <em>test_web</em> et récupérez son identifiant :</p>
<pre><code class="bash hljs">CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS                    PORTS               NAMES
22e642982db4        cmeneses/apache_php:2-5.6    <span class="hljs-string">"/bin/sh -c '/usr/sbi"</span>   6 days ago          Exited (137) 6 days ago                       symfony_test_web_1
</code>
</pre>

<p>Vous pouvez ainsi lancer une commande à l'intérieur de celui-ci. Si vous regardez bien le fichier <strong>docker-compose.yml</strong>, on a indiqué un répertoire de travail (<strong>working_dir</strong>). Cela veut dire que toute les commandes seront lancées directement dans <strong>/var/www/html</strong>, c'est à dire à la racine du projet Symfony. On peut donc aisément utiliser composer (ou quelle que soit la commande Symfony) :</p>
<pre><code class="bash hljs">docker <span class="hljs-built_in">exec</span> 22e642982db4 composer install
docker <span class="hljs-built_in">exec</span> 22e642982db4 php bin/console cache:clear
</code>
</pre>

<p>L'application Symfony est accessible à l'adresse <a href="http://localhost:60000/web/app_dev.php" target="_blank">http://localhost:60000/web/app_dev.php</a>, phpMyAdmin est accessible à l'adresse <a href="http://localhost:60002" target="_blank">http://localhost:60002</a>, MySQL est accessible à l'adresse <a href="http://localhost:60001" target="_blank">http://localhost:60001</a>.

</p><p>Vous pouvez également lancer le projet en tâche de fond. Il faut rajouter l'option <strong>-d</strong> : </p>
<pre><code class="bash hljs">docker-compose up -d
</code>
</pre>

<p>Pour arrêter l'ensemble des conteneurs du projet, il faut taper la commande : </p>
<pre><code class="bash hljs">docker-compose stop
</code>
</pre>

<h3>Modifications au niveau de Symfony</h3>

<p>Il reste quelques actions à faire pour que l'application fonctionne correctement.</p>

<p>Pour que Symfony est accès à la base de données, il faut indiquer le nom du conteneur de la base de données dans le fichier <strong>parameters.yml</strong>. Par exemple ici, on doit indiquer <em>test_database</em> :</p>

<pre><code class="yaml hljs"><span class="hljs-attr">database_host:</span> <span class="hljs-string">test_database</span>
</code>
</pre>

<p>Si vous essayer d'accéder à <strong>app_dev.php</strong>, vous aurez l'erreur suivante : <em>You are not allowed to access this file. Check app_dev.php for more information.</em>. La solution est de commenter les lignes ci-dessous dans  <strong>app_dev.php</strong> :</p>

<pre><code class="php hljs"><span class="hljs-keyword">if</span> (<span class="hljs-keyword">isset</span>($_SERVER[<span class="hljs-string">'HTTP_CLIENT_IP'</span>])
    || <span class="hljs-keyword">isset</span>($_SERVER[<span class="hljs-string">'HTTP_X_FORWARDED_FOR'</span>])
    || !in_array(@$_SERVER[<span class="hljs-string">'REMOTE_ADDR'</span>], <span class="hljs-keyword">array</span>(<span class="hljs-string">'127.0.0.1'</span>, <span class="hljs-string">'fe80::1'</span>, <span class="hljs-string">'::1'</span>))
) {
    header(<span class="hljs-string">'HTTP/1.0 403 Forbidden'</span>);
    <span class="hljs-keyword">exit</span>(<span class="hljs-string">'You are not allowed to access this file. Check '</span>.basename(<span class="hljs-keyword">__FILE__</span>).<span class="hljs-string">' for more information.'</span>);
}
</code>
</pre>

<p>C'est terminé ! Normalement tout est OK maintenant pour commencer les développements.</p>

<h2>Autres commandes utiles</h2>

<p>Vous pouvez supprimer tous les conteneurs d'un coup :</p>
<pre><code class="bash hljs">docker rm $(docker ps -a -q)
</code>
</pre>

<p>Vous pouvez supprimer toutes les images d'un coup :</p>
<pre><code class="bash hljs">docker rmi $(docker images -q)
</code>
</pre>

<h2>Version</h2>
<p>Cet article a été testé avec : </p>
<blockquote>
Ubuntu : 15.10<br>
Docker Engine : 1.11<br>
Docker Compose : 1.6<br>
Symfony : 3.0<br>
PHP : 5.6<br>
MySQL : 5.7<br>
phpMyAdmin : 4.6
</blockquote>
