# sitectl

A command-line tool for managing Apache virtual hosts on Ubuntu/Debian LAMP servers with automatic Cloudflare DNS and Let’s Encrypt SSL configuration.

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
- **Let’s Encrypt SSL** - Uses DNS-01 challenge via Cloudflare (works even before DNS propagates publicly)
- **IPv4 + IPv6 support** - Auto-detects and configures both
- **Smart domain handling**:
  - Subdomains: `api.example.com` → detects `example.com` zone automatically
  - Root domains: `example.com` → also configures `www.example.com` with redirect
- **Cloudflare proxy** - Enabled by default (orange cloud), optional `--no-proxy` flag
- **Resumable operations** - If setup fails midway, just run again to continue
- **Safe removal** - Cleans up DNS, SSL, and Apache but preserves your files
- **Preview mode** - Use `--dry-run` to see what changes would be made without executing them
- **Security hardened** - Safe config parsing, domain validation, and concurrency protection
- **Concurrent execution protection** - Lock files prevent multiple operations on the same domain

## Requirements

- Ubuntu/Debian server with Apache
- Cloudflare account managing your domain’s DNS
- Cloudflare API token with `Zone:Read` and `DNS:Edit` permissions

## Quick Start

### 1. Install

```bash
# Download the script
curl -O https://raw.githubusercontent.com/beny698/sitectl/main/sitectl

# Run the installer (installs dependencies, creates config)
sudo bash sitectl install
```

### 2. Configure

Edit `/etc/sitectl/config` with your Cloudflare API token and email:

```bash
sudo nano /etc/sitectl/config
```

```bash
# Cloudflare API Token (Zone:Read, DNS:Edit permissions)
CF_TOKEN="your_cloudflare_api_token_here"

# Email for Let's Encrypt registration
TLS_EMAIL="you@example.com"

# Optional: DNS propagation wait time in seconds (default: 180)
DNS_WAIT_SECONDS=180

# Optional: Server IP addresses (auto-detected if not set)
# SERVER_IPV4="1.2.3.4"
# SERVER_IPV6="2001:db8::1"
```

**Note**: Server IPs are auto-detected by default. Only set `SERVER_IPV4` or `SERVER_IPV6` if auto-detection doesn't work in your environment.

### 3. Add Your First Site

```bash
sudo sitectl add yourdomain.com
```

That’s it! Your site is now live at `https://yourdomain.com` with:

- Cloudflare DNS configured (A + AAAA records, proxy enabled)
- Apache virtual host running
- Valid SSL certificate from Let’s Encrypt
- Automatic HTTP → HTTPS redirect
- www → non-www redirect (for root domains)

## Commands

### `sitectl add <domain> [options]`

Create a new site with DNS, Apache, and SSL. **Git integration is enabled by default** - you'll be prompted for GitHub repository URL and slug.

```bash
# Add a subdomain with Git integration (default)
sudo sitectl add api.example.com
# Prompts for: GitHub repo URL, slug (suggests "api")
# Creates: site_api user, clones repo, updates site-pull

# Add a root domain (also configures www)
sudo sitectl add example.com

# Skip Git integration
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

Add Git integration to an existing site.

```bash
sudo sitectl add-git oldsite.example.com
# Prompts for: GitHub repo URL, slug
# Creates: Linux user, clones/registers repo, updates site-pull
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
  ─────────────────────────────────────────────────────────────────
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

- Installs dependencies (apache2, certbot, python3-certbot-dns-cloudflare, curl, jq)
- Enables required Apache modules (ssl, headers, rewrite, http2)
- Creates configuration file
- Sets up certbot renewal hooks
- Installs sitectl to `/usr/local/bin/`

```bash
sudo bash sitectl install
```

### `sitectl help`

Display full documentation.

## Git Integration

sitectl includes built-in Git integration for automatic repository deployment. When enabled (default), it creates a Linux user, clones your GitHub repository, and integrates with the site-pull deployment script.

### Quick Start with Git

```bash
# Add a new site with Git integration (default behavior)
sudo sitectl add api.example.com

# You'll be prompted for:
# - GitHub repository URL (SSH format): git@github.com:user/repo.git
# - Slug for Linux user (default suggested): api
```

This automatically:
1. Creates Linux user `site_api` with home directory at `/var/www/api.example.com`
2. Clones your repository into the document root
3. Updates `/usr/local/bin/site-pull` with domain mapping
4. Sets permissions (directories: 2750, files: 0640)

