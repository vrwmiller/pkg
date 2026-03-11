# pkg Supporting Workflows

This document describes the major operational workflows in `pkg`, tracing control flow from the CLI through `libpkg` to completion.

---

## `pkg update` — Refresh Repository Metadata

Fetches the latest package index from all configured repositories and updates the local cache.

```mermaid
sequenceDiagram
    participant User
    participant CLI as src/update.c
    participant Config as pkg_config.c
    participant RepoDB as pkg_repo.c / pkg_repo_update.c
    participant Fetch as fetch.c (curl/ssh/file)
    participant Meta as pkg_repo_meta.c
    participant DB as pkgdb.c / local.sqlite

    User->>CLI: pkg update [-f] [repo]
    CLI->>Config: load configured repos
    Config-->>CLI: list of repo definitions

    loop for each repository
        CLI->>RepoDB: pkg_repo_update(repo)
        RepoDB->>Fetch: fetch meta.conf (repo fingerprints)
        Fetch-->>RepoDB: meta.conf
        RepoDB->>Meta: parse + validate repo metadata
        Meta-->>RepoDB: signature type, compression, format version

        RepoDB->>Fetch: fetch packagesite.pkg (package index)
        Note over Fetch: HTTP/HTTPS via libcurl<br/>or SSH tunnel<br/>or local file://
        Fetch-->>RepoDB: compressed archive

        RepoDB->>Meta: verify signature / fingerprint
        Meta-->>RepoDB: OK / FAIL

        RepoDB->>RepoDB: extract packagesite.ucl / packagesite.yaml
        RepoDB->>DB: open/create repo cache database
        RepoDB->>DB: begin transaction
        RepoDB->>DB: truncate stale repo_packages rows
        loop for each package entry
            RepoDB->>DB: INSERT pkg metadata (name, version, origin, deps …)
        end
        RepoDB->>DB: COMMIT
        DB-->>CLI: repo N packages updated
    end

    CLI-->>User: All repositories are up to date.
```

---

## `pkg install` — Install Packages with Dependencies

Resolves the full dependency graph, then installs each package in topological order.

```mermaid
flowchart TD
    A([pkg install pkgA pkgB …]) --> B[src/install.c<br/>parse args, open pkgdb]
    B --> C[pkg_jobs_new PKG_JOBS_INSTALL]
    C --> D[pkg_jobs_add: enqueue requested packages]
    D --> E[pkg_jobs_solve]

    subgraph Solve["Dependency Resolution"]
        E --> F[pkg_jobs_universe_new<br/>build full package universe]
        F --> G[For each requested pkg:<br/>fetch candidate versions from repo cache]
        G --> H[Recurse on each dep:<br/>fetch dep candidates too]
        H --> I[pkg_jobs_conflicts:<br/>register file/declare conflicts]
        I --> J[pkg_solve_sat:<br/>encode deps + conflicts as SAT clauses]
        J --> K[PicoSAT: find satisfying assignment]
        K --> L[pkg_jobs_schedule:<br/>topological sort of install order]
    end

    L --> M{User confirmation<br/>--yes / -y?}
    M -->|N - show plan| N([Show install plan to user])
    M -->|Y - proceed| O[pkg_jobs_apply]

    subgraph Install["Per-Package Installation — pkg_add.c"]
        O --> P[Fetch .pkg archive<br/>fetch.c → libcurl/ssh/file]
        P --> Q[Verify checksum SHA-256 / BLAKE2]
        Q --> R[Verify package signature<br/>pkgsign_ecc / pkgsign_ossl]
        R --> S[Extract archive<br/>packing.c → libarchive]
        S --> T[Run pre-install scripts<br/>Lua 5.4 sandboxed / sh]
        T --> U[Register files, dirs, config_files<br/>pkgdb.c — INSERT into local.sqlite]
        U --> V{Config file exists<br/>on disk?}
        V -->|Yes| W[3-way merge<br/>merge3.c]
        V -->|No| X[Write file directly]
        W --> Y[Run post-install scripts<br/>Lua / sh]
        X --> Y
        Y --> Z[Update package record<br/>INSERT packages row]
    end

    Z --> AA{More packages<br/>in schedule?}
    AA -->|Yes| P
    AA -->|No| AB[Run post-transaction triggers<br/>triggers.c]
    AB --> AC([Done])
```

