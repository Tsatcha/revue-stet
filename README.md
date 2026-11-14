


 FS=$(findmnt -rn -o TARGET -t ext2,ext3,ext4,xfs,btrfs); echo "$FS"


=== 1. Vérifier l'état de départ

id elasticsearch; getent group sssd; getent group 994; getent passwd 994; getent group 499

-> continuer seulement si :
   - elasticsearch en 990:990
   - getent group 994 renvoie sssd
   - getent passwd 994 et getent group 499 ne renvoient RIEN


=== 2.  arrêter Elasticsearch


systemctl stop elasticsearch


=== 3. Libérer le GID 994 (groupe sssd)

NEW=$(for i in $(seq 999 -1 201); do [ "$i" = 994 ] || [ "$i" = 499 ] || getent group "$i" >/dev/null || { echo "$i"; break; }; done); echo "nouveau GID sssd : $NEW"

systemctl stop sssd

groupmod -g "$NEW" sssd

for m in $FS; do find "$m" -xdev -gid 994 -exec chgrp -h "$NEW" {} +; done

systemctl start sssd; systemctl is-active sssd


=== 4. Passer elasticsearch en 994:499

groupmod -g 499 elasticsearch

usermod -u 994 elasticsearch

for m in $FS; do find "$m" -xdev -uid 990 -exec chown -h 994 {} +; find "$m" -xdev -gid 990 -exec chgrp -h 499 {} +; done

-> durée selon le volume de /storage/elasticsearch


=== 5. Vérifier avant de redémarrer

id elasticsearch; getent group sssd; ls -ln /storage/elasticsearch /etc/elasticsearch

find /storage/elasticsearch /var/log/elasticsearch ! -uid 994 2>/dev/null | head

find /storage/elasticsearch /var/log/elasticsearch ! -gid 499 2>/dev/null | head

id <ton_login_ipa>

-> attendu :
   - elasticsearch en uid=994 gid=499
   - sssd sur $NEW
   - fichiers en 994 499 (ceux de /etc/elasticsearch en 0 499)
   - les deux find ne renvoient rien
   - le compte FreeIPA est résolu


=== 6. Redémarrer et 
systemctl start elasticsearch








https://download.bell-sw.com/java/17.0.12+10/bellsoft-jre17.0.12+10-linux-amd64-full.tar.gz
index="personnes-$(date '+%G-w%V')"

while IFS= read -r ligne || [[ -n "$ligne" ]]; do
    ligne="${ligne%$'\r'}"
    [[ -z "$ligne" ]] && continue

    uid=$(printf '%s' "$ligne" | sha256sum | cut -d ' ' -f 1)

    IFS=';' read -r nom ville age <<< "$ligne"

    document=$(jq -cn \
        --arg nom "$nom" \
        --arg ville "$ville" \
        --arg age "$age" \
        '{nom: $nom, ville: $ville, age: $age}') || break

    curl -fsS -X PUT "$ES_URL/$index/_doc/$uid" \
        -H "Content-Type: application/json" \
        --data-binary "$document" || break

    printf '\n'
done < <(tail -n +2 donnees.csv)





PUT _cluster/settings
{
  "persistent": {
    "indices.recovery.max_bytes_per_sec": "150mb",
    "indices.recovery.max_concurrent_file_chunks": 4
  }
}

PUT _snapshot/backup_s3
{
  "type": "s3",
  "settings": {
    "bucket": "elk-backup-production",
    "base_path": "snapshots/prod",
    "max_snapshot_bytes_per_sec": "100mb",
    "max_restore_bytes_per_sec": "200mb"
  }
}

PUT _snapshot/backup_nfs
{
  "type": "fs",
  "settings": {
    "location": "/mnt/es-backup",
    "compress": true
  }
}

PUT _cluster/settings
{
  "persistent": {
    "indices.recovery.max_bytes_per_sec": "150mb",
    "indices.recovery.max_concurrent_file_chunks": 4
  }
}

PUT _cluster/settings
{
  "persistent": {
    "indices.recovery.use_snapshots": true,
    "indices.recovery.max_concurrent_snapshot_file_downloads": 5,
    "indices.recovery.max_concurrent_snapshot_file_downloads_per_node": 25
  }
}

MANDES - OPTIMISATION RESTAURATION ELASTICSEARCH 8.x DEPUIS NFS
Contexte : 50 To, 10 data nodes, reseau 10 Gb/s
IMPORTANT : remplacer backup_nfs, /mnt/es-backup, snapshot_YYYY_MM_DD et logs-* par vos valeurs.
Executer les requetes HTTP dans Kibana Dev Tools. Tester hors production en priorite.

1. ETAT INITIAL
GET /
GET _cluster/health
GET _cluster/settings?include_defaults=true&flat_settings=true
GET _cat/nodes?v
GET _cat/shards?v
GET _cat/allocation?v
GET _snapshot/backup_nfs
GET _snapshot/backup_nfs/_all
POST _snapshot/backup_nfs/_verify

2. REGLAGE DEBIT RECOVERY (TEST PROGRESSIF)
PUT _cluster/settings
{
  "transient": {
    "indices.recovery.max_bytes_per_sec": "100mb"
  }
}

Tester ensuite 150mb, puis 200mb UNIQUEMENT si NFS et disques ne saturent pas.
Conserver par defaut les parametres de concurrence sauf diagnostic justifie.

3. DIAGNOSTIC RESTAURATION
GET _cat/recovery?v&active_only=true
GET _recovery?active_only=true&detailed=true
GET _nodes/stats/fs,process,jvm,thread_pool
GET _cluster/health

4. DIAGNOSTIC NFS / LINUX (sur les clients NFS)
findmnt -T /mnt/es-backup
nfsstat -m
nfsiostat 5 12
iostat -xz 5 12
sar -n DEV 5 12

Sur le serveur NFS :
nfsstat -s
iostat -xz 5 12
sar -n DEV 5 12

5. ANALYSE REPOSITORY (charge de test : privilegier preproduction)
POST _snapshot/backup_nfs/_analyze?timeout=10m

6. RESTAURATION SANS REPLICAS (cluster de reprise isole)
POST _snapshot/backup_nfs/snapshot_YYYY_MM_DD/_restore
{
  "indices": "logs-*",
  "include_global_state": false,
  "index_settings": {
    "index.number_of_replicas": 0
  }
}

Attention : indices cibles absents ou compatibles ; adapter pour data streams et indices systeme.

7. REACTIVER LES REPLICAS APRES RESTAURATION
PUT logs-*/_settings
{
  "index.number_of_replicas": 1
}

8. RETOUR ARRIERE
PUT _cluster/settings
{
  "transient": {
    "indices.recovery.max_bytes_per_sec": null
  }
}

9. BENCHMARK
- Restaurer un jeu representatif de 500 Go a 1 To sur cluster isole.
- Comparer les limites 40, 80, 100, 150, 200 MiB/s par noeud.
- Mesurer temps total, debit NFS agrege, latence disques, erreurs, throttling.
- Repartir du meme etat initial a chaque essai.
- Ne pas modifier manuellement les fichiers du repository Elasticsearch.

Documentation : https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore
POST _snapshot/backup_nfs/_verify



