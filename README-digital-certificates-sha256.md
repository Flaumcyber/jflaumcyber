# Digital Certificates & SHA-256 Flags

Generating and inspecting self-signed X.509 certificates with OpenSSL on Windows, then using SHA-256 hashing to validate file integrity — covering RSA key generation, certificate export/import, and cryptographic hash verification.

## What I Learned (and Why It Matters)

I learned how digital certificates are created using OpenSSL, how public and private keys work together to secure communications, and how SHA-256 hashing verifies file integrity. These skills are important because they form the foundation of secure authentication, encryption, and data protection used in modern cybersecurity.

## Deployment Instructions

1. Install OpenSSL on a Windows system.
2. Generate an RSA private key and self-signed certificate using the provided OpenSSL commands.
3. Export the certificate to a `.pfx` file and protect it with a password.
4. Import the certificate using the Windows Certificate Import Wizard.
5. Validate files by calculating their SHA-256 hash with OpenSSL and compare the result with the known valid hash to verify file integrity.

## What I Enjoyed Most

I enjoyed seeing how cryptography works in a practical setting. Creating my own digital certificate and using SHA-256 hashes to identify the correct flag showed how encryption and integrity checks are applied in real-world cybersecurity environments.
