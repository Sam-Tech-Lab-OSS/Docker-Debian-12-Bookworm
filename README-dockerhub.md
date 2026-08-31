# Debian 12 (Bookworm) — base image

Minimal, hardened, multi-architecture **Debian 12 (Bookworm)** base image, built `FROM
scratch` from the official Debian OCI rootfs. A complete Debian userland, a non-root user, a
hardened baseline — then it gets out of your way.

**Full documentation, in English and French:**
<https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm>

*Version française plus bas.*

---

## Base image support

Debian 12's regular security support ended on **11 July 2026**. Coverage now comes from
Debian LTS, which runs until **30 June 2028** — community-maintained and free of charge,
though it does not cover every package in the archive.

Monthly rebuilds pick up the security updates Debian publishes for Bookworm. Published
vulnerability scans report findings with no fix available alongside those that have one, so
the reports reflect the full exposure of the image.

---

## Tags

| Tag | Contents |
|---|---|
| `latest` | Tracks the monthly rebuild — amd64 + arm64 |
| `YYYY.MM` (e.g. `2026.08`) | The build from that month — amd64 + arm64 |

Tags point at a multi-architecture manifest; Docker selects the right image for the host platform.

**Neither tag is immutable.** `YYYY.MM` names the month, not one specific build: any build during
that month republishes it. For a genuinely fixed image, pin by digest:

```bash
docker pull samtechlab/debian-12-bookworm@sha256:<digest>
```

