# Nginx with HTTP/3, KTLS, zstd, and ModSecurity for EL7

This repository provides an unofficial Nginx RPM build for EL7.

This build includes HTTP/3, OpenSSL 3.5.7, KTLS, zstd compression, and ModSecurity WAF integration.

## Features

- Nginx 1.30.4
- OpenSSL 3.5.7
- HTTP/3 / QUIC
- KTLS
- zstd compression via zstd-nginx-module
- ModSecurity support via ModSecurity-nginx

## Repository Layout

```text
RPMS/      - Built binary RPM packages
SRPM/      - Source RPM package
SPEC/      - Spec file used for building
Logs/      - Build logs from mock
README.md  - Documentation and verification notes
```

## Important Notice

This repository is unofficial.

It is not provided, maintained, endorsed, or supported by AlmaLinux OS Foundation, CentOS, Red Hat, Nginx, OWASP, Trustwave, OpenSSL, zstd, ModSecurity, or any upstream project.

Use these RPM packages at your own risk.

No warranty is provided. Please test carefully in a verification environment before using them in production.

## Requirement

Before installing this Nginx RPM, install the ModSecurity RPM first.

ModSecurity RPM for EL7:

```text
https://github.com/redadmin-k/modsecurity-el7
```

Install the ModSecurity RPM first:

```bash
git clone https://github.com/redadmin-k/modsecurity-el7.git
cd modsecurity-el7
sudo yum localinstall $(find RPMS -type f -name '*.rpm' | sort)
```

This Nginx build expects `libmodsecurity.so.3` to be available under:

```text
/opt/modsecurity/lib64/libmodsecurity.so.3
```

Confirm that ModSecurity is available:

```bash
ldconfig -p | grep modsecurity
```

Expected result example:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

If it does not appear, run:

```bash
sudo ldconfig
```

## Installation

Install this Nginx RPM after installing the ModSecurity RPM.

```bash
git clone https://github.com/redadmin-k/nginx-http3-ktls-zstd-modsecurity-el7.git
cd nginx-http3-ktls-zstd-modsecurity-el7
sudo yum localinstall $(find RPMS -type f -name '*.rpm' | sort)
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

## Build Environment

- OS: EL7 x86_64
- Build Tool: mock
- Mock Config: centos+epel-7-x86_64
- Nginx Version: 1.30.3
- OpenSSL Version: 3.5.7
- ModSecurity Version: 3.0.16

Additional modules:

- ModSecurity-nginx
- zstd-nginx-module

## Build Details

This build uses OpenSSL 3.5.7 for TLS, HTTP/3, QUIC, and KTLS support.

Check the OpenSSL version reported by Nginx:

```bash
nginx -V 2>&1 | grep -E 'built with OpenSSL|running with OpenSSL'
```

Expected output example:

```text
built with OpenSSL 3.5.7
```

Confirm that Nginx was built with HTTP/3, ModSecurity, zstd, OpenSSL, and KTLS support:

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'http_v3|ModSecurity|zstd|openssl|ktls|compat'
```

Expected output should include:

```text
--with-compat
--with-http_v3_module
--add-module=ModSecurity-nginx-1.0.4
--add-module=zstd-nginx-module-0.1.1
--with-openssl=openssl-3.5.7
--with-openssl-opt=enable-ktls
```

Confirm that Nginx resolves `libmodsecurity.so.3`:

```bash
ldd /usr/sbin/nginx | grep modsecurity
```

Expected output:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

Confirm that Nginx resolves zstd:

```bash
ldd /usr/sbin/nginx | grep zstd
```

The Nginx binary should not contain an RPATH or RUNPATH for `/opt/modsecurity/lib64`.

Check:

```bash
readelf -d /usr/sbin/nginx | grep -E 'RPATH|RUNPATH'
```

Expected result:

```text
# no output
```

## HTTP/3 Verification

HTTP/3 can be verified with a client that supports HTTP/3.

Example:

```bash
curl -I --http3 https://example.com/
```

Expected output example:

```text
HTTP/3 200
server: nginx/1.30.3
alt-svc: h3=":443"; ma=86400
```

Nginx access logs may also show HTTP/3 requests:

```text
"GET / HTTP/3" 200
```

Example Nginx HTTP/3 configuration:

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport;

    server_name example.com;

    ssl_protocols TLSv1.3;
    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    http3 on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

## zstd Verification

zstd compression can be verified with:

```bash
curl -sS -D- -o /dev/null -H 'Accept-Encoding: zstd' https://example.com/
```

Expected output example:

```text
HTTP/1.1 200 OK
Content-Encoding: zstd
```

## KTLS Verification

