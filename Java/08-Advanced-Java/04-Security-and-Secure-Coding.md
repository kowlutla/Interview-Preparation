# Java Security and Secure Coding

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Secure Java code depends on correct API use, input validation, least privilege, safe cryptography, secure deserialization, and controlled resource consumption. The JDK provides security primitives, but application security requires correct protocol and threat-model decisions rather than simply calling a cryptographic method.

## 1. Input and injection risks

### Q1. Why is input validation important? *(Beginner)*

Input validation ensures data conforms to the domain's expected shape, range, and semantics before it reaches sensitive operations. Validate at trust boundaries and enforce business invariants inside domain logic as well.

Validation is not a substitute for output encoding or parameterized APIs. Different contexts—SQL, HTML, shell, paths, and URLs—have different injection risks.

### Q2. How do you prevent SQL injection? *(Intermediate)*

Use PreparedStatement or an ORM's parameterized query support so values remain data rather than SQL syntax. Do not concatenate untrusted input into query strings.

Placeholders generally cannot parameterize identifiers such as table names or column names. Use a strict allowlist for dynamic identifiers and least-privilege database credentials.

### Q3. How do you prevent path traversal? *(Advanced)*

Resolve user-supplied paths against a trusted root, normalize them, and verify they remain within the permitted directory. Account for symbolic links and time-of-check/time-of-use races, not only string prefixes.

Restrict filesystem permissions and define whether symlinks are allowed. For sensitive operations, validate the actual resolved filesystem target as part of a race-aware design.

### Q4. What is SSRF and how can Java code prevent it? *(Advanced)*

Server-Side Request Forgery occurs when an attacker causes a server to make requests to unintended destinations, potentially including internal services or metadata endpoints. A Java HTTP client that accepts arbitrary URLs can be vulnerable if destination policy is missing.

Allowlist schemes/hosts where possible, validate resolved addresses, control redirects, and block private/internal destinations according to the deployment topology. DNS rebinding and alternate address encodings require careful handling.

## 2. Cryptography basics

### Q5. What is the difference between hashing and encryption? *(Beginner)*

Hashing maps input to a fixed-size digest and is designed for one-way integrity/fingerprint use. Encryption is reversible with the appropriate key and is used for confidentiality. Password storage needs a password-hashing/KDF scheme, not ordinary fast hashing.

Choose a standard vetted algorithm and library. Do not invent cryptographic constructions or assume that Base64 is encryption.

### Q6. Why should passwords not be stored with plain SHA-256? *(Advanced)*

SHA-256 is designed to be fast, which helps attackers test password guesses rapidly. Password storage should use a purpose-built password-hashing algorithm such as Argon2id, scrypt, bcrypt, or PBKDF2 with appropriate parameters and salts, according to the application's security policy.

Use a maintained library and tune parameters for the deployment. Do not create a custom hash loop or rely on a secret salt alone.

### Q7. What is a salt? *(Intermediate)*

A salt is a per-password random value included in password hashing. It prevents identical passwords from producing identical stored hashes under a correctly designed scheme and makes precomputed rainbow-table attacks less useful.

A salt is not a secret key. Store it with the hash as required by the password-hashing format and use a unique, cryptographically secure random salt.

### Q8. What is the difference between a secure random generator and Random? *(Advanced)*

java.util.Random is a pseudorandom generator intended for ordinary simulation and non-security uses. SecureRandom is designed for cryptographically strong random values and should be used for tokens, salts, keys, and nonces where unpredictability matters.

Do not use timestamps, UUID variants without understanding their security properties, or Math.random as a substitute for cryptographically secure randomness in security-sensitive contexts.

### Q9. What is the difference between symmetric and asymmetric encryption? *(Intermediate)*

Symmetric cryptography uses the same secret key (or closely related keys) for encryption and decryption. Asymmetric cryptography uses a public/private key pair and supports operations such as key agreement or digital signatures depending on the algorithm.

Real protocols often combine them: asymmetric techniques establish/authenticate key material and symmetric algorithms protect bulk data. Use established protocols such as TLS rather than designing a custom handshake.

### Q10. What is a digital signature? *(Intermediate)*

A digital signature provides integrity and authenticity under a public-key signature scheme: a signer uses a private key and a verifier uses the corresponding public key. It does not by itself encrypt the signed content or guarantee that the signer is trustworthy without key validation.

