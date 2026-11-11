# Metasploit Framework: Plugins & Mixins

---

## Table of Contents

1. [What a Metasploit plugin is](#1-what-a-metasploit-plugin-is)
2. [Why plugins make life easier](#2-why-plugins-make-life-easier)
3. [Finding installed plugins](#3-finding-installed-plugins)
4. [Loading a plugin](#4-loading-a-plugin)
5. [When a plugin fails to load](#5-when-a-plugin-fails-to-load)
6. [Installing a new plugin from scratch](#6-installing-a-new-plugin-from-scratch)
7. [Popular plugins worth knowing](#7-popular-plugins-worth-knowing)
8. [Mixins: a quick conceptual note](#8-mixins-a-quick-conceptual-note)
9. [Cheat sheet](#9-cheat-sheet)

---

## 1. What a Metasploit plugin is

A **plugin** is third-party software that its creators have approved for integration directly into the Metasploit Framework. Plugins come in two general flavors:

| Type | Example |
|---|---|
| **Commercial products with a free "Community Edition"** | Nessus, Nexpose |
| **Individual, community-built projects** | Smaller tools built by independent developers |

**Analogy:** think of plugins like browser extensions. The browser (msfconsole) already works fine on its own, but extensions (plugins) bolt on extra, specialized functionality — a password manager, an ad blocker, a note-taking tool — without you needing to leave the browser to use them.

---

## 2. Why plugins make life easier

Before plugins, a pentester often had to **jump between separate pieces of software**: run a scanner somewhere else, export its results, re-import them into Metasploit, manually set options again, and repeat this cycle constantly.

**With plugins loaded into msfconsole:**

- Everything gets **automatically documented** straight into your connected database.
- **Hosts, services, and vulnerabilities** become visible at a glance, without manual import/export cycles.
- Plugins talk directly to Metasploit's **API**, meaning they can manipulate the framework itself — not just bolt extra output onto the side.

### What plugins are generally good for

| Benefit | What it means in practice |
|---|---|
| **Automating repetitive tasks** | No more manually repeating the same import/export/configure cycle |
| **Adding new commands** | Plugins can literally extend what you can type into msfconsole |
| **Extending the framework** | Deep integration, not just a surface-level add-on |

---

## 3. Finding installed plugins

Every fresh Metasploit install ships with a default set of plugins, stored at:

```
/usr/share/metasploit-framework/plugins
```

```bash
$ ls /usr/share/metasploit-framework/plugins

aggregator.rb      beholder.rb        event_tester.rb  komand.rb     msfd.rb    nexpose.rb   request.rb  session_notifier.rb  sounds.rb  token_adduser.rb  wmap.rb
alias.rb           db_credcollect.rb  ffautoregen.rb   lab.rb        msgrpc.rb  openvas.rb   rssfeed.rb  session_tagger.rb    sqlmap.rb  token_hunter.rb
auto_add_route.rb  db_tracker.rb      ips_filter.rb    libnotify.rb  nessus.rb  pcap_log.rb  sample.rb   socket_logger.rb     thread.rb  wiki.rb
```

If the `.rb` file you want is sitting in this folder, it's ready to load.

---

## 4. Loading a plugin

Use the `load <plugin name>` command (no `.rb` extension needed) inside `msfconsole`:

```
msf6 > load nessus

[*] Nessus Bridge for Metasploit
[*] Type nessus_help for a command listing
[*] Successfully loaded Plugin: Nessus
```

A **successful load** gives you a short greeting message confirming the plugin is active, usually pointing you toward its own help command.

### Exploring what the plugin can do

Most plugins add their own `<pluginname>_help` command:

```
msf6 > nessus_help

Command                     Help Text
-------                     ---------
Generic Commands
-----------------           -----------------
nessus_connect              Connect to a Nessus server
nessus_logout               Logout from the Nessus server
nessus_login                Login into the connected Nessus server with a different username

<SNIP>

nessus_user_del             Delete a Nessus User
nessus_user_passwd          Change Nessus Users Password

Policy Commands
-----------------           -----------------
nessus_policy_list          List all policies
nessus_policy_del           Delete a policy
```

Notice how every new command is **prefixed with the plugin's name** (`nessus_...`). This keeps things organized and makes it obvious which plugin a given command belongs to, even once you have several plugins loaded at once.

> **Good habit:** read a plugin's specific help menu before using it. Each one has its own set of quirks and capabilities, and understanding them up front means you'll actually get the full value out of having them available.

---

## 5. When a plugin fails to load

If you try to load something that doesn't exist (or isn't correctly placed in the plugins folder), you'll get a clear error:

```
msf6 > load Plugin_That_Does_Not_Exist

[-] Failed to load plugin from /usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb: cannot load such file -- /usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb
```

**What this tells you:** `load` is literally looking for a file at `/usr/share/metasploit-framework/plugins/<name>.rb`. If that exact file isn't there, loading fails immediately — so the fix is almost always to double-check the plugin is actually present in that directory (see the next section if it isn't).

---

## 6. Installing a new plugin from scratch

Popular, actively maintained plugins typically get pulled into Parrot OS automatically with each distro update (via Parrot's own package repository). But for plugins that **aren't** bundled that way, you can install them manually.

### Step 1: get the plugin's `.rb` file

Most plugin authors publish their code (often on GitHub). For example, DarkOperator's collection of Metasploit plugins:

```bash
$ git clone https://github.com/darkoperator/Metasploit-Plugins
$ ls Metasploit-Plugins

aggregator.rb      ips_filter.rb  pcap_log.rb          sqlmap.rb
alias.rb           komand.rb      pentest.rb           thread.rb
auto_add_route.rb  lab.rb         request.rb           token_adduser.rb
beholder.rb        libnotify.rb   rssfeed.rb           token_hunter.rb
db_credcollect.rb  msfd.rb        sample.rb            twitt.rb
db_tracker.rb      msgrpc.rb      session_notifier.rb  wiki.rb
event_tester.rb    nessus.rb      session_tagger.rb    wmap.rb
ffautoregen.rb     nexpose.rb     socket_logger.rb
growl.rb           openvas.rb     sounds.rb
```

### Step 2: copy the plugin into Metasploit's plugin folder

Taking `pentest.rb` as the example:

```bash
$ sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
```

> **Permissions matter:** the file needs to actually be readable in that directory, which is why `sudo` is used here — a permissions mismatch is a common reason a copied-in plugin still fails to load.

### Step 3: load it and confirm

```bash
$ msfconsole -q

msf6 > load pentest

Version 1.6
Pentest Plugin loaded.
by Carlos Perez (carlos_perez[at]darkoperator.com)
[*] Successfully loaded plugin: pentest
```

### Step 4: see what new commands appeared

Once loaded, the plugin's commands become part of the **main `help` menu** (not necessarily a separate `_help` command — this varies by plugin):

```
msf6 > help

Tradecraft Commands
===================

    Command          Description
    -------          -----------
    check_footprint  Checks the possible footprint of a post module on a target system.


auto_exploit Commands
=====================

    Command           Description
    -------           -----------
    show_client_side  Show matched client side exploits from data imported from vuln scanners.
    vuln_exploit      Runs exploits based on data imported from vuln scanners.


Discovery Commands
==================

    Command                 Description
    -------                 -----------
    discover_db             Run discovery modules against current hosts in the database.
    network_discover        Performs a port-scan and enumeration of services found for non pivot networks.
    pivot_network_discover  Performs enumeration of networks available to a specified Meterpreter session.
    show_session_networks   Enumerate the networks one could pivot thru Meterpreter in the active sessions.

<SNIP>
```

**What this example plugin adds:** grouped commands for footprinting, auto-exploitation based on vulnerability scanner data, and network discovery through pivoting — all things that would otherwise require separate manual steps or separate tools entirely.

---

## 7. Popular plugins worth knowing

| Plugin | Pre-installed? | What it does |
|---|---|---|
| **nmap** | ✅ Yes | Port scanning integration |
| **Nexpose** | ✅ Yes | Vulnerability scanner integration (Rapid7's own product) |
| **Nessus** | ✅ Yes | Vulnerability scanner integration (Tenable's product) |
| **Mimikatz** (v.1) | ✅ Yes | Credential extraction (passwords, hashes, tickets) |
| **Stdapi** | ✅ Yes | Standard Meterpreter API extension |
| **Railgun** | ❌ No | Lets Meterpreter call native Windows API functions directly |
| **Priv** | ❌ No | Privilege-escalation-related functionality |
| **Incognito** | ✅ Yes | Token impersonation (acting as another logged-in user) |
| **DarkOperator's plugins** | ❌ No | Community-built collection (e.g. `pentest.rb` above) |

> These are worth exploring individually once you're comfortable with the basics — each one can meaningfully speed up a specific part of an assessment.

---

## 8. Mixins: a quick conceptual note

This section is intentionally brief — it's here to make you aware of the concept, not to make you an expert in Ruby.

### What is a mixin?

Metasploit is built in **Ruby**, an object-oriented language. A **mixin** is a way for a class to borrow functionality from another "class-like" grouping (called a **module** in Ruby) **without** that grouping being its parent class.

**Why this distinction matters:** normally in object-oriented programming, a class gets extra functionality through **inheritance** — becoming a "child" of a "parent" class. A mixin is different: it's more like **inclusion**. The class pulls in a ready-made set of extra behavior without being formally related to it in a parent/child sense.

**Analogy:** imagine a gym membership that gives you access to a pool, even though the gym didn't "give birth to" the pool, build it from scratch for you, or claim ownership over your fitness routine. You simply **included** that extra feature into your overall gym experience by opting into the membership. Inheritance would be more like being handed down your parent's entire set of habits and traits; a mixin is more like picking and choosing an extra capability to plug in.

### When mixins are used

| Use case | Why |
|---|---|
| **A class needs a lot of optional features** | Instead of cramming everything into one giant class, optional capabilities get mixed in only where needed |
| **One feature needs to be reused across many different classes** | Write the behavior once as a mixin, then `include` it wherever it's needed, instead of duplicating the code |

### The `include` keyword

In Ruby, mixins are implemented with the `include` keyword, followed by the module's name:

```ruby
module Flyable
  def fly
    puts "Flying!"
  end
end

class Bird
  include Flyable
end

Bird.new.fly   # => "Flying!"
```

`Bird` doesn't inherit from `Flyable` — it **includes** it, gaining the `fly` method as if it had always had it.

> **Should a beginner worry about this?** **`No`**. If you're starting out, focus on using modules, payloads, and plugins — mixins become relevant later, mainly if you start writing or heavily customizing your own Metasploit modules.

---

## 9. Cheat sheet

### Plugin commands

| Command | Purpose |
|---|---|
| `load <plugin>` | Load a plugin (no `.rb` extension) |
| `<pluginname>_help` | Many plugins add their own dedicated help command |
| `help` | May also show newly added plugin commands, grouped by category |

### Default plugin location

```
/usr/share/metasploit-framework/plugins
```

### Installing a new plugin (manual method)

```bash
# 1. Get the .rb file (e.g. via git clone from the author's repo)
$ git clone <plugin-repo-url>

# 2. Copy it into the plugins folder
$ sudo cp ./<repo>/<plugin>.rb /usr/share/metasploit-framework/plugins/<plugin>.rb

# 3. Load it inside msfconsole
msf6 > load <plugin>
```

### Mixins, in one line

> A **mixin** lets a Ruby class gain extra functionality from a module via `include`, without becoming a "child" of that module — used to share optional or reusable features across many classes.

### Big-picture takeaways

1. **Plugins are approved, integrated third-party tools** — not Metasploit's own built-in modules, but trusted add-ons with deep API access.
2. **Plugins remove the old "jump between tools" workflow** by writing results straight into your connected database automatically.
3. **`load <name>`** looks for `<name>.rb` in the default plugins folder — missing file = load failure with a clear error.
4. **Installing a new plugin is just: get the `.rb` file → copy it into the plugins folder with proper permissions → `load` it.**
5. **Many pre-installed plugins already cover common tools**: Nmap, Nessus, Nexpose, Mimikatz, Incognito, Stdapi.
6. **Mixins are a Ruby-level concept** (inclusion, not inheritance) that underpin how flexible and modular the framework's internals are — good to know exists, not something a beginner needs to master.

---
