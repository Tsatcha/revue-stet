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



