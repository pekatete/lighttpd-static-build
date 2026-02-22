# Master Build Script for Static Lighttpd 1.4.82 + OpenSSL 3.5.5 LTS targeting x86_64-linux-musl  (WSL)

# Install MUSL (building on WSL Ubuntu)
```
sudo apt update
sudo apt install -y musl-tools musl-dev
```

# Verify MUSL install
```
musl-gcc --version
```

1. Create a folder in home directory named **lighttpdOpensslBuild**
2. Download Lighttpd 1.4.82 from: https://download.lighttpd.net/lighttpd/releases-1.4.x/lighttpd-1.4.82.tar.gz
3. Download OpenSSL 3.5.5 from: https://github.com/openssl/openssl/releases/download/openssl-3.5.5/openssl-3.5.5.tar.gz
4. Extract both tar archives into the folder above (new folders named **lighttpd-1.4.82** and **openssl-3.5.5** will be created)
5. Copy the build.sh script into the above folder and: **chmod +x build.sh** then run it thus: **./build.sh**.

NOTE: The static build includes the following 21 modules

mod_access,
mod_accesslog,
mod_alias,
mod_auth,
mod_cgi,
mod_redirect,
mod_rewrite,
mod_setenv,
mod_expire,
mod_fastcgi,
mod_scgi,
mod_proxy,
mod_openssl,
mod_deflate,
mod_magnet,
mod_maxminddb,
mod_dirlisting,
mod_indexfile,
mod_staticfile,
mod_evhost,
mod_simple_vhost

Ensure you address the requirements of the Open SSL 3.5.x series, which mandates a C99-compliant compiler and the presence of Linux development headers
```
sudo apt update
sudo apt install build-essential linux-libc-dev libc6-dev
```

On Ubuntu WSL, if it complains about bits/ when you run the script, install the multiarch support package (even if you are on 64-bit):
```
sudo apt install gcc-multilib
```

If you receive a warning thus: Clock skew detected. Your build may be incomplete - Sync the hardware clock
```
sudo hwclock -s
```
If the warning persists (clock skew), touch all files in the project directory to reset their modiication times
```
find . -type f -exec touch {} +
```

# LICENSE: MIT
