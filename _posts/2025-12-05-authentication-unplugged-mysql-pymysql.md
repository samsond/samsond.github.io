---
layout: post
title:  "Authentication Unplugged: When `mysql` Works but PyMySQL Fails"
date:   2025-12-05 18:00:00 -0800
categories: database
description: "Why your MySQL CLI logs in just fine while your Python app times out with mysterious authentication errors—and how to debug it."
tags: [mysql, python, pymysql, rds-proxy, authentication]
comments: false
last_modified_at: 2025-12-05 18:00:00 -0800
---

## Executive Summary

You're connecting to MySQL (or Aurora MySQL) through a proxy. The `mysql` CLI connects without issue, but your Python script using PyMySQL and SQLAlchemy throws:

```
pymysql.err.OperationalError: (1045, "Access denied for user 'admin'@'172.30.0.250' (using password: YES)")
```

**Same host. Same user. Same password. Different outcome.**

In this post, we'll explore why by:

- Walking through the MySQL handshake and authentication plugin negotiation.
- Comparing the official MySQL C client (`mysql` CLI) with the pure-Python PyMySQL implementation.
- Using a real failure as a case study.
- Following the stack trace into PyMySQL's `_auth.py`.
- Understanding the root cause and how to debug it.

By the end, you'll have a mental model for "why mysql works but PyMySQL doesn't" when dealing with authentication, IAM, or proxies.

---

## The Setup: One Proxy, Two Clients

The environment looks like this:

**PyMySQL path:**
```
[ PyMySQL / SQLAlchemy ]
           |
           v
[ RDS Proxy or MySQL Proxy ]
           |
           v
[ Aurora MySQL ]
```

**MySQL CLI path:**
```
[ mysql CLI ]
      |
      v
[ RDS Proxy / DB ]
```

From the same EC2 instance, the `mysql` CLI connects successfully, but the Python script fails with ERROR 1045.

---

## The Python Script & Error

The Python script is straightforward:

```python
conn = pymysql.connect(
            host=HOST,
            port=PORT,
            user=USER,
            password=PASSWORD,
            database=DB_NAME,
            ssl={"ca": str(CA_BUNDLE)},
)
```

The traceback tells the story:

```
Traceback (most recent call last):
  File "proxy-connection-with-pymysql.py", line 3, in <module>
    connection = pymysql.connect(...)
  File ".../pymysql/connections.py", line 669, in connect
    self._request_authentication()
  File ".../pymysql/connections.py", line 1003, in _process_auth
    return _auth.caching_sha2_password_auth(self, auth_packet)
  File ".../pymysql/_auth.py", line 222, in caching_sha2_password_auth
    pkt = _roundtrip(conn, scrambled)
  File ".../pymysql/protocol.py", line 219, in raise_for_error
    err.raise_mysql_exception(self._data)

pymysql.err.OperationalError: (1045, "Access denied for user 'admin'@'172.30.0.250' (using password: YES)")
```

**Key observations:**

- The server chose the `caching_sha2_password` plugin.
- PyMySQL correctly selected the handler and computed a scrambled password.
- The server replied with error 1045 (Access denied).

So, PyMySQL is using the right plugin, but the server is rejecting the credential.

---

## Authentication Plugins: Where Things Can Drift

Modern MySQL and Aurora MySQL use pluggable authentication. The common ones:

- **`mysql_native_password`** – Legacy auth, deprecated in newer versions but still widely used.
- **`caching_sha2_password`** – Default in MySQL 8+ for new accounts. Uses SHA-256 hashing with server-side caching.
- **`mysql_clear_password`** – Sends passwords in cleartext (over TLS only) for use cases like IAM tokens.
- **`AWSAuthenticationPlugin`** – Used when you enable IAM database authentication for MySQL on RDS/Aurora.

The server chooses a plugin for each account; the client must respond accordingly.

When you:

- Change `default_authentication_plugin` on the server.
- Move an account from `mysql_native_password` to `caching_sha2_password`.
- Introduce IAM authentication via RDS Proxy.

…you're also changing which code path runs inside the client. **This is where mysql CLI and PyMySQL can diverge.**

---

## How the `mysql` CLI Handles `caching_sha2_password`

The `mysql` CLI is built on the official MySQL C client library. Its authentication logic lives in `client_authentication.h` and `client_authentication.cc`.