Use Java's Signature API with appropriate algorithms and key management. Verify what exactly is signed, canonicalize structured data carefully, and validate certificates or trust anchors as required.

## 3. TLS and secrets

### Q11. Why should TLS certificate validation never be disabled in production? *(Advanced)*

Certificate and hostname validation help ensure the client is communicating with the intended server. Disabling validation can allow an attacker to intercept or alter traffic even though the connection appears encrypted.

Fix trust-store, certificate-chain, hostname, or environment configuration issues instead. Encryption without authentication does not provide the intended protection against man-in-the-middle attacks.

### Q12. How should secrets be managed in Java services? *(Senior)*

Use a dedicated secret-management system or protected runtime configuration, limit access, rotate credentials, and avoid embedding secrets in source code, logs, container images, or exception messages. Minimize copies and lifetime in memory where feasible.

Environment variables can be a delivery mechanism but are not a complete secrets-management policy by themselves. Audit access and define rotation and incident-response procedures.

### Q13. What is constant-time comparison and when does it matter? *(Advanced)*

Some secret comparisons should avoid timing differences that reveal information about the compared values. Use APIs designed for the purpose, such as MessageDigest.isEqual for byte arrays in relevant cases, and follow the cryptographic library's guidance.

Constant-time comparison does not make an insecure protocol secure. Consider the entire attack surface, including error messages, retries, network timing, and key handling.

## 4. Deserialization and secure API design

### Q14. Why is Java native deserialization dangerous for untrusted input? *(Advanced)*

ObjectInputStream can reconstruct object graphs and trigger behavior through available classes during deserialization. Gadget chains can turn attacker-controlled data into code execution or other severe effects.

Avoid native serialization for untrusted data. On Java 9 and later, if it is unavoidable, use strict ObjectInputFilter allowlists and resource limits, keep dependencies patched, and test the reachable object graph. Java 8 has no equivalent standard ObjectInputFilter API; prefer not to deserialize untrusted native object streams rather than assuming the Java 9+ filter is available.

### Q15. What is least privilege? *(Beginner)*

Least privilege means giving a component only the permissions and credentials needed for its job. A database account used by a read-only endpoint should not have schema-administration privileges, and a service should not inherit broad cloud permissions without need.

Enforce least privilege in identity, filesystem, network, database, and runtime configuration. It limits the impact of a vulnerability but does not replace input validation or secure coding.

### Q16. How do you handle security exceptions without leaking information? *(Senior)*

Return stable, safe client-facing errors and log detailed context in controlled internal systems. Avoid exposing stack traces, SQL details, filesystem paths, tokens, keys, or internal hostnames to untrusted callers.

Use correlation IDs to connect the response to internal diagnostics. Ensure logging itself does not leak secrets and that error mapping does not reveal sensitive account existence where that matters.

### Q17. How should dependency vulnerabilities be handled? *(Senior)*

Maintain an inventory of dependencies, scan for known vulnerabilities, assess reachability and exposure, upgrade to supported patched versions, and test compatibility. Avoid suppressing findings without a documented risk decision.

Security fixes should be reproducible through dependency management and CI. Monitor transitive dependencies and shaded copies, not only direct declarations.

### Q18. What is a secure-coding review checklist for a Java backend? *(Senior)*

Check authentication and authorization at every boundary, parameterized database access, output encoding, path/URL validation, TLS verification, secret handling, deserialization, dependency hygiene, bounded resource usage, and safe logging. Review retry/idempotency behavior for externally visible operations.

Test negative cases and abuse scenarios. Security is a system property involving code, configuration, identity, network, deployment, and operational monitoring.

## Official references

1. [Java SE 21 Security Developer's Guide](https://docs.oracle.com/en/java/javase/21/security/)
2. [Java SE 21 API — SecureRandom](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/security/SecureRandom.html)
3. [Java SE 21 API — Signature](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/security/Signature.html)
4. [Java SE 21 API — MessageDigest](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/security/MessageDigest.html)
5. [JEP 290 — Filter Incoming Serialization Data](https://openjdk.org/jeps/290)

## Chapter checklist

- [ ] Separate hashing, encryption, signatures, and password hashing.
- [ ] Use SecureRandom for security-sensitive randomness.
- [ ] Preserve TLS validation and least privilege.
- [ ] Prevent injection, path traversal, and SSRF.
- [ ] Avoid untrusted native deserialization and secret leakage.
