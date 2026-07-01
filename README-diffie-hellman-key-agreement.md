# Diffie-Hellman Key Agreement

A hands-on walkthrough of the Diffie-Hellman key exchange protocol — manual derivation of a shared secret using modular arithmetic, followed by a real implementation using OpenSSL to generate parameters, exchange public keys, and derive a matching shared secret between two parties.

## What I Learned (and Why It Matters)

Through this Diffie-Hellman Key Agreement project, I learned how two parties can securely establish a shared secret over an insecure network without ever transmitting the secret itself. I practiced the mathematical concepts behind the algorithm using modular arithmetic and then implemented the key exchange using OpenSSL. I also learned how to generate Diffie-Hellman parameters, create private and public keys, exchange public keys, derive a shared secret, and verify that both parties produced the same encryption key. Understanding this process is important because Diffie-Hellman is a foundational cryptographic protocol used in secure communications such as TLS/HTTPS, VPNs, and SSH to protect sensitive data.

## Deployment Instructions

Begin by generating Diffie-Hellman parameters using OpenSSL and save them as a `.pem` file. Next, create a private/public key pair for each participant using the shared parameters. Extract and exchange the public keys between both users, then use each user's private key with the other user's public key to derive the shared secret. Finally, compare the generated secret files to confirm they are identical, proving that both parties successfully established the same encryption key without exposing their private keys.

## What I Enjoyed Most

The part I enjoyed most was seeing how the Diffie-Hellman key exchange worked both mathematically and in practice. Calculating the shared secret by hand made the underlying concept easier to understand, while using OpenSSL demonstrated how the same process is implemented in real-world systems. It was especially satisfying to verify that both users independently generated the exact same shared secret, reinforcing how modern encryption protocols securely establish keys over untrusted networks.