### Deploy Updates

After making changes to your repository:

```bash
sudo /usr/local/bin/site-pull api.example.com
```

This pulls the latest code from GitHub and sets the correct permissions.

### Add Git to Existing Site

For sites created without Git integration:

```bash
sudo sitectl add-git oldsite.example.com
```

You'll be prompted for the repository URL and slug. If the site already has a `.git` directory, it will just register with site-pull without cloning.

### Skip Git Integration

To create a site without Git integration (traditional behavior):

```bash
sudo sitectl add example.com --without-git
```

### Requirements

For Git integration to work, you need:

1. **SSH Deploy Key** - Located at `/root/.ssh/github_account`
   - Generate with: `ssh-keygen -t ed25519 -f /root/.ssh/github_account -C "deploy@yourserver"`
   - Add the public key to your GitHub repository: Settings → Deploy keys → Add deploy key

2. **site-pull Script** - Located at `/usr/local/bin/site-pull`
   - This script handles deployment and permission management
   - sitectl automatically updates it with new domain mappings

### How It Works

1. **Linux User**: Each site gets a dedicated user `site_<slug>` (e.g., `site_api`)
   - User is added to `www-data` group for Apache access
   - Home directory is the site's document root

2. **Repository Clone**: Uses SSH with the deploy key
   - Environment: `GIT_SSH_COMMAND="ssh -i /root/.ssh/github_account"`
   - Clones directly into `/var/www/<domain>/`

3. **site-pull Integration**: Adds domain mapping to the case statement
   ```bash
   api.example.com) OWNER="site_api" ;;
   ```

4. **Permissions**: Matches site-pull script
   - Directories: `chmod 2750` (setgid for group inheritance)
   - Files: `chmod 0640` (readable by owner and group)
   - Owner: `site_<slug>:site_<slug>`

### site-pull Script Versioning

sitectl automatically versions the `/usr/local/bin/site-pull` script to allow easy rollback if issues occur:

- **Automatic Backup**: Before each modification (skipped if the domain already exists), a timestamped backup is created in `/var/backups/sitectl/site-pull/`
- **Backup Format**: `site-pull.YYYYMMDD-HHMMSS-NNNNNNNNN` (e.g., `site-pull.20260130-143025-123456789`)
- **Automatic Cleanup**: Last 10 backups are kept; older ones are automatically removed
- **Automatic Rollback**: If an update fails, the previous version is automatically restored

#### Manual Rollback

If you need to manually restore a previous version:

```bash
# List available backups
ls -lh /var/backups/sitectl/site-pull/

# Restore from a specific backup
sudo cp /var/backups/sitectl/site-pull/site-pull.20260130-143025-123456789 \
     /usr/local/bin/site-pull
```

### Slug Naming

The slug determines the Linux username (`site_<slug>`):

- Subdomains: First part is suggested (e.g., `api.example.com` → `api`)
- Root domains: Domain name is suggested (e.g., `example.com` → `example`)
- Must be unique (validated against existing users)
- Alphanumeric, hyphens, and underscores only

## Creating a Cloudflare API Token

1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
1. Click **Create Token**
1. Use **Custom Token** template
1. Set permissions:
- `Zone - Zone - Read`
- `Zone - DNS - Edit`
1. Zone Resources: `Include - All Zones` (or select specific zones)
1. Click **Continue to summary** → **Create Token**
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
|`/var/backups/sitectl/site-pull/`               |site-pull script backups (versioning)|

## Troubleshooting

### SSL certificate fails

**Wait longer for DNS propagation:**

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

1. **DNS Setup**: Uses Cloudflare API to create/update A and AAAA records pointing to your server’s public IPs (auto-detected). Enables Cloudflare proxy by default.
1. **Document Root**: Creates `/var/www/<domain>/` with a placeholder index.html.
1. **Apache HTTP**: Creates and enables a virtual host on port 80 with HTTPS redirect.
1. **DNS Propagation Wait**: Waits 3 minutes (configurable) for DNS to propagate.
1. **SSL Certificate**: Uses certbot with DNS-01 challenge via Cloudflare to obtain a Let’s Encrypt certificate. This method works even if HTTP isn’t publicly accessible yet.
1. **Apache HTTPS**: Creates and enables a virtual host on port 443 with security headers.

## License

MIT License - feel free to use, modify, and distribute.

## Contributing

Issues and pull requests welcome!
