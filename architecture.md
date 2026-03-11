# pkg Software Architecture

FreeBSD's `pkg` is a binary package management system built as a layered architecture: a thin CLI frontend (`pkg` binary) over a comprehensive shared library (`libpkg`), supported by pluggable fetch backends, a SAT-based dependency solver, and an SQLite local database.

---

## High-Level Component Overview

```mermaid
graph TD
    CLI["CLI (pkg binary — src/)"]
    Tests["Test Suite (tests/)"]

    libpkg(["libpkg (shared/static library)"])

    DB["SQLite Database /var/db/pkg/"]
    Fetch["Fetch Backends (curl / SSH / file)"]
    Scripts["Script Runtimes (Lua 5.4 / sh)"]
    Solver["Dependency Solver (PicoSAT)"]
    Crypto["Crypto / Signing (libecc / OpenSSL)"]
    Archives["Archives (libarchive)"]

    CLI --> libpkg
    Tests --> libpkg

    libpkg --> DB
    libpkg --> Fetch
    libpkg --> Scripts
    libpkg --> Solver
    libpkg --> Crypto
    libpkg --> Archives
```

---

## Repository Layers

```mermaid
graph TB
    subgraph CLI["CLI Layer — src/"]
        main["main.c<br/>Argument dispatch"]
        cmds["38 command modules<br/>add · install · delete · upgrade<br/>update · repo · audit · create<br/>query · rquery · …"]
        main --> cmds
    end

    subgraph Core["Core Library — libpkg/"]
        jobs["Job Orchestrator<br/>pkg_jobs.c / pkg_jobs_universe.c"]
        solver["SAT Solver<br/>pkg_solve.c + PicoSAT"]
        sched["Scheduler<br/>pkg_jobs_schedule.c (toposort)"]
        conflicts["Conflict Detector<br/>pkg_jobs_conflicts.c"]
        pkgadd["pkg_add.c<br/>Installation logic"]
        pkgdel["pkg_delete.c<br/>Removal logic"]
        pkgcreate["pkg_create.c<br/>Package creation"]
        repomgr["Repository Manager<br/>pkg_repo.c / pkg_repo_update.c"]
        config["Configuration<br/>pkg_config.c (UCL)"]
        events["Event System<br/>pkg_event.c"]
        plugins["Plugin Loader<br/>plugins.c (dlopen)"]
        audit["Security Audit<br/>pkg_audit.c"]
        abi["ABI Detection<br/>pkg_abi.c / pkg_elf.c / pkg_abi_macho.c"]
    end

    subgraph DB["Database Layer"]
        pkgdb["pkgdb.c<br/>Schema, transactions"]
        pkgdbq["pkgdb_query.c<br/>SQL query builders"]
        pkgdbi["pkgdb_iterator.c<br/>Result iterators"]
        sqlite[("SQLite<br/>local.sqlite<br/>schema v0.39")]
        pkgdb --> pkgdbq --> pkgdbi
        pkgdb --> sqlite
    end

    subgraph Fetch["Fetch Layer"]
        fetchd["fetch.c<br/>Dispatcher"]
        curl["fetch_libcurl.c<br/>HTTP / HTTPS / FTP"]
        ssh["fetch_ssh.c<br/>pkg+ssh://"]
        file["fetch_file.c<br/>file://"]
        fetchd --> curl & ssh & file
    end

    subgraph Scripting["Scripting Layer"]
        lua["lua.c + lua_scripts.c<br/>Lua 5.4 (sandboxed)"]
        sh["scripts.c<br/>/bin/sh execution"]
        sandbox["pkg_sandbox.c<br/>Capsicum isolation"]
        lua --> sandbox
    end

    subgraph Packaging["Packaging & Verification"]
        packing["packing.c<br/>libarchive (tar/tgz/txz/tzst)"]
        checksum["pkg_checksum.c<br/>SHA-256 / BLAKE2"]
        sign["pkgsign.c<br/>pkgsign_ecc.c / pkgsign_ossl.c"]
        merge["merge3.c<br/>3-way config merge"]
        triggers["triggers.c<br/>Post-transaction hooks"]
        packing --- checksum
        sign --- checksum
    end

    cmds --> jobs
    cmds --> repomgr
    cmds --> config
    jobs --> solver --> sched
    jobs --> conflicts
    sched --> pkgadd & pkgdel
    pkgadd --> packing & lua & sh & triggers & merge
    pkgdel --> triggers
    pkgadd --> pkgdb
    pkgdel --> pkgdb
    repomgr --> fetchd
    repomgr --> pkgdb
    cmds --> audit
    cmds --> pkgcreate
    pkgcreate --> packing & sign
    cmds --> plugins

    classDef centered text-align:center
    class main,cmds,jobs,solver,sched,conflicts,pkgadd,pkgdel,pkgcreate,repomgr,config,events,plugins,audit,abi,pkgdb,pkgdbq,pkgdbi,sqlite,fetchd,curl,ssh,file,lua,sh,sandbox,packing,checksum,sign,merge,triggers centered
```