---

## `pkg upgrade` — Upgrade Installed Packages

Compares installed versions against the current repo index and calculates a minimal upgrade plan.

```mermaid
flowchart TD
    A([pkg upgrade]) --> B[src/upgrade.c<br/>open pkgdb, open repo caches]
    B --> C[pkg_jobs_new PKG_JOBS_UPGRADE]
    C --> D[pkg_jobs_add ALL:<br/>enqueue every installed package]
    D --> E[pkg_jobs_solve]

    subgraph Solve["Upgrade Solver"]
        E --> F[Universe: for each installed pkg<br/>check repo for newer version]
        F --> G{Newer version<br/>available?}
        G -->|Yes| H[Mark as UPGRADE candidate]
        G -->|No| I[Skip — already up to date]
        H --> J["Recurse: resolve new deps<br/>may add new packages"]
        J --> K[Detect conflicts<br/>if ABI changed etc.]
        K --> L[SAT solve: prefer higher versions<br/>minimise changed packages]
        L --> M[Topological sort:<br/>old version remove order<br/>new version install order]
    end

    M --> N[Show plan to user]
    N -->|Confirm| O[pkg_jobs_apply]
    N -->|Cancel| NA([Abort])

    subgraph Apply["Apply Upgrade — pkg_add.c"]
        O --> P[Fetch new package archive]
        P --> Q[Verify checksum + signature]
        Q --> R[Run pre-deinstall scripts<br/>on old version]
        R --> S[backup_lib.c:<br/>save .so libraries if BACKUP_LIBRARIES set]
        S --> T[Remove old files<br/>not present in new version]
        T --> U[Extract new archive<br/>packing.c]
        U --> V[Merge config files<br/>merge3.c for changed pkg_config_file entries]
        V --> W[Run post-install scripts<br/>on new version]
        W --> X[Update pkgdb record<br/>name · version · digest · flatsize]
    end

    X --> Y{More packages?}
    Y -->|Yes| P
    Y -->|No| Z[Run post-transaction triggers]
    Z --> AA([Done])
```

---

## `pkg delete` — Remove Packages

Removes packages and their registered files, optionally cascading to reverse dependencies.

```mermaid
flowchart TD
    A([pkg delete pkgA]) --> B[src/delete.c]
    B --> C[pkg_jobs_new PKG_JOBS_DEINSTALL]
    C --> D[pkg_jobs_add: enqueue target packages]
    D --> E{--recursive / -R?}
    E -->|Yes| F[Also enqueue all reverse deps<br/>packages that depend on target]
    E -->|No| G[Check reverse dep count<br/>warn if non-zero]
    F & G --> H[pkg_jobs_solve<br/>topological sort — reverse install order]
    H --> I{User confirmation}
    I -->|Confirm| J[pkg_jobs_apply]

    subgraph Delete["Per-Package Deletion — pkg_delete.c"]
        J --> K[Run pre-deinstall scripts<br/>Lua / sh]
        K --> L[For each file in pkgdb:<br/>delete if not shared / not modified]
        L --> M[For each config_file:<br/>check if user-modified — warn / preserve]
        M --> N[For each dir:<br/>rmdir if empty and owned solely by pkg]
        N --> O[Run post-deinstall scripts<br/>Lua / sh]
        O --> P[DELETE from packages, files, dirs<br/>deps, provides, requires …]
    end

    P --> Q{More packages<br/>in plan?}
    Q -->|Yes| K
    Q -->|No| R[Run post-transaction triggers]
    R --> S([Done])
```

---

## `pkg create` — Build a Package Archive

Packages a staged directory or an installed package into a distributable `.pkg` file.

