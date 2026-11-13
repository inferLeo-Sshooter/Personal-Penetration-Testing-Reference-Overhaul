# Finding, Installing, and Porting Metasploit Modules

## 1. Keeping Metasploit Up to Date

The easiest way to get new exploits is to just update the whole framework. Running an update pulls down every module that has been merged into the official Metasploit GitHub repository, so your copy of `msfconsole` stays current with the latest exploits, auxiliary modules, and features.

But sometimes you don't want a full update — you just need *one specific* module right now. In that case, you can hunt it down manually and drop it into your local installation yourself.

## 2. Hunting for Modules on ExploitDB

[ExploitDB](https://www.exploit-db.com/) is the go-to resource for finding exploit code that hasn't made it into Metasploit's official release yet. It lets you filter results by tag, and one particularly useful tag is **"Metasploit Framework (MSF)"** — checking this shows you only the exploits that already come packaged in Metasploit-module format, ready to drop in.

**Example scenario:** Suppose you're targeting a Nagios3 installation and want to use a known command injection bug in `statuswml.cgi`. You open `msfconsole` and search for it:

```
msf6 > search nagios
```

No results. That doesn't necessarily mean the exploit doesn't exist — it likely just means it hasn't been merged into the main Metasploit branch yet.

### Using `searchsploit` Instead of the Browser

Rather than searching the ExploitDB website, you can search from the terminal with `searchsploit`:

```bash
$ searchsploit nagios3
```

This might return several results, some of which are Python scripts and some of which are Ruby (`.rb`) files. The `.rb` files are the ones worth paying attention to, since Ruby is the language Metasploit modules are written in. You can narrow your search to just those by excluding other file types:

```bash
$ searchsploit -t Nagios3 --exclude=".py"
```

> ⚠️ **Heads up:** Just because a script ends in `.rb` doesn't guarantee it's a proper Metasploit module. Some `.rb` exploit scripts are plain Ruby with no Metasploit-compatible structure at all.

## 3. Installing a Downloaded Module

Once you've picked out the `.rb` file you want, you need to place it in the right spot so `msfconsole` can see it.

**Key locations:**
- The main installation folder: `/usr/share/metasploit-framework/`
- Your personal/local modules folder (symlinked into your home directory): `~/.msf4/`

Metasploit organizes its modules by category inside subfolders (exploits, auxiliary, etc.), so you may need to create matching subfolders inside `~/.msf4/` if they don't already exist, mirroring the structure found in the main install directory.

**Naming rules matter.** When you copy your file in, rename it to follow Metasploit's conventions:
- Use `snake_case`
- Only letters, numbers, and underscores — no dashes

For example:
```
nagios3_command_injection.rb
our_module_here.rb
```

### Copying the File In

```bash
$ cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
```

You can also tell `msfconsole` to load an additional modules directory at launch time:

```bash
$ msfconsole -m /usr/share/metasploit-framework/modules/
```

Or point to it from within an already-running console:

```
msf6 > loadpath /usr/share/metasploit-framework/modules/
```

### Refreshing the Module List

If `msfconsole` is already open, you don't need to restart it. Just run:

```
msf6 > reload_all
```

Then search for or load your new module directly:

```
msf6 > use exploit/unix/webapp/nagios3_command_injection
msf6 exploit(unix/webapp/nagios3_command_injection) > show options
```

Running `show options` will display what the module needs from you — things like `RHOSTS` (target), `RPORT` (target port), credentials, and the vulnerable URI path. Once those are filled in, the module is ready to run against your target.

## 4. Porting a Script Into a Proper Metasploit Module

Sometimes the exploit you want only exists as a raw Python or PHP script — not an actual Metasploit module. Turning it into one means rewriting it in Ruby, following Metasploit's module structure.

**The good news:** you almost never have to write this from scratch. The usual approach is:

1. Find an **existing** Metasploit module in the same category as the exploit you're porting (same vulnerability type, same target software, same protocol, etc.)
2. Copy that existing module as a template/boilerplate
3. Swap in the logic and specifics from the script you're porting over

**Example:** Say you want to port over an exploit called *Bludit 3.9.2 – Authentication Bruteforce Mitigation Bypass* (`48746.rb`) into Metasploit.

First, check what's already there:

```bash
$ ls /usr/share/metasploit-framework/modules/exploits/linux/http/ | grep bludit

bludit_upload_images_exec.rb
```

There's already a Bludit-related module in that folder — perfect, that becomes your boilerplate. Copy your new script into the same directory with an appropriately descriptive snake_case name:

```bash
$ cp ~/Downloads/48746.rb /usr/share/metasploit-framework/modules/exploits/linux/http/bludit_auth_bruteforce_mitigation_bypass.rb
```

From there, open the existing boilerplate module (`bludit_upload_images_exec.rb`) and look at the `include` statements near the top of the file. These are **mixins** — reusable chunks of Metasploit functionality (HTTP handling, authentication helpers, etc.) that your module inherits. You'll need to swap these out for whichever mixins match what your new module actually needs.

> 💡 **Tip:** If you're unsure which mixins, classes, or methods apply to your situation, the official Metasploit module-development documentation is the best reference for figuring out what's available.

**Style note:** Ruby modules for Metasploit are conventionally written using **hard tabs**, not spaces — keep this in mind when editing or writing new module code.

---

## Quick Reference Summary

| Goal | Command / Action |
|---|---|
| Update all modules | Update `msfconsole` via terminal |
| Search ExploitDB from CLI | `searchsploit <keyword>` |
| Filter to Metasploit-ready Ruby files | `searchsploit -t <keyword> --exclude=".py"` |
| Install folder (system-wide) | `/usr/share/metasploit-framework/` |
| Install folder (local/home) | `~/.msf4/` |
| Load a new modules path at launch | `msfconsole -m <path>` |
| Load a new modules path mid-session | `loadpath <path>` |
| Refresh modules without restarting | `reload_all` |
| Use a module | `use <module/path>` |
| See module's required settings | `show options` |
