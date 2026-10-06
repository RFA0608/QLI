---
title: 06. Encrypted closed loop
---
# Re-Encryption
## Intro
In **encrypted control**, it is necessary to properly handle the effects of the **noise** (or error) injected during encryption. This injected noise increases the security of the ciphertext, but it also **grows larger** as homomorphic operations are continuously performed.

Therefore, in encrypted control, it is difficult to sustain an infinite number of operations **without bootstrapping**, unless a special ciphertext method (like [[05_Encryption method#Homomorphic encryption#RGSW|Encryption method>Homomorphic encryption>RGSW]]) is used.

To solve this, a technique called **"re-encryption"** is introduced.
## Comparison
Below is a typical plant and controller pair.
![[Structure_closed_loop.png]]
If we simply add encryption and decryption to this general closed loop, it can be represented as follows:

![[Structure_enc_closed_loop.png]]
As seen in [[03_Controller model|Controller model]], even if the Controller is transformed to be compatible with encryption, it **reuses the encrypted result $u$** in its next computation. Therefore, if represented simply as above, the system **cannot continue to operate properly** due to the cumulative effect of the growing noise injected by the encryption.

This problem can be solved by using **bootstrapping**, which manages the injected noise and allows for continuous homomorphic operations, but this **requires a very large amount of computation time**.

Therefore, we use a small trick called **re-encryption**, where the encrypted $u$ is **decrypted** and then fed back into the controller as a fresh, re-encrypted input.

![[Structure_re_enc.png]]
Through this method, the **accumulation of error** due to computation can be dramatically **avoided**.

Of course, this method also has drawbacks: since $u$ must be re-encrypted, an **additional encryption/decryption cycle** occurs, and an **additional communication line** is also created.

However, this library assumes that $u$ and $y$ are encrypted and decrypted at a single layer, allowing them to be **packed into a single polynomial** for transmission. This results in a **single communication line** and a **single encryption/decryption process**.