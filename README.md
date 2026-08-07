# Custom CSS for Jellyfin via Caddy

This setup assumes Caddy has network access to Jellyfin, and that both containers can reach a shared folder of static assets (CSS, fonts, etc.) mounted read-only into Caddy.

Example using Podman with Caddy in front of Jellyfin, both containers on a shared network.

## Podman network

Caddy needs to reach Jellyfin by name, so both containers must share a network:

```bash
podman network create jellynet
```

## Jellyfin container

```ini
[Container]
ContainerName=jellyfin
Image=docker.io/jellyfin/jellyfin:latest

Network=jellynet

...
```

## Caddyfile

```caddyfile
(style_handler) {
    handle /style/* {
        root * /style
        uri strip_prefix /style
        file_server
    }
}

http://{{ ip or hostname }} {
    import style_handler
    reverse_proxy jellyfin:8096
}
```

`jellyfin` here refers to Jellyfin's `ContainerName`; Podman's DNS on a shared network resolves it automatically, so no hardcoded IP is needed.

## Caddy container

```ini
[Container]
ContainerName=caddy
Image=docker.io/library/caddy:latest

Network=jellynet

Volume={{ path to Caddyfile }}:/etc/caddy/Caddyfile:ro,z
Volume={{ path to style assets on host }}:/style:ro,z

PublishPort=8080:80/tcp
PublishPort=8443:443/tcp

...
```

## Jellyfin configuration

Set the Custom CSS field under **Dashboard → General → Branding** (`http://{{ ip }}:8080/web/#/dashboard/branding`; the exact URL fragment may shift between Jellyfin versions) to:

```css
@import url("http://{{ ip }}:8080/style/stylesheet.css");
```

Use an **absolute URL**, not a relative one as some clients fail to resolve relative `url()` paths for CSS injected dynamically into the page, but resolve absolute URLs correctly.