---

## Internal `libpkg` Module Map

```mermaid
graph LR
    subgraph Public_API["Public API (pkg.h.in)"]
        api["pkg_init · pkgdb_open<br/>pkg_jobs_new · pkg_event_register<br/>pkg_addfile · pkg_addoption …"]
    end

    subgraph Data_Structures["Core Data Structures (private/pkg.h)"]
        pkg_s["struct pkg<br/>name · version · origin · digest<br/>deps · files · dirs · scripts<br/>flags (locked / automatic / vital)"]
        dep_s["struct pkg_dep<br/>name · origin · version"]
        file_s["struct pkg_file<br/>path · sum · mode · uname"]
        cfgfile_s["struct pkg_config_file<br/>path · content (3-way merge)"]
        conflict_s["struct pkg_conflict<br/>uid · type"]
        jobs_s["struct pkg_jobs<br/>type · universe · solver state"]
    end

    subgraph Repo["Repository Subsystem"]
        repo_ops["pkg_repo_ops interface<br/>init · access · open · close<br/>query · search · fetch_pkg<br/>mirror_pkg · ensure_loaded"]
        binary_be["repo/binary/ backend<br/>packagesite.sqlite cache"]
        meta["pkg_repo_meta.c<br/>fingerprints · format version<br/>compression · expiry"]
        repo_ops --> binary_be
        binary_be --> meta
    end

    subgraph Version["Version & Compatibility"]
        ver["pkg_version.c<br/>FreeBSD port version comparison<br/>alphanumeric segment parsing"]
        cudf["pkg_cudf.c<br/>CUDF output for external solvers"]
        abi["pkg_abi.c<br/>ABI string: FreeBSD:14:amd64"]
    end

    api --> pkg_s & jobs_s
    pkg_s --> dep_s & file_s & cfgfile_s & conflict_s
    jobs_s --> repo_ops & ver
```

---

## Dependency Solver Architecture

```mermaid
graph TD
    request["User Request<br/>pkg install / upgrade / delete"]

    universe["Universe Build<br/>pkg_jobs_universe.c<br/>• gather installed + remote packages<br/>• resolve missing deps transitively<br/>• include all candidate versions"]

    conflicts["Conflict Registration<br/>pkg_jobs_conflicts.c<br/>• file conflicts between packages<br/>• explicit conflict declarations"]

    sat["SAT Encoding<br/>pkg_solve.c<br/>• packages → boolean variables<br/>• deps → SAT clauses (must-install)<br/>• conflicts → SAT clauses (cannot-both)<br/>• installed-state constraints"]

    picosat["PicoSAT<br/>(external/picosat/)<br/>CDCL-based solver<br/>finds satisfying assignment"]

    schedule["Topological Sort<br/>pkg_jobs_schedule.c<br/>• build dependency DAG from solution<br/>• Kahn's algorithm for order<br/>• ensures deps installed before dependents"]

    execute["Execution<br/>pkg_add / pkg_delete in order"]

    request --> universe --> conflicts --> sat --> picosat --> schedule --> execute
```

---

## Fetch & Network Stack

```mermaid
graph LR
    caller["Repository Update /<br/>Package Download"]

    dispatcher["fetch.c<br/>URL scheme dispatch<br/>+ retry / timeout logic"]

    subgraph backends["Fetch Backends"]
        http["fetch_libcurl.c<br/>HTTP · HTTPS · FTP<br/>resumable · proxy · TLS verify"]
        pkgssh["fetch_ssh.c<br/>pkg+ssh://<br/>SSH tunnel transport"]
        localfile["fetch_file.c<br/>file://<br/>Local filesystem"]
    end

    subgraph verification["Download Verification"]
        cksum["pkg_checksum.c<br/>SHA-256 / BLAKE2"]
        sig["pkgsign.c<br/>ECC or RSA/OpenSSL<br/>signature check"]
    end

    caller --> dispatcher
    dispatcher -->|"https://"| http
    dispatcher -->|"pkg+ssh://"| pkgssh
    dispatcher -->|"file://"| localfile
    http & pkgssh & localfile --> cksum --> sig
```

---

## Configuration & Plugin Systems

