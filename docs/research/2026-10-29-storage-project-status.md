# Storage project status verified on 2026-10-29

Release/maintenance check, not a local compatibility test. Recheck before
installation or upgrades; training memory or generic `latest` is insufficient.

| Project | Observed evidence | Consequence |
|---|---|---|
| PostgreSQL | Official policy lists 18.6 as current stable, supported to November 2030. The home page identifies 19 RC1 as testing. | Start evaluation on stable PG18, refreshing minor releases at build time. |
| Apache AGE | Repository not archived; last push October 13. PG18/v1.8.0-rc0 published July 9, not flagged prerelease. Apache distributes the PG18 1.8.0 source archive with checksum/signature files. | Released PG18 artifacts exist; not merely development-branch support. |
| AGE documentation discrepancy | Website still calls PG18 1.7.0 current. PG15/v1.8.0-rc0 from September 19 is a prerelease pending approval. | Pin the PG18 distribution, not generic latest. Neither version number nor rc0 suffix alone establishes release status. |
| pgvector | Repository not archived; last push October 28. Latest observed tag v0.8.7, dated October 1 in changelog; tagged CI matrix includes PG18. GitHub Releases is empty. | Active, tagged PG18 support. Empty Releases is not abandonment. |
| LadybugDB | Repository not archived; v0.22.1 published October 28. | Active comparator; no local benchmark claimed. |
| Kuzu | Archived/read-only since October 10, 2025. | Exclude as a new dependency; assess Ladybug independently. |

Sources inspected:

- [PostgreSQL policy](https://www.postgresql.org/support/versioning/)
  and [home page](https://www.postgresql.org/).
- [AGE PG18 release](https://github.com/apache/age/releases/tag/PG18%2Fv1.8.0-rc0),
  [Apache distribution](https://downloads.apache.org/age/PG18/1.8.0/),
  [website](https://age.apache.org/download/),
  [PG15 candidate](https://github.com/apache/age/releases/tag/PG15%2Fv1.8.0-rc0),
  [repository metadata](https://api.github.com/repos/apache/age).
- [pgvector tagged changelog](https://raw.githubusercontent.com/pgvector/pgvector/v0.8.7/CHANGELOG.md),
  [tagged CI](https://raw.githubusercontent.com/pgvector/pgvector/v0.8.7/.github/workflows/build.yml),
  [tags](https://api.github.com/repos/pgvector/pgvector/tags?per_page=3),
  [metadata](https://api.github.com/repos/pgvector/pgvector).
- [Ladybug release](https://github.com/LadybugDB/ladybug/releases/tag/v0.22.1),
  [metadata](https://api.github.com/repos/LadybugDB/ladybug),
  [Kuzu archive](https://github.com/kuzudb/kuzu).

Next gate: pin PostgreSQL/AGE/pgvector/Postgrex/OS together; verify archive
checksums and signatures; build both extensions; initialize pooled AGE
connections; verify type encoding and mixed SQL/Cypher/vector commit and
rollback, concurrency, cancellation, backup/restore and upgrades. CI target
declarations do not establish that we ran these tests. Recheck licenses,
advisories and relevant defects at that point.
