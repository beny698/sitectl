# sitectl

A command-line tool for managing Apache virtual hosts on Ubuntu/Debian LAMP servers with automatic Cloudflare DNS and Let's Encrypt SSL configuration.

```
  ███████╗██╗████████╗███████╗ ██████╗████████╗██╗     
  ██╔════╝██║╚══██╔══╝██╔════╝██╔════╝╚══██╔══╝██║     
  ███████╗██║   ██║   █████╗  ██║        ██║   ██║     
  ╚════██║██║   ██║   ██╔══╝  ██║        ██║   ██║     
  ███████║██║   ██║   ███████╗╚██████╗   ██║   ███████╗
  ╚══════╝╚═╝   ╚═╝   ╚══════╝ ╚═════╝   ╚═╝   ╚══════╝
```

## Features

- **One-command site setup** - DNS, Apache vhost, and SSL in a single command
- **Git integration** - Automatically clone GitHub repositories and set up deployment workflow (enabled by default)
- **Cloudflare DNS integration** - Automatically creates/updates A and AAAA records
- **Let's Encrypt SSL** - Uses DNS-01 challenge via Cloudflare (works even before DNS propagates publicly)
- **IPv4 + IPv6 support** - Auto-detects and configures both
- **Smart domain handling**:
  - Subdomains: `api.example.com` -> detects `example.com` zone automatically
  - Root domains: `example.com` -> also configures `www.example.com` with redirect
- **Cloudflare proxy** - Enabled by default (orange cloud), optional `--no-proxy` flag
- **Resumable operations** - If setup fails midway, just run again to continue
- **Safe removal** - Cleans up DNS, SSL, and Apache but preserves your files
- **Preview mode** - Use `--dry-run` to see what changes would be made without executing them
- **Security hardened** - Safe config parsing, domain validation, and concurrency protection
- **Concurrent execution protection** - Lock files prevent multiple operations on the same domain
- **Consistent permissions** - All site files are owned by `SITE_OWNER:www-data` with fixed modes (dirs `2750`, files `0640`)

## Requirements

- Ubuntu/Debian server with Apache
- Cloudflare account managing your domain's DNS
- Cloudflare API token with `Zone:Read` and `DNS:Edit` permissions

## Quick Start

### 1. Install

```bash
# Download the script
curl -O https://raw.githubusercontent.com/beny698/sitectl/main/sitectl

# Run the installer (prompts for site owner, installs dependencies, creates config)
sudo bash sitectl install
```

During install you will be prompted for `SITE_OWNER` — the Linux user that will own all website files. This user must already exist on the system. If the user does not exist, the installer will exit with a clear error.

**Reinstalling** (upgrading an existing install) preserves the existing `/etc/sitectl/config` unchanged.

### 2. Configure

Edit `/etc/sitectl/config` to add your Cloudflare API token and email:

```bash
sudo nano /etc/sitectl/config
```

```bash
# Site owner: Linux user that owns all site files (set during install)
SITE_OWNER="yourusername"

# Cloudflare API Token (Zone:Read, DNS:Edit permissions)
CF_TOKEN="your_cloudflare_api_token_here"

# Email for Let's Encrypt registration
TLS_EMAIL="you@example.com"

# Optional: max seconds to poll for DNS propagation before timing out (default: 180)
DNS_WAIT_SECONDS=180

# Optional: Server IP addresses (auto-detected if not set)
# SERVER_IPV4="1.2.3.4"
# SERVER_IPV6="2001:db8::1"
```

**Note**: Server IPs are auto-detected by default. Only set `SERVER_IPV4` or `SERVER_IPV6` if auto-detection does not work in your environment.

### 3. Add Your First Site

```bash
sudo sitectl add yourdomain.com
```

That's it! Your site is now live at `https://yourdomain.com` with:

- Cloudflare DNS configured (A + AAAA records, proxy enabled)
- Apache virtual host running
- Valid SSL certificate from Let's Encrypt
- Automatic HTTP -> HTTPS redirect
- www -> non-www redirect (for root domains)
- All files owned by `SITE_OWNER:www-data`, directories `2750`, files `0640`

## Permissions

sitectl applies a fixed, consistent permission model to every site it manages:

| What           | Value                      |
|----------------|----------------------------|
| Owner          | `SITE_OWNER` (from config) |
| Group          | `www-data` (always fixed)  |
| Directory mode | `2750` (setgid, rwxr-x---) |
| File mode      | `0640` (rw-r-----)         |