Linux Kernel TLS can be used for sendfile-based TLS data transfer when supported by the kernel, OpenSSL, cipher suite, and Nginx configuration.

This build is compiled with OpenSSL KTLS support:

```text
--with-openssl-opt=enable-ktls
```

To use KTLS, the Linux kernel TLS module may need to be loaded.

Check whether the module is loaded:

```bash
lsmod | grep '^tls'
```

Load it manually:

```bash
sudo modprobe tls
```

To load it automatically at boot:

```bash
echo tls | sudo tee /etc/modules-load.d/tls.conf
```

Confirm again:

```bash
lsmod | grep '^tls'
```

If KTLS is built into the kernel instead of provided as a module, `lsmod` may not show `tls`.

Nginx configuration may also require enabling KTLS through OpenSSL configuration commands, depending on the environment:

```nginx
ssl_conf_command Options KTLS;
```

A possible runtime indication is `sendfile()` activity from an Nginx worker process:

```text
sendfile(...)
```

KTLS behavior depends on kernel support, OpenSSL support, TLS version, cipher suite, and Nginx configuration.

## ModSecurity Installation

This Nginx package does not include the ModSecurity runtime library.

Install the ModSecurity RPM first:

```bash
git clone https://github.com/redadmin-k/modsecurity-el7.git
cd modsecurity-el7
sudo yum localinstall $(find RPMS -type f -name '*.rpm' | sort)
sudo ldconfig
```

Confirm the installed ModSecurity package:

```bash
rpm -qa | grep modsecurity
```

Confirm installed files:

```bash
rpm -ql modsecurity | grep /opt/modsecurity
```

Confirm library resolution:

```bash
ldconfig -p | grep libmodsecurity
```

Expected output:

```text
libmodsecurity.so.3 => /opt/modsecurity/lib64/libmodsecurity.so.3
```

## OWASP Core Rule Set

OWASP Core Rule Set is not bundled in this package.

CRS should be installed or placed manually.

A typical ModSecurity include layout is:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

For initial operation, OWASP CRS 3.3.x is recommended because it is stable and easier to validate with ModSecurity v3.

CRS 4.x may work, but it should be validated separately before production use.

## ModSecurity Configuration Example

Create the ModSecurity configuration directory:

```bash
sudo mkdir -p /etc/nginx/modsec
```

Copy the recommended ModSecurity configuration:

```bash
sudo cp /opt/modsecurity/share/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
sudo cp /opt/modsecurity/share/unicode.mapping /etc/nginx/modsec/unicode.mapping
```

For initial production testing, start with detection-only mode:

```bash
sudo sed -i 's/^SecRuleEngine .*/SecRuleEngine DetectionOnly/' /etc/nginx/modsec/modsecurity.conf
```

Example `/etc/nginx/modsec/main.conf`:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

Example Nginx server configuration:

```nginx
server {
    listen 80;
    server_name example.local;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

Test Nginx configuration:

```bash
sudo nginx -t
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

## Simple WAF Test

After CRS is installed and blocking mode is enabled, a simple SQL injection test can be used for verification:

```bash
curl -i -H 'Host: example.local' \
  'http://127.0.0.1/?id=1%20UNION%20SELECT%201,2,3'
```

Expected result when blocking is active:

```text
HTTP/1.1 403 Forbidden
```

If the request is detected but still returns `200`, check CRS blocking evaluation and local exclusions, especially rule `949110`.

## Production Notes

Do not enable blocking mode directly in production without verification.

Recommended rollout:

1. Install the ModSecurity RPM.
2. Install this Nginx RPM.
3. Configure OWASP CRS manually.
4. Start with `SecRuleEngine DetectionOnly`.
5. Review audit logs.
6. Add local exclusions for false positives.
7. Enable `SecRuleEngine On` after validation.
8. Keep CRS configuration and local exclusions under version control.

ModSecurity and CRS should be treated as one layer of defense, not as a replacement for application fixes, patching, access control, rate limiting, or regular security updates.

## License

- nginx - BSD 2-Clause License
- OpenSSL - Apache License 2.0
- zstd-nginx-module - BSD 2-Clause License
- ModSecurity - Apache License 2.0
- ModSecurity-nginx - Apache License 2.0

Packaging files in this repository are provided for RPM build and integration testing purposes.

## Notice

This is an unofficial Nginx build for EL7.

This repository is not affiliated with, endorsed by, maintained by, or supported by AlmaLinux OS Foundation, CentOS, Red Hat, Nginx, OpenSSL, OWASP, Trustwave, ModSecurity, zstd, or any upstream project.

Use these RPM packages at your own risk.