Each published image is signed. The signature establishes that the image was built by this
repository's publish workflow and has not been replaced since — something the SBOM and the
provenance attestation, on their own, do not show. Verify it with
[Cosign](https://docs.sigstore.dev/cosign/system_config/installation/):

```bash
cosign verify \
  --certificate-identity-regexp '^https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/\.github/workflows/' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  samtechlab/debian-12-bookworm@sha256:<digest>
```

Both flags matter: without them Cosign only confirms that *a* signature exists, not that it is
this repository's. Anyone can sign any public image.

Also published on GHCR as `ghcr.io/sam-tech-lab-oss/debian-12-bookworm`.

---

## Quick start

```bash
# Shell as the unprivileged appuser
docker run -it --rm samtechlab/debian-12-bookworm:latest

# Check who you are
docker run --rm samtechlab/debian-12-bookworm:latest id
```

Build on top of it. The image ends with `USER appuser`, so switch to root to install, then switch
back:

```dockerfile
FROM samtechlab/debian-12-bookworm:latest

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends your-package && \
    rm -rf /var/lib/apt/lists/*
USER appuser
```

---

## No init system — your CMD is PID 1

This image ships no init system: your `CMD` runs directly as PID 1. That is fine for a single
well-behaved process, but PID 1 has special duties on Linux — it must reap orphaned children and
handle signals itself. A process that does neither leaves zombies behind, or ignores `SIGTERM` so
`docker stop` waits out its timeout and kills it.

If your process is not designed for that role, let Docker supply a minimal init:

```bash
docker run --init samtechlab/debian-12-bookworm:latest your-command
```

```yaml
services:
  app:
    image: samtechlab/debian-12-bookworm:latest
    init: true
```

To supervise **several** processes in one container, a plain `CMD` is not enough: use a real init
system, or split them into separate containers. The
[s6-overlay variant](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm-S6-Overlay)
of this same Bookworm base covers that case.

---

## Key features

- Built `FROM scratch` from the official Debian OCI rootfs — no third-party base layer
- Published as a single multi-arch manifest (`amd64`, `arm64`)
- **Runs as a non-root user (`appuser`) end to end** — there is no privileged process, PID 1 included
- **Hardening** — `root` locked, SUID/SGID stripped, world-writable bits removed, `umask 027`
- **Service managers neutralised** (`policy-rc.d`, `initctl`) so packages do not try to start daemons
- **Supply-chain integrity** — Alpine builder pinned by digest, CI actions pinned by commit SHA,
  SBOM, SLSA provenance and a Cosign signature attached to every image
- APT & dpkg optimisation — no recommended/suggested packages, no translations, clean cache
- Locale and timezone configured (`en_US.UTF-8`, `UTC`)
- Continuously verified — hadolint, 9 container integration tests on both architectures, weekly
  Trivy scans

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `HOME` | `/config` | Home directory of `appuser` |
| `TZ` | `UTC` | Timezone |
| `LANG` | `en_US.UTF-8` | Locale (also `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Terminal type |
| `DEBIAN_FRONTEND` | `noninteractive` | Suppresses interactive APT prompts |
| `PATH` | `/usr/local/sbin:/usr/local/bin:…` | Standard system path |

**`PUID` and `PGID` are build-time arguments, not runtime settings.** `appuser` is created at
`1000:1000` when the image is built, and nothing applies them at container start — this image has
no init system. Passing `-e PUID=1001` to `docker run` has **no effect**. To use different IDs,
rebuild with `--build-arg PUID=1001 --build-arg PGID=1001`, or align permissions host-side.

---

## Filesystem and defaults

| Path | Purpose |
|---|---|
| `/config` | Home of `appuser`, mode `750`, owned by `appuser` — mount your persistent data here |

| Setting | Value |
|---|---|
| User | `appuser` (UID `1000`, GID `1000`) |
| Command | `CMD ["/bin/bash"]` |
| Entrypoint | none |

---

## Security model

The container has **no privileged process**: the image ends with `USER appuser`, so everything —
including PID 1 — runs unprivileged.

| Control | Implementation |
|---|---|
| Container user | `appuser` (UID `1000`), set with `USER` — no root process |
| `root` account | Password locked, `/root` mode `700` |
| Login shell for `appuser` | `/usr/sbin/nologin` |
| SUID/SGID binaries | Stripped image-wide at build time |
| World-writable files | Write bit removed image-wide at build time |
| Default umask | `027` |
| `/config` | Mode `750`, owned by `appuser` |
| Service managers | `policy-rc.d` and `initctl` neutralised |
| Supply chain | Alpine builder pinned by digest; CI actions pinned by commit SHA; SBOM, provenance and signature published |

Recommended runtime hardening:

```yaml
services:
  app:
    image: samtechlab/debian-12-bookworm:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
```

Vulnerability reporting:
[SECURITY.md](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/SECURITY.md)

---

## Troubleshooting

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
The image runs as `appuser`. Switch to `USER root` in your Dockerfile to install packages, then
back to `USER appuser`.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Unprivileged processes cannot bind ports below 1024. Use a port ≥ 1024 inside the container and
remap it on the host (`-p 80:8080`).

**`docker stop` takes ~10 seconds**
Your process is not handling `SIGTERM`, or is not reaping children as PID 1. Run it with
`--init`, or handle signals in the process itself.

**Files created in a mounted volume have the wrong owner**
`appuser` is fixed at `1000:1000`. Either `chown` the host directory to `1000:1000`, or rebuild
with `--build-arg PUID=… --build-arg PGID=…`.

**Zombie processes accumulate**
Your PID 1 is not reaping orphans. Use `--init`.

More entries in the
[full documentation](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm#troubleshooting).

---

## Support this work

These images are rebuilt every month, signed, scanned and documented. The work is done
in the open and given away — sponsoring is what keeps the schedule.

[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub-ea4aaa.svg?style=for-the-badge&logo=githubsponsors&logoColor=white)](https://github.com/sponsors/Sam-Tech-Lab-OSS)

---

## License

Apache 2.0 — see
[LICENSE](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/LICENSE) and
[NOTICE](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/NOTICE).
Copyright (c) 2026 Sam Tech Lab.

---
---

# Debian 12 (Bookworm) — image de base

Image de base **Debian 12 (Bookworm)** minimale, durcie et multi-architecture, construite
`FROM scratch` à partir du rootfs OCI officiel de Debian. Un userland Debian complet, un
utilisateur non-root, un socle durci — puis elle vous laisse travailler.

**Documentation complète, en anglais et en français :**
<https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm>

---

## Support de l'image de base

Le support de sécurité régulier de Debian 12 s'est achevé le **11 juillet 2026**. La
couverture est désormais assurée par la LTS Debian, jusqu'au **30 juin 2028** — maintenue
par la communauté et gratuite, mais elle ne couvre pas tous les paquets de l'archive.

Les reconstructions mensuelles récupèrent les mises à jour de sécurité que Debian publie pour
Bookworm. Les analyses de vulnérabilités publiées remontent aussi bien les vulnérabilités sans
correctif disponible que celles qui en ont un : les rapports reflètent l'exposition complète
de l'image.

---

## Tags

| Tag | Contenu |
|---|---|
| `latest` | Suit la reconstruction mensuelle — amd64 + arm64 |
| `YYYY.MM` (par ex. `2026.08`) | Le build de ce mois-là — amd64 + arm64 |

Les tags pointent vers un manifeste multi-architecture : Docker sélectionne l'image correspondant
à la plateforme hôte.

**Aucun de ces tags n'est immuable.** `YYYY.MM` désigne le mois, pas un build en particulier :
tout build de ce mois-là le republie. Pour une image réellement figée, épinglez par digest :

```bash
docker pull samtechlab/debian-12-bookworm@sha256:<digest>
```

Chaque image publiée est signée. La signature établit que l'image a été construite par le workflow
de publication de ce dépôt et n'a pas été remplacée depuis — ce que le SBOM et l'attestation de
provenance, seuls, ne montrent pas. Vérifiez-la avec
[Cosign](https://docs.sigstore.dev/cosign/system_config/installation/) :

```bash
cosign verify \
  --certificate-identity-regexp '^https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/\.github/workflows/' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  samtechlab/debian-12-bookworm@sha256:<digest>
```

Les deux options comptent : sans elles, Cosign confirme seulement qu'*une* signature existe, pas
qu'elle est celle de ce dépôt. N'importe qui peut signer n'importe quelle image publique.

Également publiée sur GHCR : `ghcr.io/sam-tech-lab-oss/debian-12-bookworm`.

---

## Démarrage rapide

```bash
# Shell en tant qu'appuser, non privilégié
docker run -it --rm samtechlab/debian-12-bookworm:latest

# Vérifier l'utilisateur effectif
docker run --rm samtechlab/debian-12-bookworm:latest id
```

Construire par-dessus. L'image se termine par `USER appuser` : repassez en root pour installer,
puis redescendez :

```dockerfile
FROM samtechlab/debian-12-bookworm:latest

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends votre-paquet && \
    rm -rf /var/lib/apt/lists/*
USER appuser
```

---

## Pas de système d'init — votre CMD est PID 1

Cette image ne fournit aucun système d'init : votre `CMD` s'exécute directement en tant que
PID 1. C'est adapté à un processus unique et bien élevé, mais PID 1 a des devoirs particuliers
sous Linux — il doit récupérer les processus orphelins et gérer lui-même les signaux. Un processus
qui ne fait ni l'un ni l'autre laisse des zombies, ou ignore `SIGTERM` et oblige `docker stop` à
attendre son délai avant de le tuer.

Si votre processus n'est pas conçu pour ce rôle, laissez Docker fournir un init minimal :

```bash
docker run --init samtechlab/debian-12-bookworm:latest votre-commande
```

```yaml
services:
  app:
    image: samtechlab/debian-12-bookworm:latest
    init: true
```

Pour superviser **plusieurs** processus dans un même conteneur, un simple `CMD` ne suffit pas :
utilisez un vrai système d'init, ou séparez-les en conteneurs distincts. La
[variante s6-overlay](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm-S6-Overlay)
de la même base Bookworm couvre ce cas.

---

## Points forts

- Construite `FROM scratch` depuis le rootfs OCI officiel Debian — aucune couche de base tierce
- Publiée comme un manifeste multi-architecture unique (`amd64`, `arm64`)
- **Tourne en utilisateur non-root (`appuser`) de bout en bout** — aucun processus privilégié,
  PID 1 compris
- **Durcissement** — `root` verrouillé, bits SUID/SGID supprimés, bits world-writable retirés,
  `umask 027`
- **Gestionnaires de services neutralisés** (`policy-rc.d`, `initctl`) : les paquets n'essaient
  pas de démarrer de daemons
- **Intégrité de la chaîne d'approvisionnement** — builder Alpine figé par digest, actions CI
  figées par SHA de commit, SBOM, provenance SLSA et signature Cosign joints à chaque image
- Optimisation APT & dpkg — aucun paquet recommandé/suggéré, aucune traduction, cache propre
- Locale et fuseau horaire configurés (`en_US.UTF-8`, `UTC`)
- Vérifiée en continu — hadolint, 9 tests d'intégration sur les deux architectures, scans Trivy
  hebdomadaires

---

## Variables d'environnement

| Variable | Défaut | Description |
|---|---|---|
| `HOME` | `/config` | Répertoire personnel de `appuser` |
| `TZ` | `UTC` | Fuseau horaire |
| `LANG` | `en_US.UTF-8` | Locale (également `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Type de terminal |
| `DEBIAN_FRONTEND` | `noninteractive` | Supprime les invites APT interactives |
| `PATH` | `/usr/local/sbin:/usr/local/bin:…` | Chemin système standard |

**`PUID` et `PGID` sont des arguments de build, pas des réglages d'exécution.** `appuser` est créé
en `1000:1000` à la construction de l'image, et rien ne les applique au démarrage du conteneur —
cette image n'a pas de système d'init. Passer `-e PUID=1001` à `docker run` n'a **aucun effet**.
Pour d'autres identifiants, reconstruisez avec `--build-arg PUID=1001 --build-arg PGID=1001`, ou
alignez les permissions côté hôte.

---

## Arborescence et valeurs par défaut

| Chemin | Rôle |
|---|---|
| `/config` | Home de `appuser`, mode `750`, lui appartenant — montez vos données persistantes ici |

| Réglage | Valeur |
|---|---|
| Utilisateur | `appuser` (UID `1000`, GID `1000`) |
| Commande | `CMD ["/bin/bash"]` |
| Entrypoint | aucun |

---

## Modèle de sécurité

Le conteneur n'a **aucun processus privilégié** : l'image se termine par `USER appuser`, donc tout
— y compris PID 1 — s'exécute sans privilèges.

| Contrôle | Mise en œuvre |
|---|---|
| Utilisateur du conteneur | `appuser` (UID `1000`), défini par `USER` — aucun processus root |
| Compte `root` | Mot de passe verrouillé, `/root` en mode `700` |
| Shell de connexion d'`appuser` | `/usr/sbin/nologin` |
| Binaires SUID/SGID | Supprimés sur toute l'image au build |
| Fichiers world-writable | Bit d'écriture retiré sur toute l'image au build |
| Umask par défaut | `027` |
| `/config` | Mode `750`, appartenant à `appuser` |
| Gestionnaires de services | `policy-rc.d` et `initctl` neutralisés |
| Chaîne d'approvisionnement | Builder Alpine figé par digest ; actions CI figées par SHA ; SBOM, provenance et signature publiés |

Durcissement recommandé à l'exécution :

```yaml
services:
  app:
    image: samtechlab/debian-12-bookworm:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
```

Signalement de vulnérabilité :
[SECURITY.md](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/SECURITY.md)

---

## Dépannage

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
L'image tourne en `appuser`. Passez en `USER root` dans votre Dockerfile pour installer des
paquets, puis revenez à `USER appuser`.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Un processus non privilégié ne peut pas écouter sous le port 1024. Utilisez un port ≥ 1024 dans
le conteneur et remappez-le côté hôte (`-p 80:8080`).

**`docker stop` traîne une dizaine de secondes**
Votre processus ne gère pas `SIGTERM`, ou ne récupère pas ses enfants en tant que PID 1. Lancez-le
avec `--init`, ou gérez les signaux dans le processus lui-même.

**Les fichiers créés dans un volume monté ont le mauvais propriétaire**
`appuser` est figé à `1000:1000`. Soit vous faites un `chown` du répertoire hôte vers `1000:1000`,
soit vous reconstruisez avec `--build-arg PUID=… --build-arg PGID=…`.

**Des processus zombies s'accumulent**
Votre PID 1 ne récupère pas les orphelins. Utilisez `--init`.

D'autres entrées dans la
[documentation complète](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm#dépannage).

---

## Soutenir ce travail

Ces images sont reconstruites chaque mois, signées, analysées et documentées. Ce travail
est mené au grand jour et mis à disposition — le parrainage est ce qui en maintient le
rythme.

[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub-ea4aaa.svg?style=for-the-badge&logo=githubsponsors&logoColor=white)](https://github.com/sponsors/Sam-Tech-Lab-OSS)

---

## Licence

Apache 2.0 — voir
[LICENSE](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/LICENSE) et
[NOTICE](https://github.com/Sam-Tech-Lab-OSS/Docker-Debian-12-Bookworm/blob/main/NOTICE).
Copyright (c) 2026 Sam Tech Lab.
