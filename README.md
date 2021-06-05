# Postgres Configurer Ssl

Créer les certificats nécessaires avec OpenSSL pour activer l'authentification par certificat client au serveur de base de données.

## Création des Certificats

### Préparation

1. Définir les constantes

    ##### Linux
    >```bash
    >repertoire_certificats="/tmp/certificats"
    >repertoire_certificat_client="${repertoire_certificats}/client"
    >repertoire_certificats_serveur="/etc/postgresql/13/main"
    >sujet_certificat_autorite_de_confiance="/CN=MonAutoriteDeConfiance"
    >nom_hote=$(hostname)
    >nom_utilisateur=$(whoami)
    >nom_base_de_donnees=postgres
    >mot_de_passe_certificat_client="mdp_client"
    >```

2. Créer un répertoire pour sauvegarder les certificats

    ##### Linux
    >```bash
    >mkdir $repertoire_certificats
    >mkdir $repertoire_certificat_client
    >```

### Certificat de l'autorité de confiance

>```bash
>pushd $repertoire_certificats
>openssl req -x509 -extensions v3_ca -sha512 -nodes -days 365 -newkey rsa:4096 -keyout bdd_ca.key -out bdd_ca.crt -passout pass:serveur -subj "$sujet_certificat_autorite_de_confiance"
>```
    

### Certificats pour le serveur de base de données