```mermaid
flowchart TD
    A([pkg create -m MANIFEST -r STAGEDIR]) --> B[src/create.c]
    B --> C[pkg_create_from_dir OR<br/>pkg_create_from_installed]

    subgraph Manifest["Manifest Processing — pkg_create.c"]
        C --> D[Read +MANIFEST / plist<br/>pkg_manifest.c]
        D --> E[Resolve file list from STAGEDIR<br/>pkg_abi.c: detect ABI string]
        E --> F[Scan ELF/Mach-O binaries<br/>pkg_elf.c / binfmt_macho.c<br/>for shlib deps and provided libs]
        F --> G[Build struct pkg:<br/>name · version · files · dirs<br/>deps · shlibs · scripts …]
    end

    subgraph Sign["Optional Signing — pkgsign.c"]
        G --> H{SIGNING_COMMAND<br/>or key set?}
        H -->|Yes ECC| I[pkgsign_ecc.c:<br/>generate ECC signature]
        H -->|Yes RSA| J[pkgsign_ossl.c:<br/>generate RSA signature]
        H -->|No| K[No signature]
    end

    subgraph Archive["Archive Creation — packing.c"]
        I & J & K --> L[Create tar archive<br/>libarchive backend]
        L --> M{Compression format<br/>COMPRESSION_FORMAT}
        M -->|txz| N[xz compression]
        M -->|tzst| O[zstd compression]
        M -->|tgz| P[gzip compression]
        M -->|tbz| Q[bzip2 compression]
        N & O & P & Q --> R[Write +MANIFEST inside archive]
        R --> S[Append all package files]
        S --> T[Compute per-file SHA-256<br/>pkg_checksum.c]
    end

    T --> U([Output: pkgname-version.pkg])
```

---

## `pkg repo` — Create a Repository

Indexes a directory of `.pkg` files into a repository suitable for `pkg update` consumption.

```mermaid
flowchart TD
    A([pkg repo /path/to/packages/]) --> B[src/repo.c]
    B --> C[pkg_repo_create<br/>pkg_repo_create.c]

    C --> D[Scan directory for *.pkg files]

    subgraph Index["Package Indexing"]
        D --> E[For each .pkg file:<br/>open archive, read +MANIFEST]
        E --> F[Extract: name · version · origin<br/>deps · shlibs · provides · requires<br/>checksum · flatsize · abi]
        F --> G[INSERT into packagesite.sqlite<br/>in-memory then written out]
    end

    G --> H[Write packagesite.ucl / packagesite.yaml<br/>human-readable index]
    H --> I[Compress index<br/>→ packagesite.pkg]

    subgraph Meta["Repository Metadata"]
        I --> J[Build meta.conf<br/>format version · compression<br/>pubkey or fingerprint refs]
        J --> K{Signing key<br/>provided?}
        K -->|Yes| L[Sign packagesite.pkg<br/>pkgsign.c]
        K -->|No| M[Unsigned repo]
        L & M --> N[Write meta.conf to repo root]
    end

    N --> O([Repository ready for<br/>pkg update consumption])
```

---

## `pkg audit` — Security Vulnerability Check

Checks installed packages against the FreeBSD VuXML vulnerability database.

```mermaid
sequenceDiagram
    participant User
    participant CLI as src/audit.c
    participant Audit as pkg_audit.c
    participant Fetch as fetch.c
    participant DB as pkgdb.c

    User->>CLI: pkg audit [-F]

    alt -F flag: fetch fresh vuln db
        CLI->>Fetch: download vulns.xml.bz2<br/>from https://vuxml.freebsd.org/
        Fetch-->>CLI: vulns.xml.bz2
        CLI->>CLI: decompress + save to<br/>/var/db/pkg/vuln.xml
    end

    CLI->>Audit: pkg_audit_load(vuln.xml)
    Audit->>Audit: parse XML (yxml)<br/>build vulnerability index by CPE + range

    CLI->>DB: pkgdb_query(MATCH_ALL)
    DB-->>CLI: iterator over installed packages

    loop for each installed package
        CLI->>Audit: pkg_audit_package(pkg)
        Audit->>Audit: match name + version against<br/>vuln ranges (pkg_version.c comparison)
        alt vulnerable
            Audit-->>CLI: CVE list + description
            CLI-->>User: pkgname-version is vulnerable:<br/>CVE-XXXX-XXXX: description
        end
    end

    CLI-->>User: X problem(s) in the installed packages found.
```

---

## `pkg fetch` — Download Without Installing

Downloads package archives to the local cache without modifying the system.