Permissions are enforced automatically at the end of every `sitectl add` (both git and non-git flows) and `sitectl add-git`.

The setgid bit (`2` in `2750`) ensures new files created inside site directories automatically inherit the `www-data` group.

No per-site Linux users are created. All sites are owned by the single `SITE_OWNER` configured at install time.

## Commands

### `sitectl add <domain> [options]`

Create a new site with DNS, Apache, and SSL. **Git integration is enabled by default** - you'll be prompted for a GitHub repository URL.

```bash
# Add a subdomain with Git integration (default)
sudo sitectl add api.example.com
# Prompts for: GitHub repo URL
# Clones repo, sets SITE_OWNER:www-data ownership, dirs 2750, files 0640

# Add a root domain (also configures www)
sudo sitectl add example.com

# Skip Git integration (creates placeholder index.html)
sudo sitectl add api.example.com --without-git

# Disable Cloudflare proxy (direct connection)
sudo sitectl add api.example.com --no-proxy

# Skip DNS setup (if already configured elsewhere)
sudo sitectl add api.example.com --skip-dns

# Skip SSL (HTTP only)
sudo sitectl add api.example.com --skip-ssl

# Custom DNS propagation wait time
sudo sitectl add api.example.com --wait 300

# Preview changes without executing (dry run)
sudo sitectl add example.com --dry-run
```

### `sitectl add-git <domain>`

Add Git integration to an existing site. Prompts for a GitHub repository URL, clones the repository (or uses an existing `.git`), and enforces `SITE_OWNER:www-data` ownership and permissions.

```bash
sudo sitectl add-git oldsite.example.com
# Prompts for: GitHub repo URL (SSH format)
```

### `sitectl remove <domain>`

Remove a site completely (requires confirmation).

**Removes:**

- Apache configurations (disables and deletes)
- SSL certificates (deletes and stops renewal)
- Cloudflare DNS records

**Preserves:**

- Document root directory (`/var/www/<domain>/`)

```bash
sudo sitectl remove api.example.com
```

### `sitectl list`

Show all configured sites with their status.

```
  DOMAIN                              HTTP       HTTPS      SSL EXPIRY
  -----------------------------------------------------------------
  api.example.com                     enabled    enabled    2025-04-15
  example.com                         enabled    enabled    2025-04-10
```

### `sitectl status <domain>`

Show detailed status for a site including:

- Document root existence and file count
- Apache site status (enabled/disabled)
- SSL certificate details and expiry
- Cloudflare DNS records (with proxy status)
- Live connectivity tests (HTTP/HTTPS)

```bash
sudo sitectl status api.example.com
```

### `sitectl install`

First-time setup on a new server:

- **Prompts for `SITE_OWNER`** (required) - the Linux user that will own all site files. Must exist on the system; installer exits with a clear error if the user is not found.
- Installs dependencies (apache2, certbot, python3-certbot-dns-cloudflare, curl, jq)
- Enables required Apache modules (ssl, headers, rewrite, http2)
- Creates `/etc/sitectl/config` (includes `SITE_OWNER`)
- Sets up certbot renewal hooks
- Installs sitectl to `/usr/local/bin/sitectl`
- Installs the site-pull deployment script to `/usr/local/bin/site-pull`

**Reinstalling** preserves the existing `/etc/sitectl/config` without overwriting it.

```bash
sudo bash sitectl install
```

### `sitectl help`

Display full documentation.

## Git Integration

sitectl includes built-in Git integration for automatic repository deployment. When enabled (default), it clones your GitHub repository and integrates with the site-pull deployment script.

### Quick Start with Git

```bash
# Add a new site with Git integration (default behavior)
sudo sitectl add api.example.com

# You will be prompted for:
# - GitHub repository URL (SSH format): git@github.com:user/repo.git
```

This automatically:
1. Creates document root at `/var/www/api.example.com`
2. Clones your repository into the document root
3. Enforces ownership `SITE_OWNER:www-data`, directories `2750`, files `0640`

### Deploy Updates

After pushing changes to your repository, pull them to the server:

```bash
# Deploy latest commit on the default branch (main)
sudo /usr/local/bin/site-pull api.example.com

# Deploy a specific branch
sudo /usr/local/bin/site-pull api.example.com staging
```

site-pull reads `SITE_OWNER` from `/etc/sitectl/config` automatically, so no changes to the script are needed when adding new sites. After each pull it re-applies the same permissions as `sitectl add`: ownership `SITE_OWNER:www-data`, directories `2750`, files `0640`.

### Add Git to Existing Site