1. Créer un fichier de configuration pour les certificats de serveur

    'localhost' est rajouté à la liste des dns pour le [développement de l'application](https://letsencrypt.org/docs/certificates-for-localhost/#making-and-trusting-your-own-certificates) et doit être supprimé pour la mise en production.

    ##### Contenu
    >```bash
    >basicConstraints=CA:FALSE
    >extendedKeyUsage=serverAuth
    >subjectAltName=DNS.1:[nom_hote],DNS.2:[nom_hote].local,DNS.3:localhost
    >```

    ##### Linux
    >```bash
    >echo -e "basicConstraints=CA:FALSE\nextendedKeyUsage=serverAuth\nsubjectAltName=DNS.1:${nom_hote},DNS.2:${nom_hote}.local,DNS.3:localhost" > sslServeurExtConfig.txt
    >```

2. Créer le certificat

    >```bash
    >openssl req -new -newkey rsa:4096 -nodes -sha512 -days 365 -out serveur.csr -keyout serveur.key -subj "/CN=$nom_hote"
    >openssl x509 -req -sha512 -days 365 -in serveur.csr -out serveur.crt -CA bdd_ca.crt -CAkey bdd_ca.key -CAcreateserial -CAserial ca.srl -extfile sslServeurExtConfig.txt
    >popd
    >```

3. Copier le certificat dans le répertoire de la base de données

    ##### Linux
    >```bash
    >cp -r ${repertoire_certificats}/bdd_ca.crt ${repertoire_certificats}/serveur.* $repertoire_certificats_serveur
    >```

4. Configurer les droits d'accès des certificats et vérifier que les fichiers appartiennent à l'utilisateur _postgres_

    ##### Linux
    >```bash
    >chown postgres ${repertoire_certificats_serveur}/serveur.* $repertoire_certificats_serveur/bdd_ca.crt
    >chmod 600 ${repertoire_certificats_serveur}/serveur.* ${repertoire_certificats_serveur}/bdd_ca.crt
    >```


### Certificats pour le client de base de données

1. Créer un fichier de configuration pour les certificats de serveur

    #### Contenu
    >```bash
    >basicConstraints=CA:FALSE
    >extendedKeyUsage=clientAuth
    >```

    ##### Linux
    >```bash
    >pushd $repertoire_certificat_client
    >echo -e "basicConstraints=CA:FALSE\nextendedKeyUsage=clientAuth" > sslClientExtConfig.txt
    >```

2. Créer le certificat et l'exporter (pfx & pk8)
    >```bash
    >openssl req -new -newkey rsa:4096 -nodes -sha512 -days 365 -out "$nom_utilisateur.csr" -keyout "$nom_utilisateur.key" -subj "/CN=$nom_utilisateur"  
    >openssl x509 -req  -sha512 -days 365 -in "$nom_utilisateur.csr" -out "$nom_utilisateur.crt" -CA ../bdd_ca.crt -CAkey ../bdd_ca.key -CAcreateserial -CAserial ca.srl -extfile sslClientExtConfig.txt
    >openssl pkcs12 -export -out "$nom_utilisateur.pfx" -inkey "$nom_utilisateur.key" -in "$nom_utilisateur.crt" -passout pass:$mot_de_passe_certificat_client
    >openssl pkcs8 -topk8 -inform PEM -outform DER -in "$nom_utilisateur.key" -out "$nom_utilisateur.pk8" -nocrypt
    >popd
    >```

## Configuration du serveur de base de données

1. Activation de SSL

    Dans le fichier de configuration _postgresql.conf_, configurer les propriétés suivantes :

    |Propriété | Valeur|
    |------------ | -------------|
    |ssl | on|
    |ssl_cert_file | '/etc/postgresql/13/main/serveur.crt'|
    |ssl_key_file | '/etc/postgresql/13/main/serveur.key'|
    |ssl_ca_file | '/etc/postgresql/13/main/bdd_ca.crt'|
    |listen_addresses | '*'|

    #### Linux
    >```bash
    >pushd $repertoire_certificats_serveur
    >sed -i -E 's/(#){0,1}(ssl_cert_file = '\'')(.*)('\'')/\2\/etc\/postgresql\/13\/main\/serveur.crt\4/g' postgresql.conf
    >sed -i -E 's/(#){0,1}(ssl_key_file = '\'')(.*)('\'')/\2\/etc\/postgresql\/13\/main\/serveur.key\4/g' postgresql.conf
    >sed -i -E 's/(#){0,1}(ssl_ca_file = '\'')(.*)('\'')/\2\/etc\/postgresql\/13\/main\/bdd_ca.crt\4/g' postgresql.conf
    >```


2. Configuration des méthodes d'authentication

    Dans le fichier de configuration _pg_hba.conf_, ajouter la méthode d'authentification suivante : 

    >```bash
    >hostssl all all ::1/128 cert
    >```

    ##### Linux
    >```bash
    >sed -i '/# IPv6 local connections:/ a hostssl\tall\t\tall\t\t::1/128\t\t\tcert' pg_hba.conf
    >sed -i '/#/! s/\(^.*md5.*$\)/#\ \1/' pg_hba.conf
    >popd
    >```

3. Création d'un utilisateur et de la base de données

    ##### Linux
    >```bash
    >su postgres
    >psql
    >```

    >```sql
    >psql -c "CREATE ROLE $nom_utilisateur LOGIN NOSUPERUSER NOINHERIT NOCREATEDB NOCREATEROLE NOREPLICATION;"
    >```

4. Redémarrer le service Postgresql
    
    ##### Linux
    >```bash
    >systemctl restart postgresql
    >```

5. Tester l'authentification en utilisant PSQL

    |Propriété | Valeur|
    |------------ | ------------|
    |PGSSLCERT | $repertoire_certificat_client/$nom_utilisateur.crt|
    |PGSSLKEY | $repertoire_certificat_client/$nom_utilisateur.key|
    |PGSSLROOTCERT | $repertoire_certificat_client/$nom_utilisateur/bdd_ca.crt|
    |PGSSLMODE | require|
    |PGUSER | $nom_utilisateur|
    |PGHOST | $nom_hote|
    |PGHOSTADDR | ::1|
    |PGDATABASE | $nom_base_de_donnees|

    Vérifier que le mode d'authentification SSL est activé

    ##### Exemple
    >```bash
    >Connexion SSL (chiffrement : DHE-RSA-AES256-SHA, bits : 256)
    >```

    ##### Linux
    >```bash
    >export PGSSLCERT="$repertoire_certificat_client/$nom_utilisateur.crt"
    >export PGSSLKEY="$repertoire_certificat_client/$nom_utilisateur.key"
    >export PGSSLROOTCERT="$repertoire_certificats/bdd_ca.crt"
    >export PGSSLMODE=require
    >export PGUSER=$nom_utilisateur
    >export PGHOST=$nom_hote
    >export PGHOSTADDR=::1
    >export PGDATABASE=$nom_base_de_donnees
    >psql
    >```

    Vérifier que le sujet du certificat client est bien reconnu

    >```sql
    >SELECT usename,application_name,client_addr,client_hostname,client_port,pss.* 
    >FROM pg_stat_activity psa INNER JOIN pg_stat_ssl pss ON psa.pid=pss.pid 
    >WHERE psa.pid = (SELECT * FROM pg_backend_pid());
    >```

    ## Sources

    * [pg_hba.conf](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
    * [postgresql.conf](https://www.postgresql.org/docs/current/runtime-config-connection.html#RUNTIME-CONFIG-CONNECTION-SSL)
    * [x509v3_config](https://www.openssl.org/docs/man1.0.2/man5/x509v3_config.html)