For `caching_sha2_password`, the CLI:

1. **Detects the connection security** – Is TLS enabled?
2. **Chooses an auth method** – Use RSA, send over TLS, or use cached auth.
3. **Follows the server's lead** – The handshake packet tells it which plugin to use.

As long as you're on a reasonably recent MySQL client and connect with:

```sh
mysql \
  -h database1-proxy.proxy-....us-west-2.rds.amazonaws.com \
  -P 3306 \
  -u admin \
  --ssl-ca=amazon-trust-bundle.pem \
  -p
```

…the CLI will handle the handshake as the server expects. You flip a flag on the server side, and the CLI just follows along.

---

## How PyMySQL Handles the Same Handshake

PyMySQL is pure Python and implements the wire protocol itself, including authentication plugins.

The implementation lives in `pymysql/_auth.py`. When the server requests `caching_sha2_password`, PyMySQL:

1. **Parses the handshake packet** – Extracts the server nonce/salt.
2. **Computes a scrambled password** – Uses the algorithm from the MySQL protocol.
3. **Sends the scramble** – Via `_roundtrip(conn, scrambled)`.
4. **Handles full auth if needed** – May request the public key or rely on TLS.

---

## Where the 1045 Comes From

The stack trace shows:

- The server requested `caching_sha2_password`.
- PyMySQL responded correctly with the scrambled password.
- The server rejected the credential → **Error 1045**.

**Common reasons this happens only for PyMySQL:**

### 1. Account Plugin Changed

You switched the `admin` user from `mysql_native_password` to `caching_sha2_password` or `AWSAuthenticationPlugin`.

- The `mysql` CLI now uses a different handshake or sends an IAM token.
- Your Python script still uses a static password with the old assumptions.

### 2. Proxy Enforces Different Rules

RDS Proxy can require IAM auth, even if the underlying DB user still exists with a password.

- From PyMySQL's perspective, the handshake says "use `caching_sha2_password`", so it does.
- The proxy validates the credential and rejects it (1045).

### 3. Password Changed at the Proxy Level

If RDS Proxy is using a rotated secret or a token behind the scenes, the old static password in your code no longer matches.

### 4. Proxy/Server Configuration Mismatch

The proxy is configured to use a different auth plugin than the underlying database, leading to incompatibilities.

---

## Comparing the Two Code Paths

### MySQL CLI

```sh
mysql \
  -h <proxy-endpoint> \
  -u admin \
  -p
```

**What happens:**

- Uses `caching_sha2_password_auth_client` from the C client library.
- Detects TLS and sends the password securely.
- Works transparently, even if the server changes plugins.

**For IAM/RDS Proxy:**

```sh
mysql \
  -h <proxy-endpoint> \
  -u <db_user> \
  --enable-cleartext-plugin \
  -p <iam_token>
```

- Uses `mysql_clear_password` plugin.
- Sends the IAM token in cleartext over TLS.
- Proxy validates the token and grants access.

### PyMySQL

```python
conn = pymysql.connect(
    host=ENDPOINT,
    user='admin',
    password='my_password',
    port=3306,
    database='mydb'
)
```

**What happens:**

- Uses `caching_sha2_password_auth` from `_auth.py`.
- Computes the scrambled password and sends it.
- If the server changed plugins, this may fail.

**For IAM/RDS Proxy:**

```python
import boto3

# Generate IAM token
client = boto3.client('rds')
token = client.generate_db_auth_token(
    DBHostname=ENDPOINT,
    Port=3306,
    DBUser=USER
)

# Connect with the token
conn = pymysql.connect(
    host=ENDPOINT,
    user=USER,
    password=token,  # IAM token
    port=3306,
    database=DBNAME,
    ssl_ca="ssl-ca-bundle.pem",
    ssl_verify_identity=True,
    auth_plugin_map={"mysql_clear_password": None},
)
```

**Critical bits:**

- The "password" is actually an IAM token (temporary, 15-minute expiry).
- The plugin is mapped to `mysql_clear_password`.
- TLS is **required**.

---

## Practical Debugging Steps

Here's a checklist to use when `mysql` connects but PyMySQL fails.

### 1. Check the Account's Authentication Plugin

Run this on the database (via an admin connection that still works):

```sql
SELECT user, host, plugin FROM mysql.user WHERE user = 'admin';
```

