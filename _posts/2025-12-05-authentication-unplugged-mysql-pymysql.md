---
layout: post
title:  "Authentication Unplugged: When MySQL CLI Works but PyMySQL Fails"
date:   2025-12-05 18:00:00 -0800
categories: database
description: "Why your MySQL CLI logs in just fine while your Python app times out with mysterious authentication errors—and how to debug it."
tags: [mysql, python, pymysql, rds-proxy, authentication]
comments: false
last_modified_at: 2025-12-05 18:00:00 -0800
---

## Executive Summary

You're connecting to Aurora MySQL through a proxy. The MySQL CLI connects without issue, but your Python script using PyMySQL throws:

```
pymysql.err.OperationalError: (1045, "Access denied for user 'admin'@'172.30.0.250' (using password: YES)")
```

**Same host. Same user. Same password. Different outcome.**

In this post, we'll explore why by:

- Walking through the MySQL handshake and authentication plugin negotiation.
- Comparing the official MySQL C client (MySQL CLI) with the pure-Python PyMySQL implementation.
- Using a real failure as a case study.
- Following the stack trace into PyMySQL's `_auth.py`.
- Understanding the root cause and how to debug it.

By the end, you'll have a mental model for "why MySQL works but PyMySQL doesn't" when dealing with authentication, IAM, or proxies.

---

## The Setup

The environment is straightforward: from the same EC2 instance, the MySQL CLI connects successfully through an RDS Proxy to Aurora MySQL, but the Python script fails with ERROR 1045.

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

The server chose `caching_sha2_password` and PyMySQL correctly selected the handler and computed a scrambled password—but the server rejected the credential anyway.

---

## Authentication Plugins: Where Clients Diverge

Modern MySQL uses pluggable authentication. When the server switches plugins (e.g., to `caching_sha2_password`, SHA-256, or IAM), clients must implement the corresponding handler. This is where MySQL CLI and PyMySQL can diverge.

---

## How the MySQL CLI and PyMySQL Handle the Handshake

The MySQL CLI correctly detects the plugin from the server handshake and adapts. PyMySQL is pure Python and implements the wire protocol itself, including plugins like `caching_sha2_password`. This implementation lives in `pymysql/_auth.py`.

When the server requests `caching_sha2_password`, PyMySQL should:

1. Parse the handshake packet and extract the server salt (nonce).
2. Compute a scrambled password using the MySQL protocol algorithm.
3. Send the scramble via the wire.
4. Handle full auth if needed (request public key or rely on TLS).

---

## The PyMySQL v1.1.1 Bug: Salt Null Byte Issue

If you're using **PyMySQL v1.1.1 with RDS Proxy**, you've likely hit a known bug in `caching_sha2_password` authentication.

The server sends a 21-byte salt (20 bytes of data + a trailing null terminator). **PyMySQL v1.1.1 reads all 21 bytes without stripping the null**, then hashes with the wrong salt length (21 instead of 20), producing an invalid hash. The server rejects the invalid response with ERROR 1045.

**What gets sent back:**
```python
# 1) Server → Client: auth plugin switch with salt
MySQL Protocol - authentication switch request
    Packet Length: 44
    Packet Number: 2
    Response Code: EOF Packet (0xfe)
    EOF marker: 254
    Auth Method Name: caching_sha2_password
    Auth Method Data: 50013806443c5442477f28202d19185b2c3d490a00

# 2) Server → Client: fast-auth result / auth-more-data after bad scramble
MySQL Protocol
    Packet Length: 32
    Packet Number: 3
    Auth Method Data: 316ed600

```

The server says: "Your response hash doesn't match."

Strip the null byte before hashing:

```python
conn.salt = pkt.read_all()
if conn.salt.endswith(b"\0"):
    conn.salt = conn.salt[:-1]  # Remove it
scrambled = scramble_caching_sha2(conn.password, conn.salt)
```

**Result:** Server accepts authentication.

---

## Conclusion

When the MySQL CLI connects but PyMySQL v1.1.1 fails with ERROR 1045 using `caching_sha2_password`:

- **The bug:** PyMySQL v1.1.1 reads the salt with its trailing null byte (21 instead of 20), producing an invalid hash.
- **The fix:** Upgrade to PyMySQL v1.1.2+, which strips the null byte before hashing.

The stack trace is the server saying, "Your response hash doesn't match what I expected." For other plugin scenarios or proxies, trace the handshake and verify both clients use the same plugin and response format.


---

## References

- [MySQL Caching SHA-2 Authentication](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)
- [PyMySQL `_auth.py`](https://github.com/PyMySQL/PyMySQL/blob/master/pymysql/_auth.py)
- [PyMySQL v1.1.2 Fix – Salt Null Byte](https://github.com/PyMySQL/PyMySQL/commit/01af30fea0880c3b72e6c7b3b05d66a8c28ced7a#diff-dcb4167b7d6f338b8e2410301c2551d5cf36c7e6024252a3b3cea1cb47b217ea)
- [AWS IAM Authentication with Python](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Python.html)

---

