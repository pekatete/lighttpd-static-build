# lighttpd-static-build
#!/bin/bash
# Master Build Script for Static Lighttpd 1.4.82 + OpenSSL 3.5.5 LTS
# Targets: x86_64-linux-musl | Support EOL: April 2030

set -e # Exit on error

# 1. Environment Setup
```
export MUSL_PATH="${HOME}/musl-libs"
export PATH="${MUSL_PATH}/bin:${PATH}"
mkdir -p "${MUSL_PATH}/include/sys"
mkdir -p "${MUSL_PATH}/lib"
```

# Create Dummy Header to block Glibc leakage

```
touch "${MUSL_PATH}/include/sys/cdefs.h"

echo "--- Building OpenSSL 3.5.5 LTS ---"
cd ~/opensslbuild/openssl-3.5.5
make clean || true
CC="musl-gcc" ./Configure linux-x86_64 no-shared no-async no-afalgeng \
    -DOPENSSL_NO_SECURE_MEMORY --prefix="${MUSL_PATH}" --openssldir="${MUSL_PATH}/ssl"
make -j$(nproc)
make install_sw

echo "--- Configuring Lighttpd 1.4.82 ---"
cd ~/opensslbuild/lighttpd-1.4.82
make distclean || true
```

# Recreate Version Stamp

```
cat <<EOF > src/versionstamp.h
#define LIGHTTPD_VERSION_ID 10482
#define LIGHTTPD_VERSION_STRING "1.4.82-musl-static"
#define REPO_VERSION ""
EOF
```

# Recreate Static Plugin Map

```
cat <<EOF > src/plugin-static.h
PLUGIN_INIT(mod_access)
PLUGIN_INIT(mod_accesslog)
PLUGIN_INIT(mod_alias)
PLUGIN_INIT(mod_auth)
PLUGIN_INIT(mod_cgi)
PLUGIN_INIT(mod_redirect)
PLUGIN_INIT(mod_rewrite)
PLUGIN_INIT(mod_setenv)
PLUGIN_INIT(mod_expire)
PLUGIN_INIT(mod_fastcgi)
PLUGIN_INIT(mod_scgi)
PLUGIN_INIT(mod_proxy)
PLUGIN_INIT(mod_openssl)
PLUGIN_INIT(mod_deflate)
PLUGIN_INIT(mod_magnet)
PLUGIN_INIT(mod_maxminddb)
PLUGIN_INIT(mod_dirlisting)
PLUGIN_INIT(mod_indexfile)
PLUGIN_INIT(mod_staticfile)
PLUGIN_INIT(mod_evhost)
PLUGIN_INIT(mod_simple_vhost)
EOF
```

```
CC="musl-gcc" ./configure \
    --build=x86_64-pc-linux-gnu --host=x86_64-unknown-linux-musl \
    --enable-static --disable-shared \
    --with-openssl="${MUSL_PATH}" --with-pcre2="${MUSL_PATH}" \
    --with-zlib="${MUSL_PATH}" --with-zstd="${MUSL_PATH}" \
    --with-bzip2="${MUSL_PATH}" --with-brotli="${MUSL_PATH}" \
    --with-maxminddb="${MUSL_PATH}" --with-lua="${MUSL_PATH}" \
    --without-pcre \
    CPPFLAGS="-I${MUSL_PATH}/include -DLIGHTTPD_STATIC -D_GNU_SOURCE" \
    LDFLAGS="-L${MUSL_PATH}/lib -L${MUSL_PATH}/lib64 -static"

echo "--- Compiling Objects ---"
cd src
make lighttpd-mod_openssl.o lighttpd-mod_deflate.o lighttpd-mod_magnet.o \
     lighttpd-mod_magnet_cache.o lighttpd-mod_maxminddb.o lighttpd-mod_dirlisting.o \
     lighttpd-mod_auth.o lighttpd-mod_accesslog.o lighttpd-mod_proxy.o \
     lighttpd-mod_cgi.o lighttpd-mod_fastcgi.o lighttpd-algo_hmac.o \
     lighttpd-mod_expire.o lighttpd-mod_evhost.o lighttpd-mod_simple_vhost.o
```

# Build Static-Aware Plugin Loader & Auth API

```
musl-gcc -DHAVE_CONFIG_H -DLIGHTTPD_STATIC -I. -I.. -I${MUSL_PATH}/include -g -O2 -c plugin.c -o lighttpd-plugin.o
musl-gcc -DHAVE_CONFIG_H -DLIGHTTPD_STATIC -I. -I.. -I${MUSL_PATH}/include -g -O2 -c mod_auth_api.c -o lighttpd-mod_auth_api.o
```

# Build Core Objects

```
make lighttpd || true

echo "--- Final Link ---"
musl-gcc -g -O2 -pipe -Wall -W -Wshadow -pedantic -o lighttpd \
    lighttpd-server.o lighttpd-response.o lighttpd-connections.o lighttpd-h1.o \
    lighttpd-sock_addr_cache.o lighttpd-network.o lighttpd-network_write.o \
    lighttpd-fdevent_impl.o lighttpd-http_range.o lighttpd-data_config.o \
    lighttpd-configfile.o lighttpd-configparser.o lighttpd-mod_rewrite.o \
    lighttpd-mod_redirect.o lighttpd-mod_access.o lighttpd-mod_alias.o \
    lighttpd-mod_indexfile.o lighttpd-mod_staticfile.o lighttpd-mod_setenv.o \
    lighttpd-mod_expire.o lighttpd-mod_simple_vhost.o lighttpd-mod_evhost.o \
    lighttpd-mod_fastcgi.o lighttpd-mod_scgi.o lighttpd-mod_proxy.o \
    lighttpd-mod_cgi.o lighttpd-base64.o lighttpd-buffer.o lighttpd-burl.o \
    lighttpd-log.o lighttpd-http_header.o lighttpd-http_kv.o lighttpd-keyvalue.o \
    lighttpd-chunk.o lighttpd-http_chunk.o lighttpd-fdevent.o \
    lighttpd-fdevent_fdnode.o lighttpd-gw_backend.o lighttpd-stat_cache.o \
    lighttpd-http_etag.o lighttpd-array.o lighttpd-algo_md5.o lighttpd-algo_sha1.o \
    lighttpd-algo_splaytree.o lighttpd-configfile-glue.o lighttpd-http-header-glue.o \
    lighttpd-http_cgi.o lighttpd-http_date.o lighttpd-plugin.o lighttpd-reqpool.o \
    lighttpd-request.o lighttpd-sock_addr.o lighttpd-rand.o lighttpd-fdlog_maint.o \
    lighttpd-fdlog.o lighttpd-sys-setjmp.o lighttpd-ck.o lighttpd-fdevent_win32.o \
    lighttpd-fs_win32.o \
    lighttpd-mod_openssl.o lighttpd-mod_deflate.o lighttpd-mod_magnet.o \
    lighttpd-mod_magnet_cache.o lighttpd-mod_maxminddb.o lighttpd-mod_dirlisting.o \
    lighttpd-mod_auth.o lighttpd-mod_auth_api.o lighttpd-mod_accesslog.o \
    lighttpd-algo_hmac.o \
    -Wl,--export-dynamic -static \
    -L"${MUSL_PATH}/lib" -L"${MUSL_PATH}/lib64" \
    -lssl -lcrypto -lpcre2-8 -lz -lzstd -lbz2 -lbrotlienc -lbrotlidec -lbrotlicommon -lmaxminddb -llua -lm -ldl -lpthread -latomic

strip --strip-all lighttpd

echo "Build Successful: $(pwd)/lighttpd"
file lighttpd
```