**Example output:**

```
+-------+-----------+----------------------+
| user  | host      | plugin               |
+-------+-----------+----------------------+
| admin | %         | caching_sha2_password |
+-------+-----------+----------------------+
```

If the plugin is `caching_sha2_password` or `AWSAuthenticationPlugin`, but you assumed `mysql_native_password`, you've found a clue.

### 2. Confirm if a Proxy + IAM is Involved

Ask yourself:

- Is the hostname actually an RDS Proxy endpoint (e.g., `database-proxy.proxy-....us-west-2.rds.amazonaws.com`)?
- Is IAM authentication required on that proxy?
- Are you using Secrets Manager or AWS IAM tokens?

If yes to all, **you shouldn't be sending a static password from PyMySQL**. Instead:

- Generate an IAM token via boto3.
- Use `auth_plugin_map={"mysql_clear_password": None}`.
- Enforce TLS.

### 3. Install the Right PyMySQL Extras

```sh
python3 -m pip install "PyMySQL[rsa]"
```

Without the `cryptography` package, certain `caching_sha2_password` flows fail.

### 4. Turn On Debug Logging

Inside your Python script:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("pymysql").setLevel(logging.DEBUG)
```

**What to look for:**

- Which plugin the server asked for.
- When the handshake completes or fails.
- When the server returns an error packet.

### 5. Compare the "Good" and "Bad" Paths

**Side-by-side checklist:**

| Aspect | mysql CLI | PyMySQL |
|--------|-----------|---------|
| Host | Same? | Same? |
| Port | Same? | Same? |
| User | Same? | Same? |
| Plugin (from mysql.user) | ? | ? |
| Auth method | Password / IAM token? | Password / IAM token? |
| TLS enabled? | Yes? | Yes? |

Once you align these, 1045 usually disappears.

---

## Architecture Diagram: Static Password vs IAM Token

**Old world: static password, native plugin**

```
[ PyMySQL / mysql ]
      |
      |  password, mysql_native_password
      v
[ MySQL / Aurora ]
```

**New world: IAM + proxy + caching_sha2_password**

```
[ mysql CLI ]                    [ PyMySQL script ]
      |                                   |
      |  IAM token via mysql_clear_password  |  static password via caching_sha2_password
      v                                   v
      [ RDS Proxy (IAM Required) ]
                  |
                  v
             [ Aurora MySQL ]
```

The CLI follows the new world; the Python script is still living in the old one. The stack trace is simply the server saying, "I've moved on. Your old password doesn't live here anymore."

---

## Code References

For deeper dives, here are the key resources:

### MySQL C Client: `caching_sha2_password`

- `caching_sha2_password_auth_client` in [client_authentication.h](https://github.com/mysql/mysql-server/blob/trunk/include/mysql/client_authentication.h)
- Implementation in [client_authentication.cc](https://github.com/mysql/mysql-server/blob/trunk/sql-common/client_authentication.cc)

### PyMySQL: Authentication Plugins

- Auth implementations in [`_auth.py`](https://github.com/PyMySQL/PyMySQL/blob/master/pymysql/_auth.py)
- Connection logic in [`connections.py`](https://github.com/PyMySQL/PyMySQL/blob/master/pymysql/connections.py)

### MySQL Docs: Caching SHA-2 Authentication

- [MySQL Caching SHA-2 Authentication](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)

### AWS: IAM Authentication + PyMySQL

- [Using IAM Authentication with Python](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Python.html)
- [RDS Proxy IAM Authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-iam-setup.html)

---

## Conclusion

When the `mysql` CLI connects but PyMySQL fails with 1045, the root cause is rarely a bug in either client. Instead, it's usually:

- An **authentication plugin change** (e.g., to `caching_sha2_password`, IAM, or `AWSAuthenticationPlugin`).
- A **proxy enforcing updated rules**.
- A **Python script still assuming the old world** of static passwords.

By understanding:

- How the MySQL handshake chooses an auth plugin.
- Where the official C client code lives and how it responds.
- How PyMySQL's `_auth.py` implements the same dance.

…you can debug these issues **on purpose instead of by superstition**.

So the next time Prod throws a 1045 at 2 AM, you'll know exactly where to look: not just at the password, but at the plugin, proxy, and path that carried it.