```mermaid
graph TB
    subgraph Config["Configuration System"]
        conf["/etc/pkg/pkg.conf<br/>/usr/local/etc/pkg/<br/>ENV overrides"]
        ucl["libucl parser<br/>(UCL format)"]
        ctx["struct pkg_ctx<br/>global runtime config"]
        repos["repo definitions<br/>URL · mirror · signature<br/>tls_ca_cert · priority"]
        conf --> ucl --> ctx
        ctx --> repos
    end

    subgraph Plugins["Plugin System"]
        loader["plugins.c<br/>dlopen() at startup<br/>/usr/local/lib/pkg/*.so"]
        hooks["Hook Points<br/>pre/post install · remove<br/>upgrade · validation · events"]
        pluginapi["Plugin API<br/>pkg_plugin_hook_register()<br/>pkg_plugin_info()"]
        loader --> hooks
        hooks --> pluginapi
    end

    subgraph Signing["Package Signing"]
        sigtype{"SIGNATURE_TYPE"}
        pubkey["pubkey<br/>RSA public key<br/>(pkgsign_ossl.c)"]
        fingerprints["fingerprints<br/>ECC keys via libder/libecc<br/>(pkgsign_ecc.c)"]
        nosig["none<br/>No verification"]
        sigtype -->|pubkey| pubkey
        sigtype -->|fingerprints| fingerprints
        sigtype -->|none| nosig
    end
```

---

## Database Schema (key tables)

```mermaid
erDiagram
    packages {
        int id PK
        text name
        text version
        text origin
        text comment
        text desc
        text maintainer
        text prefix
        text abi
        text digest
        int flatsize
        int automatic
        int locked
        int vital
        int time
    }
    deps {
        text name FK
        text origin
        text version
        int package_id FK
    }
    files {
        text path PK
        text sha256
        int package_id FK
    }
    config_files {
        text path PK
        text content
        int package_id FK
    }
    shlibs_required {
        int package_id FK
        int shlib_id FK
    }
    shlibs_provided {
        int package_id FK
        int shlib_id FK
    }
    shlibs {
        int id PK
        text name
    }
    provides {
        int package_id FK
        text provide
    }
    requires {
        int package_id FK
        text require
    }
    annotations {
        int package_id FK
        int tag_id FK
        int value_id FK
    }

    packages ||--o{ deps : "depends on"
    packages ||--o{ files : "owns"
    packages ||--o{ config_files : "manages"
    packages ||--o{ shlibs_required : "needs"
    packages ||--o{ shlibs_provided : "provides"
    packages ||--o{ provides : "provides capability"
    packages ||--o{ requires : "requires capability"
    packages ||--o{ annotations : "annotated with"
    shlibs ||--o{ shlibs_required : "required by"
    shlibs ||--o{ shlibs_provided : "provided by"
```

---

## Build System

```mermaid
graph LR
    autodef["auto.def<br/>Top-level feature checks<br/>(platform, openssl, libarchive…)"]
    autosetup["autosetup/<br/>Tcl-based build tool<br/>(replaces autoconf)"]
    makefiles["Makefile.autosetup fragments<br/>(libpkg, src, compat, docs…)"]

    autodef --> autosetup --> makefiles

    subgraph outputs["Build Outputs"]
        libpkg_so["libpkg.so.X.Y"]
        libpkg_a["libpkg_flat.a<br/>(all deps statically linked)"]
        pkg_bin["/usr/local/sbin/pkg"]
        pkg_static["/usr/local/sbin/pkg-static<br/>(fully static binary)"]
    end

    makefiles --> libpkg_so & libpkg_a --> pkg_bin & pkg_static

    subgraph bundled["Bundled External Libraries"]
        b2["blake2"]
        sq["sqlite"]
        ucl["libucl"]
        pico["picosat"]
        yx["yxml"]
        llua["liblua 5.4"]
        lc["libcurl (optional)"]
        lder["libder"]
        lecc["libecc"]
        lelf["libelf (optional)"]
        lnoise["linenoise"]
    end

    bundled --> libpkg_a
```

---

## Platform Support Matrix

```mermaid
graph LR
    core["libpkg core"]

    subgraph freebsd["FreeBSD (primary)"]
        caps["Capsicum sandbox<br/>for Lua scripts"]
        elf_f["ELF ABI detection<br/>(pkg_elf.c)"]
        kqueue["kqueue event loop"]
    end

    subgraph linux["Linux"]
        elf_l["ELF ABI detection<br/>(libelf)"]
        fts_l["fts_open (libfts compat)"]
    end

    subgraph macos["macOS"]
        macho["Mach-O parsing<br/>(binfmt_macho.c / pkg_abi_macho.c)"]
        cf["CoreFoundation<br/>SystemConfiguration"]
    end

    subgraph dragonfly["DragonFly BSD"]
        elf_d["ELF ABI detection"]
    end

    core --> freebsd & linux & macos & dragonfly
```
