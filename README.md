# pg_tde recovery-path key matrix

By Haifeng Cheng. This is a synthetic research draft for editorial consideration, not an accepted Percona article or a production incident report.

## Read and reproduce

- [Current reviewed article](article.md): the current reading edition, with local download links.
- [Exact reviewed manuscript](source/reviewed-manuscript.md): unchanged author-reviewed source, including Percona Hugo front matter and site-relative links.
- [Original experiment companion](downloads/percona-pg-tde-recovery-path-lab-20261113.zip): frozen 13 November experiment, scripts and sanitized recorded evidence. The article.md inside this archive is a historical draft, not the current reviewed article.
- [Clean-install supplement](downloads/percona-pg-tde-clean-install-20261115.zip): separately dated Debian 13 / OpenSSL 3.5.7 replay evidence and instructions.
- [Download checksums](downloads/SHA256SUMS).

The reading edition changes only presentation: a title/byline/cover precedes the manuscript body, Hugo front matter is omitted, and three site-relative download destinations point to local files. The exact reviewed source is preserved separately.

## Evidence and limits

The original three rounds contain 27 core cases: 15 complete recoveries and 12 expected missing-key failures. Two later fresh-container replays on one provisioned dependency image reproduced the nine-case matrix and controls. These are synthetic, automated observations, not production validation or independent platform certification. The scope is one database and one principal-key rotation, with fixed WAL and repository keys. External KMS, HA/failover, repeated rotations, concurrent-backup rotation and cross-version restoration are outside scope.

AI tools assisted the research, lab implementation and writing. Execution and checks were automated; Haifeng reviewed the revised manuscript. No unaided-authorship or detector-passing claim is made.

The archives are frozen evidence packages. Their status wording records their preparation time; it does not assert editorial acceptance. Do not publish raw replay directories: they can contain generated test keys, credentials, database files and backup repositories. Follow the archived export/sanitation instructions. Use only disposable, authorized test environments and synthetic data.

Source-preservation and recorded-evidence checks passed for the reviewed manuscript. A fresh full Percona Hugo build and desktop/mobile review of this manuscript have not been completed. Earlier rendering results belong to an earlier manuscript and are not a pass for this version.

No software binaries, real production data or new license grant are included. References remain attributed to their original publishers.