```mermaid
flowchart TD
    A([pkg fetch -r repo pkgA]) --> B[src/fetch.c]
    B --> C[pkg_jobs_new PKG_JOBS_FETCH]
    C --> D[pkg_jobs_add: enqueue packages]
    D --> E[pkg_jobs_solve:<br/>resolve deps if --dependencies flag]
    E --> F[pkg_jobs_apply]
    F --> G[For each package:<br/>pkg_repo_fetch_pkg<br/>pkg_repo.c]
    G --> H[Fetch .pkg from repo URL<br/>fetch.c → libcurl/ssh/file]
    H --> I[Verify checksum + signature]
    I --> J[Write to /var/cache/pkg/<br/>without extracting]
    J --> K([Cache populated — ready for<br/>offline pkg add or air-gapped install])
```

---

## Lua Script Execution Lifecycle

Lua scripts are the preferred, sandboxed alternative to shell scripts for package hooks.

```mermaid
stateDiagram-v2
    [*] --> Loaded : pkg_add / pkg_delete begins

    Loaded --> Sandboxed : pkg_sandbox.c (Capsicum, FreeBSD only)

    Sandboxed --> PRE_INSTALL : run pre-install.lua
    Sandboxed --> PRE_UPGRADE : run pre-upgrade.lua
    Sandboxed --> PRE_DEINSTALL : run pre-deinstall.lua

    PRE_INSTALL --> FileExtract : extract files to disk
    PRE_UPGRADE --> FileExtract
    PRE_DEINSTALL --> FileRemove : remove registered files

    FileExtract --> POST_INSTALL : run post-install.lua
    FileExtract --> POST_UPGRADE : run post-upgrade.lua
    FileRemove --> POST_DEINSTALL : run post-deinstall.lua

    POST_INSTALL --> Done
    POST_UPGRADE --> Done
    POST_DEINSTALL --> Done

    Done --> [*]

    note right of Sandboxed
        Lua env restrictions:
        - No exec() / popen()
        - No network access
        - File I/O respects ROOTDIR
        - pkg.* API for registration ops
    end note
```

---

## Post-Transaction Trigger Execution

Triggers run once per transaction (not per package), deduplicating common post-install tasks.

```mermaid
flowchart LR
    A[Transaction completes<br/>pkg_add / pkg_delete / pkg_upgrade] --> B[triggers.c:<br/>collect trigger events<br/>from all modified packages]
    B --> C[Deduplicate:<br/>same trigger may be registered<br/>by many packages]
    C --> D{Trigger type}
    D -->|path-based| E[Check if any installed/removed<br/>file matches trigger path glob]
    D -->|dir-based| F[Check if any installed/removed<br/>dir matches trigger pattern]
    E & F --> G{Match found?}
    G -->|No| H[Skip trigger]
    G -->|Yes| I[Execute trigger script<br/>/usr/local/share/pkg/triggers/*.ucl<br/>or Lua / shell]
    I --> J([ldconfig · gtk-update-icon-cache<br/>mandb · etc.])
```

---

## Plugin Hook Integration

```mermaid
sequenceDiagram
    participant libpkg
    participant Loader as plugins.c
    participant Plugin as /usr/local/lib/pkg/*.so

    Note over libpkg,Loader: At startup
    libpkg->>Loader: pkg_plugins_init()
    Loader->>Plugin: dlopen() each .so
    Loader->>Plugin: call plugin_init()
    Plugin->>Loader: pkg_plugin_hook_register(HOOK_TYPE, callback)
    Loader-->>libpkg: hooks registered

    Note over libpkg,Plugin: During operation
    libpkg->>Loader: pkg_plugins_hook_run(PRE_INSTALL, pkg)
    Loader->>Plugin: invoke registered callback(pkg)
    Plugin-->>Loader: return (modify pkg, log, abort)
    Loader-->>libpkg: continue or abort

    Note over libpkg,Plugin: Available hook points
    Note right of Plugin: PRE_INSTALL / POST_INSTALL<br/>PRE_DEINSTALL / POST_DEINSTALL<br/>PRE_UPGRADE / POST_UPGRADE<br/>PRE_FETCH / POST_FETCH<br/>EVENT (progress / error notifications)
```