For sites created without Git integration:

```bash
sudo sitectl add-git oldsite.example.com
```

You will be prompted for the repository URL. If the site already has a `.git` directory, the clone step is skipped and permissions are enforced on the existing files.

### Skip Git Integration

To create a site without Git integration (placeholder `index.html`):

```bash
sudo sitectl add example.com --without-git
```

### Requirements

For Git integration to work, you need:

1. **SSH Deploy Key** - Located at `/root/.ssh/github_account`
   - Generate with: `ssh-keygen -t ed25519 -f /root/.ssh/github_account -C "deploy@yourserver"`
   - Add the public key to your GitHub repository: Settings -> Deploy keys -> Add deploy key

2. **site-pull Script** - Installed to `/usr/local/bin/site-pull` by `sudo sitectl install`
   - Pulls the latest commit from GitHub, runs Composer if present, and re-applies permissions
   - Reads `SITE_OWNER` from `/etc/sitectl/config` at runtime — no per-site edits needed
   - To reinstall or update: run `sudo sitectl install` again (idempotent)

## Creating a Cloudflare API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
1. Click **Create Token**
1. Use **Custom Token** template
1. Set permissions:
- `Zone - Zone - Read`
- `Zone - DNS - Edit`
1. Zone Resources: `Include - All Zones` (or select specific zones)
1. Click **Continue to summary** -> **Create Token**
1. Copy the token to your `/etc/sitectl/config`

## File Locations

|Path                                            |Description                         |
|------------------------------------------------|------------------------------------|
|`/etc/sitectl/config`                           |Configuration file                  |
|`/etc/sitectl/state/`                           |State files for resumable operations|
|`/var/log/sitectl.log`                          |Log file                            |
|`/var/www/<domain>/`                            |Document roots                      |
|`/etc/apache2/sites-available/<domain>.conf`    |HTTP vhost config                   |
|`/etc/apache2/sites-available/<domain>-ssl.conf`|HTTPS vhost config                  |
|`/etc/letsencrypt/live/<domain>/`               |SSL certificates                    |
|`/usr/local/bin/site-pull`                      |Deployment script (installed by `sitectl install`)|

## Troubleshooting

### SSL certificate fails

**DNS timed out before resolving — increase the timeout:**

```bash
sudo sitectl add example.com --wait 300
```

**If DNS is already configured:**

```bash
sudo sitectl add example.com --skip-dns
```

**Check the logs:**

```bash
tail -f /var/log/sitectl.log
```

### DNS not updating

- Verify your API token has `Zone:Read` and `DNS:Edit` permissions
- Ensure the zone is active in Cloudflare
- Check that the root domain is correct (e.g., `example.com` not `www.example.com`)

### SITE_OWNER user not found

If `sitectl add` (or `sitectl add-git`) fails with a message about `SITE_OWNER` not existing:

1. Verify the user is set correctly in `/etc/sitectl/config`
2. Confirm the user exists: `id <username>`
3. Create the user if needed, then retry the command

### Resume a failed setup

Just run the same `add` command again. sitectl tracks progress and will skip completed steps:

```bash
sudo sitectl add example.com
# If it fails at SSL step, just run again:
sudo sitectl add example.com
# It will skip DNS and Apache setup, retry SSL
```

### Check what went wrong

```bash
sudo sitectl status example.com
```

## How It Works

1. **DNS Setup**: Uses Cloudflare API to create/update A and AAAA records pointing to your server's public IPs (auto-detected). Enables Cloudflare proxy by default.
1. **Document Root**: Creates `/var/www/<domain>/` with a placeholder index.html (non-git sites) or clones your repository (git sites).
1. **Apache HTTP**: Creates and enables a virtual host on port 80 with HTTPS redirect.
1. **DNS Propagation Wait**: Polls Cloudflare's resolver (`1.1.1.1`) every 5 seconds and proceeds as soon as the record is visible. `DNS_WAIT_SECONDS` (default 180) is the maximum wait before the script errors out, not a fixed delay.
1. **SSL Certificate**: Uses certbot with DNS-01 challenge via Cloudflare to obtain a Let's Encrypt certificate. This method works even if HTTP isn't publicly accessible yet.
1. **Apache HTTPS**: Creates and enables a virtual host on port 443 with security headers.
1. **Permissions**: Recursively sets ownership to `SITE_OWNER:www-data`, directories to `2750`, and files to `0640`.

## License

MIT License - feel free to use, modify, and distribute.

## Contributing

Issues and pull requests welcome!
